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
совмещают с паттерном Unit of Work. Он позволяет выделить скоуп объектов в рамках бизнес-транзакции, который можно
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
Ключевой момент здесь в том, что запросы в бд выполняются только после вызова `$entityManager->flush()`. Все остальное
время мы оперируем с сущностями
Здесь мы имеем дело с Bi-directional ManyToOne связыванием. Для Doctrine важна только, та
сторона, которая хранит внешний ключ. Она еще называется Owning side. У нас это Twit, т.к. именно здесь -- ссылка на юзера. 
Подробнее про Owning и Inverse в официальной документации [тут](https://www.doctrine-project.org/projects/doctrine-orm/en/3.5/reference/unitofwork-associations.html).

Doctrine отслеживает изменение только в Owning side. Это означает что, если мы захотим поменять user у твита, то 
нам достаточно просто присвоить нового пользователя.



