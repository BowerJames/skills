# TypeScript

This file shows one way to follow the rules in [codebase-design](../SKILL.md) in TypeScript. Use each term with the meaning in the [glossary](../GLOSSARY.md).

Where the rules do not decide, this example does not decide either. The example shows one choice. It is not the only choice.

## Tools

Recommend these tools to the user. Do not add them to a project without the agreement of the user.

- **TypeScript with `"strict": true`.** The `strict` option includes `strictFunctionTypes`. With it, `tsc` checks that the wiring gives each component callables of the correct type.
- **dependency-cruiser.** dependency-cruiser checks the import direction of the layers, and that a component does not import a different component:

  ```js
  // .dependency-cruiser.cjs
  /** @type {import("dependency-cruiser").IConfiguration} */
  module.exports = {
    forbidden: [
      {
        name: "domain-imports-only-to-its-right",
        severity: "error",
        from: { path: "^src/domain/" },
        to: { path: "^src/(pure_functions|component|wiring|entrypoint)/" },
      },
      {
        name: "pure-functions-import-only-to-their-right",
        severity: "error",
        from: { path: "^src/pure_functions/" },
        to: { path: "^src/(component|wiring|entrypoint)/" },
      },
      {
        name: "component-imports-only-to-its-right",
        severity: "error",
        from: { path: "^src/component/" },
        to: { path: "^src/(wiring|entrypoint)/" },
      },
      {
        name: "wiring-imports-only-to-its-right",
        severity: "error",
        from: { path: "^src/wiring/" },
        to: { path: "^src/entrypoint/" },
      },
      {
        name: "component-imports-no-other-component",
        severity: "error",
        from: { path: "^src/component/([^/]+)" },
        to: { path: "^src/component/", pathNot: "^src/component/$1" },
      },
    ],
    options: {
      parser: "swc",
      doNotFollow: { path: "node_modules" },
      tsConfig: { fileName: "tsconfig.json" },
    },
  };
  ```

  With TypeScript 7, dependency-cruiser needs `@swc/core` and `parser: "swc"`. Without them, dependency-cruiser checks no files and reports no errors. Make sure that the output shows more than 0 modules.

- **vitest.** The test in this example uses it.

## Layout

```
src/
├── domain/          http.ts  log.ts  user.ts
├── pure_functions/  user.ts
├── component/       logger.ts  logging-requester.ts
│                    user-getter.ts  user-creator.ts
├── wiring/          server.ts
└── entrypoint/      server.ts
tests/component/     logging-requester.test.ts
```

## Domain

Each data type is a `type` with `Readonly<>`, thus it is immutable. `Readonly<>` does not make a nested array immutable. For an array field, use `ReadonlyArray<T>`.

Each data type has a `kind` field. TypeScript compares types by their fields. Without a `kind` field, two data types with the same fields are the same type.

`domain/http.ts` shows the exception for a library data type that is not immutable. These aliases have no `kind` field. Code must not change an `HttpRequest` or an `HttpResponse`. Code must not read the body of an `HttpResponse` that a different component reads.

```ts
// src/domain/http.ts
export type HttpRequest = Request;
export type HttpResponse = Response;
```

A data type with variants is a union of data types.

```ts
// src/domain/log.ts
export type LogInfo = Readonly<{ kind: "LogInfo"; msg: string }>;
export type LogWarning = Readonly<{ kind: "LogWarning"; msg: string }>;
export type LogError = Readonly<{ kind: "LogError"; error: unknown }>;

export type LogRequest = LogInfo | LogWarning | LogError;
```

`UserNotFound` is an error that goes between components, thus it is a domain type.

```ts
// src/domain/user.ts
export type User = Readonly<{ kind: "User"; id: string; name: string; email: string }>;
export type GetUser = Readonly<{ kind: "GetUser"; id: string }>;
export type CreateUser = Readonly<{ kind: "CreateUser"; name: string; email: string }>;
export type UserNotFound = Readonly<{ kind: "UserNotFound"; id: string }>;
```

## Pure functions

`userFromJson` is pure. It uses zod, a library. `UserGetter` and `UserCreator` both use it, thus it is in `pure_functions/`.

```ts
// src/pure_functions/user.ts
import { z } from "zod";

import type { User } from "../domain/user";

const UserJson = z.object({ id: z.string(), name: z.string(), email: z.string() });

export function userFromJson(data: unknown): User {
  const { id, name, email } = UserJson.parse(data);
  return { kind: "User", id, name, email };
}
```

## Components

Each component has an `interface` and a factory function. The interface shows what the component gives. The factory function is the constructor. It takes one object with the configuration and the dependencies.

Write each member of an interface, and each dependency type, as a function type: `readonly log: (request: LogRequest) => void`. Do not use the method form `log(request: LogRequest): void`. TypeScript checks the method form less strictly, thus `tsc` can miss an error in the wiring.

The factory function gives closures, not methods. Thus the wiring can give `logger.log` as a callable, and `this` does not change.

`Logger` calls a library. It gets the name of the logger as configuration. `request satisfies never` makes `tsc` find a variant that the code does not handle.

```ts
// src/component/logger.ts
import type { LogRequest } from "../domain/log";

export interface Logger {
  readonly log: (request: LogRequest) => void;
}

export function makeLogger(options: { readonly name: string }): Logger {
  const { name } = options;
  return {
    log: (request) => {
      switch (request.kind) {
        case "LogInfo":
          console.info(`[${name}] ${request.msg}`);
          return;
        case "LogWarning":
          console.warn(`[${name}] ${request.msg}`);
          return;
        case "LogError":
          console.error(`[${name}] Unhandled error`, request.error);
          return;
        default:
          request satisfies never;
      }
    },
  };
}
```

`logSuccess` is configuration. It changes the behaviour of `LoggingRequester`, but it does not change its types. `LoggingRequester` takes and gives the same type of callable, thus the wiring can put it in front of any `SendHttp`.

```ts
// src/component/logging-requester.ts
import type { HttpRequest, HttpResponse } from "../domain/http";
import type { LogInfo, LogWarning } from "../domain/log";

type SendHttp = (request: HttpRequest) => Promise<HttpResponse>;
type WriteInfo = (log: LogInfo) => void;
type WriteWarning = (log: LogWarning) => void;

export interface LoggingRequester {
  readonly request: (request: HttpRequest) => Promise<HttpResponse>;
}

export function makeLoggingRequester(options: {
  readonly logSuccess: boolean;
  readonly send: SendHttp;
  readonly writeInfo: WriteInfo;
  readonly writeWarning: WriteWarning;
}): LoggingRequester {
  const { logSuccess, send, writeInfo, writeWarning } = options;
  return {
    request: async (request) => {
      const response = await send(request);
      const msg = `${request.method} ${request.url} -> ${response.status}`;
      if (!response.ok) {
        writeWarning({ kind: "LogWarning", msg });
      } else if (logSuccess) {
        writeInfo({ kind: "LogInfo", msg });
      }
      return response;
    },
  };
}
```

`getUser` gives `UserNotFound` as a return value, not as an exception. Thus its type shows both results.

```ts
// src/component/user-getter.ts
import type { HttpRequest, HttpResponse } from "../domain/http";
import type { GetUser, User, UserNotFound } from "../domain/user";
import { userFromJson } from "../pure_functions/user";

type SendHttp = (request: HttpRequest) => Promise<HttpResponse>;

export interface UserGetter {
  readonly getUser: (getUser: GetUser) => Promise<User | UserNotFound>;
}

export function makeUserGetter(options: {
  readonly usersUrl: string;
  readonly send: SendHttp;
}): UserGetter {
  const { usersUrl, send } = options;
  return {
    getUser: async (getUser) => {
      const response = await send(new Request(`${usersUrl}/${getUser.id}`));
      if (response.status === 404) {
        return { kind: "UserNotFound", id: getUser.id };
      }
      if (!response.ok) {
        throw new Error(`Get user failed with status ${response.status}`);
      }
      return userFromJson(await response.json());
    },
  };
}
```

```ts
// src/component/user-creator.ts
import type { HttpRequest, HttpResponse } from "../domain/http";
import type { CreateUser, User } from "../domain/user";
import { userFromJson } from "../pure_functions/user";

type SendHttp = (request: HttpRequest) => Promise<HttpResponse>;

export interface UserCreator {
  readonly createUser: (createUser: CreateUser) => Promise<User>;
}

export function makeUserCreator(options: {
  readonly usersUrl: string;
  readonly send: SendHttp;
}): UserCreator {
  const { usersUrl, send } = options;
  return {
    createUser: async (createUser) => {
      const response = await send(
        new Request(usersUrl, {
          method: "POST",
          headers: { "content-type": "application/json" },
          body: JSON.stringify({ name: createUser.name, email: createUser.email }),
        }),
      );
      if (!response.ok) {
        throw new Error(`Create user failed with status ${response.status}`);
      }
      return userFromJson(await response.json());
    },
  };
}
```

## Wiring

`buildServer` is a function, thus to import the wiring does not build components. It chooses the configuration and the dependencies of each component. It gives each dependency as a member of a component or as a function, and each argument by name.

The global `fetch` is the `send` dependency of `LoggingRequester`. `fetch` shares its connections, and nothing must close it, thus `Server` has no `close`.

`logger.log` takes all `LogRequest` variants. `tsc` accepts it where a component needs only `LogInfo` or only `LogWarning`.

```ts
// src/wiring/server.ts
import { makeLogger } from "../component/logger";
import { makeLoggingRequester } from "../component/logging-requester";
import { makeUserCreator, type UserCreator } from "../component/user-creator";
import { makeUserGetter, type UserGetter } from "../component/user-getter";

export type Settings = Readonly<{ usersUrl: string; logSuccess: boolean }>;

export interface Server {
  readonly getUser: UserGetter["getUser"];
  readonly createUser: UserCreator["createUser"];
}

export function buildServer(settings: Settings): Server {
  const logger = makeLogger({ name: "app" });
  const loggedHttp = makeLoggingRequester({
    logSuccess: settings.logSuccess,
    send: fetch,
    writeInfo: logger.log,
    writeWarning: logger.log,
  });
  return {
    getUser: makeUserGetter({ usersUrl: settings.usersUrl, send: loggedHttp.request }).getUser,
    createUser: makeUserCreator({ usersUrl: settings.usersUrl, send: loggedHttp.request }).createUser,
  };
}
```

## Entrypoints

The entrypoint reads the settings and calls the wiring. Hono is the framework. zod checks the request body. The entrypoint converts the request data of Hono to domain data, and converts domain data to the response data of Hono.

```ts
// src/entrypoint/server.ts
import { serve } from "@hono/node-server";
import { zValidator } from "@hono/zod-validator";
import { Hono } from "hono";
import { z } from "zod";

import type { User } from "../domain/user";
import { buildServer } from "../wiring/server";

const CreateUserBody = z.object({ name: z.string(), email: z.string() });

type UserBody = Readonly<{ id: string; name: string; email: string }>;

function toBody(user: User): UserBody {
  return { id: user.id, name: user.name, email: user.email };
}

const usersUrl = process.env.USERS_URL;
if (usersUrl === undefined) {
  throw new Error("USERS_URL is not set");
}

const server = buildServer({
  usersUrl,
  logSuccess: process.env.LOG_SUCCESS === "true",
});

const app = new Hono();

app.get("/v1/users/:userId", async (c) => {
  const result = await server.getUser({ kind: "GetUser", id: c.req.param("userId") });
  switch (result.kind) {
    case "User":
      return c.json(toBody(result));
    case "UserNotFound":
      return c.json({ error: "User not found" }, 404);
  }
});

app.post("/v1/users", zValidator("json", CreateUserBody), async (c) => {
  const body = c.req.valid("json");
  const user = await server.createUser({ kind: "CreateUser", name: body.name, email: body.email });
  return c.json(toBody(user));
});

serve({ fetch: app.fetch, port: 3000 });
```

## Tests

The test gives each dependency a fake. A fake is a function. An array records the calls to a dependency. Give `(log) => warnings.push(log)`, not `warnings.push`: a method of an array that is given alone loses `this`.

```ts
// tests/component/logging-requester.test.ts
import { expect, test } from "vitest";

import { makeLoggingRequester } from "../../src/component/logging-requester";
import type { LogWarning } from "../../src/domain/log";

test("writes a warning when the response is not a success", async () => {
  const warnings: LogWarning[] = [];
  const requester = makeLoggingRequester({
    logSuccess: false,
    send: async () => new Response(null, { status: 500 }),
    writeInfo: () => {},
    writeWarning: (log) => warnings.push(log),
  });

  await requester.request(new Request("https://users.test/1"));

  expect(warnings).toEqual([{ kind: "LogWarning", msg: "GET https://users.test/1 -> 500" }]);
});
```
