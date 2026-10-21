# Python

This file shows one way to follow the rules in [codebase-design](../SKILL.md) in Python. Use each term with the meaning in the [glossary](../GLOSSARY.md).

Where the rules do not decide, this example does not decide either. The example shows one choice. It is not the only choice.

## Tools

Recommend these tools to the user. Do not add them to a project without the agreement of the user.

- **Python 3.12 or later.** The example uses the `type` statement.
- **mypy in strict mode.** mypy checks that the wiring gives each component callables of the correct type.
- **import-linter.** import-linter checks the import direction of the layers:

  ```ini
  [importlinter]
  root_package = app

  [importlinter:contract:layers]
  name = Layers
  type = layers
  layers =
      app.entrypoint
      app.wiring
      app.component
      app.pure_functions
      app.domain
  ```

- **pytest and pytest-asyncio.** The test in this example uses them.

## Layout

```
src/app/
├── domain/          http.py  log.py  user.py
├── pure_functions/  user.py
├── component/       logger.py  http_requester.py  logging_requester.py
│                    user_getter.py  user_creator.py
├── wiring/          server.py
└── entrypoint/      server.py
tests/component/     test_logging_requester.py
```

## Domain

Each data type is a frozen dataclass, thus it is immutable.

`domain/http.py` shows the exception for a library data type that is not immutable. Code must not change an `HttpRequest` or an `HttpResponse`.

```python
# src/app/domain/http.py
import httpx

type HttpRequest = httpx.Request
type HttpResponse = httpx.Response
```

A data type with variants is a union of dataclasses.

```python
# src/app/domain/log.py
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class LogInfo:
    msg: str


@dataclass(frozen=True, slots=True)
class LogWarning:
    msg: str


@dataclass(frozen=True, slots=True)
class LogError:
    exception: BaseException


type LogRequest = LogInfo | LogWarning | LogError
```

`UserNotFound` is an error that goes between components, thus it is a domain type.

```python
# src/app/domain/user.py
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class User:
    id: str
    name: str
    email: str


@dataclass(frozen=True, slots=True)
class GetUser:
    id: str


@dataclass(frozen=True, slots=True)
class CreateUser:
    name: str
    email: str


@dataclass(frozen=True, slots=True)
class UserNotFound:
    id: str
```

## Pure functions

`user_from_json` is pure. `UserGetter` and `UserCreator` both use it, thus it is in `pure_functions/`.

```python
# src/app/pure_functions/user.py
from collections.abc import Mapping
from typing import Any

from app.domain.user import User


def user_from_json(data: Mapping[str, Any]) -> User:
    return User(id=str(data["id"]), name=str(data["name"]), email=str(data["email"]))
```

## Components

Each component is a class. Its constructor takes configuration first and dependencies after. A `type` statement in the file of the component gives a name to the type of each dependency.

`Logger` calls a library. It gets the name of the logger as configuration. `match` with `assert_never` makes mypy find a variant that the code does not handle.

```python
# src/app/component/logger.py
import logging
from typing import assert_never

from app.domain.log import LogError, LogInfo, LogRequest, LogWarning


class Logger:
    def __init__(self, name: str) -> None:
        self._logger = logging.getLogger(name)

    def log(self, request: LogRequest) -> None:
        match request:
            case LogInfo(msg=msg):
                self._logger.info(msg)
            case LogWarning(msg=msg):
                self._logger.warning(msg)
            case LogError(exception=exception):
                self._logger.error("Unhandled error", exc_info=exception)
            case _:
                assert_never(request)
```

`make_client` is a dependency. A test can give a client that uses `httpx.MockTransport`.

```python
# src/app/component/http_requester.py
from collections.abc import Callable

import httpx

from app.domain.http import HttpRequest, HttpResponse

type MakeClient = Callable[[], httpx.AsyncClient]


class HttpRequester:
    def __init__(self, make_client: MakeClient) -> None:
        self._make_client = make_client

    async def request(self, request: HttpRequest) -> HttpResponse:
        async with self._make_client() as client:
            return await client.send(request)
```

`log_success` is configuration. It changes the behaviour of `LoggingRequester`, but it does not change its types. `LoggingRequester` takes and gives the same type of callable, thus the wiring can put it in front of any `SendHttp`.

```python
# src/app/component/logging_requester.py
from collections.abc import Awaitable, Callable

from app.domain.http import HttpRequest, HttpResponse
from app.domain.log import LogInfo, LogWarning

type SendHttp = Callable[[HttpRequest], Awaitable[HttpResponse]]
type WriteInfo = Callable[[LogInfo], None]
type WriteWarning = Callable[[LogWarning], None]


class LoggingRequester:
    def __init__(
        self,
        log_success: bool,
        send: SendHttp,
        write_info: WriteInfo,
        write_warning: WriteWarning,
    ) -> None:
        self._log_success = log_success
        self._send = send
        self._write_info = write_info
        self._write_warning = write_warning

    async def request(self, request: HttpRequest) -> HttpResponse:
        response = await self._send(request)
        msg = f"{request.method} {request.url} -> {response.status_code}"
        if not response.is_success:
            self._write_warning(LogWarning(msg=msg))
        elif self._log_success:
            self._write_info(LogInfo(msg=msg))
        return response
```

`get_user` gives `UserNotFound` as a return value, not as an exception. Thus its type shows both results.

```python
# src/app/component/user_getter.py
from collections.abc import Awaitable, Callable

import httpx

from app.domain.http import HttpRequest, HttpResponse
from app.domain.user import GetUser, User, UserNotFound
from app.pure_functions.user import user_from_json

type SendHttp = Callable[[HttpRequest], Awaitable[HttpResponse]]


class UserGetter:
    def __init__(self, users_url: str, send: SendHttp) -> None:
        self._users_url = users_url
        self._send = send

    async def get_user(self, get_user: GetUser) -> User | UserNotFound:
        response = await self._send(httpx.Request("GET", f"{self._users_url}/{get_user.id}"))
        if response.status_code == 404:
            return UserNotFound(id=get_user.id)
        response.raise_for_status()
        return user_from_json(response.json())
```

```python
# src/app/component/user_creator.py
from collections.abc import Awaitable, Callable

import httpx

from app.domain.http import HttpRequest, HttpResponse
from app.domain.user import CreateUser, User
from app.pure_functions.user import user_from_json

type SendHttp = Callable[[HttpRequest], Awaitable[HttpResponse]]


class UserCreator:
    def __init__(self, users_url: str, send: SendHttp) -> None:
        self._users_url = users_url
        self._send = send

    async def create_user(self, create_user: CreateUser) -> User:
        body = {"name": create_user.name, "email": create_user.email}
        response = await self._send(httpx.Request("POST", self._users_url, json=body))
        response.raise_for_status()
        return user_from_json(response.json())
```

## Wiring

`build_server` is a function, thus to import the wiring does not build components. It chooses the configuration and the dependencies of each component. It gives each dependency as a bound method, and each argument by keyword.

`Logger.log` takes all `LogRequest` variants. mypy accepts it where a component needs only `LogInfo` or only `LogWarning`.

```python
# src/app/wiring/server.py
from collections.abc import Awaitable, Callable
from dataclasses import dataclass

import httpx

from app.component.http_requester import HttpRequester
from app.component.logger import Logger
from app.component.logging_requester import LoggingRequester
from app.component.user_creator import UserCreator
from app.component.user_getter import UserGetter
from app.domain.user import CreateUser, GetUser, User, UserNotFound


@dataclass(frozen=True, slots=True)
class Settings:
    users_url: str
    log_success: bool


@dataclass(frozen=True, slots=True)
class Server:
    get_user: Callable[[GetUser], Awaitable[User | UserNotFound]]
    create_user: Callable[[CreateUser], Awaitable[User]]


def build_server(settings: Settings) -> Server:
    logger = Logger(name="app")
    http = HttpRequester(make_client=httpx.AsyncClient)
    logged_http = LoggingRequester(
        log_success=settings.log_success,
        send=http.request,
        write_info=logger.log,
        write_warning=logger.log,
    )
    return Server(
        get_user=UserGetter(users_url=settings.users_url, send=logged_http.request).get_user,
        create_user=UserCreator(users_url=settings.users_url, send=logged_http.request).create_user,
    )
```

## Entrypoints

The entrypoint reads the settings and calls the wiring. FastAPI is the framework. The entrypoint converts the pydantic models of FastAPI to domain data, and converts domain data to these models.

```python
# src/app/entrypoint/server.py
import os

import fastapi
import pydantic

from app.domain.user import CreateUser, GetUser, User, UserNotFound
from app.wiring.server import Settings, build_server


class UserBody(pydantic.BaseModel):
    id: str
    name: str
    email: str


class CreateUserBody(pydantic.BaseModel):
    name: str
    email: str


def _to_body(user: User) -> UserBody:
    return UserBody(id=user.id, name=user.name, email=user.email)


server = build_server(
    Settings(
        users_url=os.environ["USERS_URL"],
        log_success=os.environ.get("LOG_SUCCESS") == "true",
    )
)
app = fastapi.FastAPI()


@app.get("/v1/users/{user_id}")
async def get_user(user_id: str) -> UserBody:
    match await server.get_user(GetUser(id=user_id)):
        case User() as user:
            return _to_body(user)
        case UserNotFound():
            raise fastapi.HTTPException(status_code=404)


@app.post("/v1/users")
async def create_user(body: CreateUserBody) -> UserBody:
    return _to_body(await server.create_user(CreateUser(name=body.name, email=body.email)))
```

## Tests

The test gives each dependency a fake. A fake is a function. `list.append` records the calls to a dependency.

```python
# tests/component/test_logging_requester.py
import httpx
import pytest

from app.component.logging_requester import LoggingRequester
from app.domain.http import HttpRequest, HttpResponse
from app.domain.log import LogWarning


@pytest.mark.asyncio
async def test_writes_a_warning_when_the_response_is_not_a_success() -> None:
    async def send(request: HttpRequest) -> HttpResponse:
        return httpx.Response(500, request=request)

    warnings: list[LogWarning] = []
    requester = LoggingRequester(
        log_success=False,
        send=send,
        write_info=lambda _: None,
        write_warning=warnings.append,
    )

    await requester.request(httpx.Request("GET", "https://users.test/1"))

    assert warnings == [LogWarning(msg="GET https://users.test/1 -> 500")]
```
