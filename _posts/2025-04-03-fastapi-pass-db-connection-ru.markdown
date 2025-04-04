---
layout: post
title:  "Передаем подключение к базе данных в Fastapi"
date:   2025-04-03
categories: ru
tags: "python fastapi contextvars"
---

## Проблематика

Разработчики Fastapi [предлагают](https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/#a-database-dependency-with-yield) 
вполне конкретный способ передать подключение к базе данных через Depends. Выглядит это как-то так.
```python
async def db_dependency() -> AsyncGenerator[asyncpg.Connection, None]:
    connection = await db_connect()
    try:
        yield connection
    finally:
        await connection.close()
```
И далее где-то в обработчике запроса
```python
@app.get("/foo")
async def foo(db: Annotated[Connection, Depends(db_dependency)]):
    # do some job further using db connection
```
Таким образом соединение с базой данных попадает в пользовательский код. 
И если бы весь код всегда помещался в одном обработчике, то никаких проблем с этим не было бы. 
Однако чаще мы имеем следующую ситуацию.
```python
@app.get("/foo")
async def foo(db: Annotated[Connection, Depends(db_dependency)]):
    return await bar(db)

async def bar(db: Connection):
    result = await baz(db)
    return json.dumps(result)

async def baz(db: Connection):
    return await db.fetch("SELECT * FROM baz")
```
Будем честны, без базы данных мало какие запросы можно обработать. Поэтому мы везде будем 
"тащить" зависимость `db: Connection` через аргумент функций. И когда у нас в кодовой базе сотни таких функций, 
у каждой из которых есть еще и свои "полезные" зависимости, то становится совсем нерадостно от перспективы 
всегда иметь занятым +1 аргумент просто так.

## Решение

Интуитивно понятно, что соединение с базой - это что-то очень важное, что-то, что должно всегда быть под рукой.
Кроме того, оно должно быть уникальным, но при этом локальным для каждого запроса, 
чтоб иметь возможность выполнять код с использованием транзакций. Таким образом соединение с базой должно обладать
следующими свойствами:
* Доступность
* Локальность

Во многих языках программирования подобная проблема решается с помощью IoC контейнера и Dependency Injection (DI). 
Depends в FastAPI с натяжкой можно считать полноценным DI. Об этом чуть подробнее будет ниже. 
Но к счастью в python есть другой механизм, который может помочь. 
Это [contextvars](https://docs.python.org/3/library/contextvars.html).

Если говорить очень грубо, то contextvars --- это глобальные переменные на стероидах. Глобальными они являются, 
поскольку доступны из любого участка кода. Однако отличие от обычных глобальных переменных в том, что
их значения зависят от того, в какой цепочке вызова асинхронных функций они используются. Таким образом, contextvars
обладают одновременно _доступностью_ и _локальностью_. 

Примеры выше могут быть переписаны следующим образом с использованием contextvars
```python
db_connection: ContextVar[asyncpg.Connection | None] = ContextVar(
    "db_connection", default=None
)

async def foo(db: Annotated[Connection, Depends(db_dependency)]):
    # put db connection in contextvar in the begining of the handler 
    db_connection.set(db)
    return await bar(db)

async def bar():
    # no db dependency anymore
    result = await baz()
    return json.dumps(result)

async def baz():
    # get db connecttion here
    db = db_connection.get()
    if db is None:
        raise RuntimeError("No database connection")
    return await db.fetch("SELECT * FROM baz")
```
Решение сводится к следующим шагам
1. Создаем переменную contextvar для хранения соединения
2. Кладем соединение с базой в эту переменную в начале каждого обработчика
3. Там, где требуется соединение, достаем его из переменной
4. Profit!!!

## Улучшения
Подход выше будет работать, но он все еще не лишен boilerplate кода. Например, скорее всего нам бы не хотелось видеть 
подобную проверку везде
```python
db = db_connection.get()
if db is None:
    raise RuntimeError("No database connection")
```
Но, если убрать проверку, то mypy будет ругаться. Поэтому можно вынести ее в отдельную функцию
```python
def get_db() -> asyncpg.Connection:
    db = db_connection.get()
    if db is None:
        raise RuntimeError("No database connection")
    return db
    
async def baz():
    db = get_db()
    return await db.fetch("SELECT * FROM baz")
```

Другой момент, это вызов `db_connection.set(db)` в начале каждого обработчика. Логично было бы вынести его
в `db_dependency()`, чтоб при использовании подключения к базе оно всегда попадало в contextvar, но тогда в самом 
обработчике останется ненужная зависимость `db: Annotated[Connection, Depends(db_dependency)]`. Выйти из положения
помогут [Dependencies in path operation decorators](https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-in-path-operation-decorators/),
которые как раз используются, когда нужно исполнить какой-то код в зависимости, но не нужен результат выполнения. Тогда
`db_dependency()` можно переписать так
```python
async def db_dependency() -> AsyncGenerator[None, None]:
    connection = await db_connect()
    db_connection.set(connection)
    try:
        # now yield empty result
        yield
    finally:
        await connection.close()
```
А обработчик запроса будет выглядеть так 
```python
@app.get("/foo", dependencies=[Depends(db_dependency)])
async def foo(): # no arguments here
    return await bar(db)
```

Таким образом, чтоб подключение стало доступно в коде через contextvar, нужно добавить
`dependencies=[Depends(db_dependency)]` в декоратор. Можно подумать, что мы строчку 
`db_connection.set(db)` просто заменили на `dependencies=[Depends(db_dependency)]`, но на самом деле мы заменили две
строчки, ведь до этого было еще `db: Annotated[Connection, Depends(db_dependency)]`. Ну и субъективно, мне так больше 
нравится:) Мы будто наделяем всю цепочку асинхронных вызовов определенным свойством через параметр в деораторе. Это 
свойство --- доступность подключения к базе данных.


## Минусы

Несмотря на то, что описанный подход справляется с проблемой, у него есть ряд недостатков.

Небольшим практическим недостатком является необходимость явного указания `dependencies=[Depends(db_dependency)]` 
в декораторе каждого обработчика.
К сожалению, я не нашел элегантного способа заставить FastAPI выполнить данную операцию для всех обработчиков 
по умолчанию. Ни [global dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/global-dependencies/), ни
middlewares не помогают. В случае global dependencies вызов кода зависимости происходит в отдельном контексте, поэтому
значение переменной сбрасывается до исходного `None` в обработчиках. А middlewares в FastAPI вообще 
[не принимают](https://github.com/fastapi/fastapi/discussions/8193) на вход Depends. На мой взгляд, это 
~~косяк~~ недостаток гибкости FastAPI. И это один из аргументов, почему Depends - это обрубок от полноценного DI.

Далее, не весь код может исполняться в контексте обработки запроса. За примерами далеко ходить не надо. Достаточно 
вспомнить про тестирование или про [background tasks](https://fastapi.tiangolo.com/tutorial/background-tasks/). Это
означает, что при переиспользовании кода вне обработки запросов надо позаботиться о корректном подключении к базе. Но
это ожидаемо, и это пол-беды. Менее заметная проблема в том, что после завершения задачи нужно закрыть 
соединение. И это главная причина, почему **не рекомендуется использовать contextvars для хранения подключения
к базе данных в общем случае**. Именно наличие 
[dependencies with yield](https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/) 
в FastAPI делает возможным использование contextvars для этих целей. FastAPI гарантирует 
вызов `await connection.close()` после отправки ответа. Но описанный подход нельзя обобщить на любой асинхронный код.

Ну и конечно не забываем про появившуюся неявную зависимость в коде. 

## Выводы

Описанное решение подходит не только для передачи соединения с базой. Его легко адаптировать для любого другого 
"глобального" объекта. Например, я также использую его для передачи соединения с aws bedrock в ендпоинте, у которого
пайплайн обработки подразумевает множественные запросы в этот сервис. Это позволяет сэкономить драгоценные секунды на
реконнектах. Однако необходимо помнить, что таким образом можно передавать только те объекты, использование которых 
ограничено рамками обработки конкретного запроса. 

В заключении кратко подсвечу ключевые моменты 
* contextvars можно использовать в FastAPI для предоставления доступа к базе данных и другим объектам во всей кодовой базе
* такой подход работает во многом благодаря dependencies with yield в FastAPI
* вне обработки запросов надо быть осторожным с открытием и закрытием соединения
* н~~е является инвестиционной рекомендацией~~




