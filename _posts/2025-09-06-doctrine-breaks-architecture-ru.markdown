---
layout: post
title:  "Почему Doctrine ORM ломает вашу архитектуру"
date:   2025-09-06
categories: ru
tags: "php doctrineorm"
---

## Введение
Все ORM можно условно разделить на две категории: Active Records и Data Mappers. 

При использовании Active Record строке в базе данных ставится в соответствии некий объект. С ним можно выполнять CRUD 
операции, которые отобразятся на самой строке. Про (анти)паттерн Active Record 
написано много. Ему вменяют такие проблемы, как
* смешивание слоев абстракции, поскольку в объект часто помещают как персистентный слой, так и бизнес логику
* слабое представление доменной модели, поскольку active record -- это отображение строки в таблице, а не доменного объекта
* сложности с юнит-тестированием

В основе подхода с Data Mappers лежит идея явного разделения слоя бизнес логики и персистентного слоя. Предполагается,
что вы работаете с сущностями, не задумываясь о том, когда и где они будут синхронизированы с базой. Такой подход часто
совмещают с паттерном Unit of Work (UoW). Он позволяет выделить скоуп объектов в рамках бизнес-транзакции, который можно
синхронизировать с базой в конце этой транзакции.

Так это должно работать. По крайней мере, по задумке. По этой же задумке Data Mappers справляются с 
минусами Active Record. 

Однако идея "_давайте притворимся, что никакого персистентного 
слоя нет, а потом там где-нибудь все сохраним_" на практике приводит к протеканию абстракций, либо к деградации 
производительности. Именно об этом хотелось бы сегодня поговорить подробнее на примере Doctrine ORM.

## Мягкое погружение в Doctrine ORM.
Допустим мы делаем клон twitter. И для этого нам понадобятся как минимум две сущности User и Twit. Вот как могли бы
выглядеть они в Doctrine ORM.
```php
#[ORM\Entity]
class User
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    private(set) int $id;

    #[ORM\OneToMany(targetEntity: Twit::class, mappedBy: 'user')]
    private(set) Collection $twits;

    public function __construct()
    {
        $this->twits = new ArrayCollection();
    }
}

#[ORM\Entity]
class Twit
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    private(set) int $id;

    #[ORM\Column(type: 'string', length: 255)]
    public string $text;

    #[ORM\ManyToOne(targetEntity: User::class, inversedBy: 'twits')]
    public User $user;

    public function __construct(User $user, string $text)
    {
        $this->user = $user;
        $this->text = $text;
    }
}
```
Теперь для примера мы можем создать пользователя и сразу добавить ему парочку твитов.
```php
// Тут создаем сущности.
$user = new User();
$twit1 = new Twit($user, 'first twit');
$twit2 = new Twit($user, 'second twit');
// Тут говорим доктрине, что надо бы за ними наблюдать. Никаких запросов в бд здесь.
$entityManager->persist($user);
$entityManager->persist($twit1);
$entityManager->persist($twit2);
// А это тот самый момент, синхронизации объектов с бд.
$entityManager->flush();
```
Вызовы persist() помещают указанный объект в UoW. А запросы в бд выполняются только после вызова `$entityManager->flush()`. 

Более наглядно UoW проявляется при обновлении сущностей.
```php
$twitRepo = $entityManager->getRepository(Twit::class);
// Находим какой-то твит и меняем текст.
$twit = $twitRepo->find($id);
$twit->text = 'changed text';
// Сходим в бд за всеми твитами.
// Все, что мы загрузили через repository автоматически попадает в UoW.
$twits = $twitRepo->findAll();
foreach ($twits as $tw) {
    if ($tw->id === $id) {
        // Доктрина "помнит" что мы поменяли текст, хотя в бд ничего не сохранено.
        assert($tw->text === 'changed text');
        // Более того, это один и тот же объект.
        assert($tw === $twit);
    }
}
// А вот теперь сохраняем в бд.
$entityManager->flush();
```
Таким образом, работа с Doctrine сводится к следующим принципам
* объекты помещаются в UoW либо через вызов `$entityManager->persist()`, либо автоматически, при загрузке из бд 
* далее в соответствии с бизнес логикой с ними производятся операции
* после вызова `$entityManager->flush()` doctrine пробегает по всем сущностям в UoW, и формирует список изменений
* из списка изменений генерируются sql запросы, происходит синхронизация с бд

## Doctrine ORM in its prime
Рассмотрим следующую бизнес задачу. У нас было несколько способов регистрации пользователей, и один и тот же человек 
мог создать множество аккаунтов разными способами. И теперь пользователи жалуются, что они не могут переместить все
свои твиты в какой-то один аккаунт. Поэтому нам надо предоставить возможность слияния двух аккаунтов.

Что значит "слияние двух аккаунтов"? Это означает, что нам нужно
* переместить все твиты из одного аккаунта в другой (возможно перенести еще какую-то информацию, но в нашем примере больше ничего нет)
* удалить первый аккаунт

Давайте разбираться последовательно с каждым из этих пунктов. Начнем с перемещением. Создадим двух пользователей, и один
твит.
```php
$user1 = new User();
$user2 = new User();
$twit = new Twit($user1,'some text');

$entityManager->persist($user1);
$entityManager->persist($user2);
$entityManager->persist($twit);
$entityManager->flush();
```
Тогда кажется, чтоб поменять владельца твита, достаточно вызвать
```php
$twit->user = $user2;
$entityManager->flush();
```
И действительно, это сработает. Теперь твит принадлежит другому пользователю. Но есть одна проблема.
```php
assert($user1->twits->contains($twit));
$twit->user = $user2;
assert($user1->twits->contains($twit));
$entityManager->flush();
assert($user1->twits->contains($twit));
```
Обратите внимание, что все asserts показывают один и тот же результат: твит все еще принадлежит `$user1`, что неверно, и
может привести к ошибкам в коде. Последние два assert должны падать.

Здесь мы имеем дело с ManyToOne связыванием. Для Doctrine важна только, та
сторона, которая хранит внешний ключ. Она еще называется Owning side. У нас это Twit, т.к. именно здесь -- ссылка на юзера. 

Другая сторона, inverse side, находится внутри User. Это поле twits. Она используется лишь для удобства получения данных, 
и является опциональной.
Doctrine не отслеживает внешние изменения здесь. Также doctrine не обновляет inverse сторону при изменении owning, 
поэтому это нужно делать нам самим. Для этого придется немного изменить класс Twit.
```php
class Twit
{
    // остальной код тот же
    #[ORM\ManyToOne(targetEntity: User::class, inversedBy: 'twits')]
    public User $user {
        set(User $user) {
            if (!isset($this->user)) {
                // Первое присваивание.
                $this->user = $user;
                return;
            }
            // Удаляем этот твит у прошлого юзера.
            $this->user->twits->removeElement($this);
            $this->user = $user;
            if (!$user->twits->contains($this)) {
                // Добавляем тви текущему юзеру.
                $user->twits->add($this);
            }
        }
    }
}
```

Вот теперь у нас корректно обновляются данные у обоих юзеров при изменении владельца твита.

Подробнее про Owning и Inverse в официальной документации [тут](https://www.doctrine-project.org/projects/doctrine-orm/en/3.5/reference/unitofwork-associations.html).

Далее по плану, надо разобраться с удалением. Тут все просто: для того, чтобы удалить пользователя, необходимо сначала
удалить всего его твиты, иначе получим ошибку. Сделать это можно несколькими способами, например добавить 
`cascade: ['remove']` в аннотации
```php
#[ORM\OneToMany(targetEntity: Twit::class, mappedBy: 'user', cascade: ['remove'])]
private(set) Collection $twits;
```

Но я предлагаю сделать это более явно с помощью [lifecycle callbacks](https://www.doctrine-project.org/projects/doctrine-orm/en/3.5/reference/events.html#lifecycle-callbacks)
```php
#[ORM\Entity]
#[ORM\HasLifecycleCallbacks]
class User
{
    // остальной код тот же
    #[ORM\PreRemove]
    public function preRemove(PreRemoveEventArgs $args): void
    {
        foreach ($this->twits as $twit) {
            $args->getObjectManager()->remove($twit);
        }

    }
}
```

Опять же, здесь не выполняются никакие запросы в бд. Doctrine лишь помечает все твиты, чтоб выполнить удаление при 
следующем вызове `flush()`. Теперь мы можем корректно удалять любого юзера со всеми его твитами, не опасаясь foreign 
key constraint проблем.
```php
$entityManager->remove($user);
$entityManager->flush();
```

Теперь мы готовы написать класс, который будет заниматься слиянием двух аккаунтов.
```php
final readonly class UserMergeManager
{
    public function __construct(
        private EntityManagerInterface $entityManager,
    ) {
    }

    public function merge(User $source, User $target): void
    {
        foreach ($source->twits as $twit) {
            $twit->user = $target;
        }
        $this->entityManager->remove($source);
        $this->entityManager->flush();
    }
}
```
Будет ли это все работать? Конечно. Все ли здесь понятно? Безусловно. 

Таким образом мы видим, как Doctrine ORM позволяет решить бизнес задачу, и в результате у нас написан чистый и 
понятный код.

## Черный день для Doctrine ORM
`UserMergeManager` выглядит корректным в общем случае, и кажется, будто он должен работать всегда. К 
сожалению это не так. Я не случайно сказал, что inverse side опциональная. Она не нужна doctrine для синхронизации 
объектов с бд и в некоторых случаях даже может быть вредной.

Представьте себе базу данных, которая хранит пользователей и города. Мы хотим обозначить, в каком городе живет 
пользователь. Это все та же связь ManyToOne, только Owning сторона теперь пользователь. У города же могло бы быть поле
`users`, в котором был бы список пользователей, проживающих в этом городе. Но в городе живут сотни тысяч людей. 
Случайное обращение к такому полю может вызвать запрос в базу и гидрацию сотни тысяч объектов. Поэтому inverse сторону
добавлять не стоит.

Теперь давайте посмотрим, а что будет если в нашем твиттере убрать эту связь, т.е. убрать в User поле twits



