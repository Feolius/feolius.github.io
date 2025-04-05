---
layout: post
title:  "Как правильно подключиться к базе данных в Fastapi"
date:   2025-04-03
categories: ru
tags: "python fastapi contextvars"
---

## Проблематика
В FastAPI есть прикол: подключение к базе данных порождает +1 аргумент во всех функциях.

В чем суть: FastAPI [предлагают](https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/#a-database-dependency-with-yield) способ передавать подключение к базе данных через Depends:
```python
async def db_dependency() -> AsyncGenerator[asyncpg.Connection, None]:
    connection = await db_connect()
    try:
        yield connection
    finally:
        await connection.close()
```
И далее где-то в обработчике запроса:
```python
@app.get("/foo")
async def foo(db: Annotated[Connection, Depends(db_dependency)]):
    # do some job further using db connection
```
Если бы весь код всегда помещался в одном обработчике, то проблем бы не было. Но:
```python
@app.get("/foo")
async def foo(db: Annotated[Connection, Depends(db_dependency)]):
    return await bar(db)

async def bar(db: Connection):
    # no actual db usage here, just passing it further
    result = await baz(db)
    return json.dumps(result)

async def baz(db: Connection):
    return await db.fetch("SELECT * FROM baz")
```
Будем честны, без базы данных мало какие запросы можно обработать, поэтому везде тащим зависимость `db: Connection` 
через аргумент функций. 

По итогу имеем нерадостную перспективу – сотни функций с "полезными" зависимостями и всегда +1 занятый аргумент 💩

## Решение

Интуитивно понятно, что соединение с базой – дело важное. Оно должно быть:
* уникальным для каждого запроса, чтобы выполнять код через транзакции
* доступным в любом месте кода


Во многих языках программирования помогает IoC контейнер и Dependency Injection (DI). Depends в FastAPI с натяжкой можно
считать полноценным DI, но в Python есть [contextvars](https://docs.python.org/3/library/contextvars.html) – глобальные переменные на стероидах. 
Они доступны из любого участка кода, однако в отличие от обычных глобальных переменных, их значения зависят от того, в какой цепочке вызова асинхронных функций они используются. 
То есть, contextvars обладают одновременно _доступностью_ и _уникальностью_ 🤩

1. Создаем переменную contextvar для хранения соединения
2. Кладем соединение с базой в эту переменную в начале каждого обработчика
3. Там, где требуется соединение, достаем его из переменной
4. Profit

Перепишем наш код на contextvars:
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


## Классно!.. Почти
Тема получается рабочая, но все еще немного boilerplate. Например, хотелось бы убрать проверку:
```python
db = db_connection.get()
if db is None:
    raise RuntimeError("No database connection")
```
Но mypy будет ругаться. Так что выносим в отдельную функцию:
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
Уже лучше!

Еще один boilerplate-персонаж это вызов `db_connection.set(db)` в начале каждого обработчика. Логично было бы вынести его
в `db_dependency()`, чтоб при использовании подключения к базе он всегда попадал в contextvar. Но в 
обработчике останется ненужная зависимость `db: Annotated[Connection, Depends(db_dependency)]`💩 

Выйти из положения помогут [Dependencies in path operation decorators](https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-in-path-operation-decorators/). Они применяются, когда в зависимости нужно 
исполнить код, но не нужен результат. 

Переписываем `db_dependency()`:
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
Добавляем `dependencies=[Depends(db_dependency)]` в декоратор:
```python
@app.get("/foo", dependencies=[Depends(db_dependency)])
async def foo(): # no arguments here
    return await bar()
```

 
Кажется, что мы всего лишь заменили одну строчку кода (`db_connection.set(db)`на `dependencies=[Depends(db_dependency)]`). 
Все не так просто! Мы заменили целых две строчки: `db_connection.set(db)` и `db: Annotated[Connection, Depends(db_dependency)]`. 

Через параметр в декораторе мы будто наделяем всю цепочку асинхронных вызовов волшебным свойством. Это 
свойство --- доступность подключения к базе данных.


## Минусы подхода
**1. Необходимость явного указания `dependencies=[Depends(db_dependency)]`в декораторе каждого обработчика**

К сожалению, я не нашел элегантного способа заставить FastAPI выполнять операцию для всех обработчиков по умолчанию:
- С [global dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/global-dependencies/) вызов кода зависимости происходит в отдельном контексте, поэтому значение переменной сбрасывается до исходного `None` в обработчиках. 
- Middlewares в FastAPI вообще [не принимают](https://github.com/fastapi/fastapi/discussions/8193) на вход Depends. На мой взгляд, это ~~косяк~~ негибкость FastAPI. В том числе поэтому Depends - это обрубок от полноценного DI.

**2. Не весь код может исполняться в контексте обработки запроса** 

За примерами далеко ходить не надо: взять хотя бы тестирование или [background tasks](https://fastapi.tiangolo.com/tutorial/background-tasks/). 

При переиспользовании кода вне обработки запросов придется позаботиться о корректном подключении к базе, но это пол беды. После завершения задачи нужно закрыть 
соединение. И это главная причина, почему **не рекомендуется использовать contextvars для хранения подключения
к базе данных в общем случае**. Благодаря 
[dependencies with yield](https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/) 
мы можем использовать для хранения подключения contextvars. FastAPI гарантирует 
вызов `await connection.close()` после отправки ответа, но подход нельзя обобщить на любой асинхронный код.

**3. Неявная зависимость в коде**

## Выводы
Благодаря описанному решению мы избавляемся от вездесущего аргумента `db` в FastAPI, а еще оно подходит не только для передачи 
соединения с базой. Его легко адаптировать для любого 
"глобального" объекта. Например, я использую его для передачи соединения с aws bedrock в endpoint'е, у которого
пайплайн обработки подразумевает множественные запросы. Так я экономлю драгоценные секунды на
реконнектах. Помним, что так можно передавать только объекты в рамках обработки конкретного запроса. 

Итого:
* contextvars можно использовать в FastAPI для предоставления доступа к базе данных и другим объектам во всей кодовой базе
* такой подход работает во многом благодаря dependencies with yield в FastAPI
* вне обработки запросов надо быть осторожным с открытием и закрытием соединения
* н~~е является инвестиционной рекомендацией~~




