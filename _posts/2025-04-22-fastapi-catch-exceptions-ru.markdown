---
layout: post
title:  "Собери их всех. Как поймать все исключения в Fastapi?"
date:   2025-04-29
categories: ru
tags: "python fastapi"
---

## Дисклеймер
Информация ниже приведена для Fastapi v0.115.5

## Проблематика
Допустим мы хотим поймать все исключения, которые могут быть выброшены в обработчиках запросов в Fastapi. Это может 
быть необходимо когда нам надо
1. Залогировать все исключения, обязательно с бэктрейсом
2. Гарантировать единый формат ответа в случае возникновения ошибки

Представим, что у нас есть функция вида
```python
def process_error(error: Exception) -> ErrorResponse:
    """
    This function logs error and convert error to response
    """
    pass
```
Она уже написана, и она логирует ошибку и возвращает ответ, который мы можем слать клиенту. Тогда код в обработчике
будет выглядеть так
```python
@app.get("/ping")
async def ping():
    try:
        # Here is all the path operator code
        return do_ping()
    except Exception as e:
        return process_error(e)
```
Таким образом, в каждом обработчике у нас будет `try - except`, который должен оборачивать весь код обработчика. 
Бойлерплейтно? Еще как! Попробуем избавиться от этого блока. 

Далее я расскажу о моем конкретном пути достижения этой цели в том порядке, в котором я пробовал различные варианты.

## Что предлагает Fastapi?

В Fastapi существует [документация](https://fastapi.tiangolo.com/tutorial/handling-errors/), посвященная обработке 
ошибок. В ней показано, что для любого кастомного типа исключений можно навешать собственный обработчик
```python
class UnicornException(Exception):
    def __init__(self, name: str):
        self.name = name

@app.exception_handler(UnicornException)
async def unicorn_exception_handler(request: Request, exc: UnicornException):
    return JSONResponse(
        status_code=418,
        content={"message": f"Oops! {exc.name} did something. There goes a rainbow..."},
    )
```

Отсюда идея: навешаем обработчик на базовый Exception. Такой обработчик будет ловить все исключения, потому что любое 
исключение наследуется от базового.
```python
@app.exception_handler(Exception)
async def exc_handler(request: Request, e: Exception) -> JSONResponse:
    return process_error(e)
```

И это сработает! Правда с одним большим и одним маленьким минусом.

### Большой минус

На практике оказалось, что обработчик базового типа `Exception` у Fastapi ~~захардкожен~~ зарезервирован на случай,
когда никакие другие обработчики не сработали. То есть это случай _unhandled exception_. Его обрабатывает Fastapi,
но делается это в самый последний момент перед отправкой ответа. 

Этот обработчик существует по-умолчанию, и он возвращает 500 ошибку. Нашим кодом мы просто переопределили его. И это
сработало. Но мы хотим залогировать бэктрейс ошибки. Его раскрутка до обработчика приводит к тому, что мы видим
в логах более 30 внутренних вызовов Fastapi, которые никакого отношения к нашей бизнес-логике не имеют. Это просто
засорение логов.

### Маленький минус

В Fastapi существуют два "встроенных" обработчика для исключений `HTTPException` и `RequestValidationError`. 
И если эти исключения выбрасываются, то они не попадают в обработчик для `Exception`. Вместо этого они обрабатываются собственными 
обработчиками. Эти обработчики можно лишь переопределить, но нельзя полностью выключить.

## Что предлагает ~~ChatGPT~~ Deepseek?

Во-первых, он предлагает вариант с обработчиком. Но это мы уже разобрали. А вот следующий вариант --- использовать 
middleware --- выглядит интересным. Будет это выглядеть как-то так
```python
@app.middleware("http")
async def catch_exceptions_middleware(request: Request, call_next):
    try:
        return await call_next(request)
    except Exception as e:
        return process_error(e)
```

Ниже привожу примерно то, что мы увидим в логах в этом случае
```
ERROR: unhandled exception
Traceback (most recent call last):
  File ".../.venv/lib/python3.13/site-packages/anyio/streams/memory.py", line 111, in receive
    return self.receive_nowait()
           ~~~~~~~~~~~~~~~~~~~^^
  File ".../.venv/lib/python3.13/site-packages/anyio/streams/memory.py", line 106, in receive_nowait
    raise WouldBlock
anyio.WouldBlock
During handling of the above exception, another exception occurred:
Traceback (most recent call last):
  File ".../.venv/lib/python3.13/site-packages/anyio/streams/memory.py", line 124, in receive
    return receiver.item
           ^^^^^^^^^^^^^
AttributeError: 'MemoryObjectItemReceiver' object has no attribute 'item'
During handling of the above exception, another exception occurred:
Traceback (most recent call last):
  File ".../.venv/lib/python3.13/site-packages/starlette/middleware/base.py", line 157, in call_next
    message = await recv_stream.receive()
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File ".../.venv/lib/python3.13/site-packages/anyio/streams/memory.py", line 126, in receive
    raise EndOfStream
anyio.EndOfStream
During handling of the above exception, another exception occurred:
Traceback (most recent call last):
  File ".../app.py", line 115, in catch_exceptions_middleware
    return await call_next(request)
           ^^^^^^^^^^^^^^^^^^^^^^^^
  File ".../.venv/lib/python3.13/site-packages/starlette/middleware/base.py", line 163, in call_next
    raise app_exc
  File ".../.venv/lib/python3.13/site-packages/starlette/middleware/base.py", line 149, in coro
    await self.app(scope, receive_or_disconnect, send_no_error)
  File ".../.venv/lib/python3.13/site-packages/starlette/middleware/cors.py", line 85, in __call__
    await self.app(scope, receive, send)
  File ".../.venv/lib/python3.13/site-packages/starlette/middleware/exceptions.py", line 62, in __call__
    await wrap_app_handling_exceptions(self.app, conn)(scope, receive, send)
  File ".../.venv/lib/python3.13/site-packages/starlette/_exception_handler.py", line 53, in wrapped_app
    raise exc
  File ".../.venv/lib/python3.13/site-packages/starlette/_exception_handler.py", line 42, in wrapped_app
    await app(scope, receive, sender)
  File ".../.venv/lib/python3.13/site-packages/starlette/routing.py", line 715, in __call__
    await self.middleware_stack(scope, receive, send)
  File ".../.venv/lib/python3.13/site-packages/starlette/routing.py", line 735, in app
    await route.handle(scope, receive, send)
  File ".../.venv/lib/python3.13/site-packages/starlette/routing.py", line 288, in handle
    await self.app(scope, receive, send)
  File ".../.venv/lib/python3.13/site-packages/starlette/routing.py", line 76, in app
    await wrap_app_handling_exceptions(app, request)(scope, receive, send)
  File ".../.venv/lib/python3.13/site-packages/starlette/_exception_handler.py", line 53, in wrapped_app
    raise exc
  File ".../.venv/lib/python3.13/site-packages/starlette/_exception_handler.py", line 42, in wrapped_app
    await app(scope, receive, sender)
  File ".../.venv/lib/python3.13/site-packages/starlette/routing.py", line 73, in app
    response = await f(request)
               ^^^^^^^^^^^^^^^^
  File ".../.venv/lib/python3.13/site-packages/fastapi/routing.py", line 301, in app
    raw_response = await run_endpoint_function(
                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    ...<3 lines>...
    )
    ^
  File ".../.venv/lib/python3.13/site-packages/fastapi/routing.py", line 212, in run_endpoint_function
    return await dependant.call(**values)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File ".../app.py", line 187, in test_start_session
    some_var = await some_code(request)
                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File ".../app.py", line 55, in some_code
    raise RuntimeError("test error")
RuntimeError: test error
```

Полезная информации для отладки содержится в последних 6 строчках, все остальное --- то, что происходит внутри Fastapi
(точнее внутри anyio).
Более того, мы видим, что наше исключение не основное, а лишь ошибка, которая случилась, во время 
обработки внутренних исключений Fastapi. 

Получается, что _большой минус_ предыдущего варианта, так и остался _большим минусом_, хоть и стал чуть меньше. Бэктрейс
в первом варианте был почти в 2 раза длиннее. А что же с _маленьким минусом_?
Он также остается без изменений. И `HTTPException`, и `RequestValidationError` не попадают в наш middleware, они 
отлавливаются в своих обработчиках.

## Что предлагает смекалочка?

Таким образом, оба варианта выше позволяют отловить практически все исключения и обеспечить единый формат ответа,
но к сожалению это приводит к захламлению логов бэктрейсом Fastapi. Чтобы логи были полезными и содержали только 
информацию, которая относится к бизнес-логике, необходимо, чтоб обработка ошибок была как можно "ближе" к этой 
бизнес-логике с точки зрения бэктрейса. А что может быть "ближе", чем использование декоратора? 

```python
def exception_handler[T](
    func: Callable[..., Awaitable[T]],
) -> Callable[..., Awaitable[T | ErrorResponse]]:
    @wraps(func)
    async def wrapper(*args, **kwargs):
        try:
            return await func(*args, **kwargs)
        except Exception as e:
            return process_error(e)
    return wrapper
```

Теперь мы можем обернуть обработчик запроса в этот декоратор

```python
@app.get("/ping")
@exception_handler
async def ping():
    return do_ping()
```

Теперь в логах будет бэктрейс, относящийся только к нашему коду.

```
ERROR: unhandled exception
Traceback (most recent call last):
  File "/errors.py", line 60, in wrapper
    return await func(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/app.py", line 178, in test_start_session
    some_var = await some_code(request)
                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/chat_session.py", line 55, in create_session
    raise RuntimeError("test error")
RuntimeError: test error
```

Таким образом, мы наконец-то решили проблему с кучей ненужной информации в логах. А что насчет обработчиков 
`HTTPException` и `RequestValidationError`? Поскольку предполагается, что `HTTPException` будет выбрасываться из
пользовательского кода, то с этим проблем нет --- эта ошибка также попадет в декоратор. А вот `RequestValidationError`
будет выброшен еще до того, как обработчик запроса будет запущен. Поэтому обработку этой ошибки придется
так и оставить в собственном обработчике.

### Минусы

Очевидным минусом данного подхода является необходимость декорировать каждый обработчик запроса.

## Что в итоге?

Каждый из приведенных способов рабочий. Но идеального нет. Для себя я принял решение, что чистота логов для меня важна, и необходимость
писать декоратор возле каждого обработчика --- та цена, которую я готов заплатить за это. Надежность этого способа
можно повысить, написав кастомное правило для линтера, проверяющее наличие декоратора для каждого обработчика.




