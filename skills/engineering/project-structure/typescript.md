# Project Structure in TypeScript

The rules in [SKILL.md](SKILL.md) are language-agnostic. This file is their TypeScript spelling.

## Vocabulary → TypeScript

| Term | TypeScript |
|---|---|
| **Package** | a directory with an `index.ts` |
| **Export file** | the package's `index.ts`, re-exporting names |
| **Data carrier** | an `interface` or `type` of plain, `readonly` data, or a union of literals |
| **Domain package** | `core/domain`, with one file per concept — `core/domain/currency.ts`, `core/domain/agent-tool.ts` |
| **Domain interface** | an `interface` in its own file in `core/domain` — `AgentTool` in `core/domain/agent-tool.ts` |
| **Module factory package** | `infrastructure/factories/modules/<module-name>`, with the factories in `factory.ts` |
| **Adapter factory package** | `infrastructure/factories/adapters/<module-name>/<adapter-name>`, or `<entrypoint-name>/<adapter-name>` for a driving adapter, or `<interface-name>/<adapter-name>` for an adapter of a domain interface, with the factories in `factory.ts` |
| **Entrypoint** | the framework or runtime that calls the program, never written in it: Fastify, Commander, or Node running a file, named `main` |
| **Driving adapter** | a function returned by a constructor in `infrastructure/adapters/<entrypoint-name>/<adapter-name>` — `newCli` in `infrastructure/adapters/main/cli/adapter.ts` |
| **Startup code** | a file in `infrastructure/startup`, one per entrypoint — `infrastructure/startup/main.ts` |

**Package** here is a directory in one project, not an npm package. An ES module is a file; in these skills **module** keeps its codebase-design meaning, so this file says "file" for a `.ts` file.

Directories are kebab-case, so `<module_name>` in the rules is spelled `order-processor`.

## Example 1: a module and its adapters

An `OrderProcessor` module charges for an order through its `OrderProcessorPort`. A Stripe adapter and a PayPal adapter each satisfy that port.

```
src/
  core/
    domain/
      currency.ts              # Currency
    order-processor/
      index.ts                 # the export file
      interface.ts             # OrderProcessor
      port.ts                  # OrderProcessorPort
      data.ts                  # Order, OrderItem, Receipt
      errors.ts                # PaymentDeclined
      implementation.ts        # newOrderProcessor; could be several files
  infrastructure/
    adapters/
      order-processor/
        stripe/
          index.ts             # the export file
          adapter.ts           # newStripeAdapter
          fetch.ts             # SyncFetch: its view of the sync-fetch library's function
        paypal/
          index.ts             # the export file
          adapter.ts           # newPaypalAdapter
          fetch.ts             # SyncFetch
      main/                    # the driving adapters of Node running a file
        cli/
          index.ts             # the export file
          adapter.ts           # newCli
    factories/
      modules/
        order-processor/
          index.ts             # the export file
          factory.ts           # createOrderProcessor
      adapters/
        order-processor/
          stripe/
            index.ts           # the export file
            factory.ts         # createStripeAdapter
          paypal/
            index.ts           # the export file
            factory.ts         # createPaypalAdapter
        main/
          cli/
            index.ts           # the export file
            factory.ts         # createCli
    startup/
      main.ts                  # the startup code for Node running a file
```

`Currency` means the same thing to every module and belongs to none, so it is in the domain package. `Order` and `Receipt` appear in the `OrderProcessor` interface, so they are in the module's data file:

```ts
// src/core/domain/currency.ts
export type Currency = "GBP" | "USD" | "EUR";
```

```ts
// src/core/order-processor/data.ts
import type { Currency } from "../domain/currency";

export interface OrderItem {
  readonly name: string;
  readonly cost: number; // in the currency's minor unit
}

export interface Order {
  readonly id: string;
  readonly items: readonly OrderItem[];
}

export interface Receipt {
  readonly total: number;
  readonly currency: Currency;
}
```

The interface, the port, the errors and the implementation each have their own file:

```ts
// src/core/order-processor/interface.ts
import type { Order, Receipt } from "./data";

export type OrderProcessor = (order: Order) => Receipt; // Interface for the module
```

```ts
// src/core/order-processor/port.ts
import type { Currency } from "../domain/currency";

export interface OrderProcessorPort { // The module's port
  charge(amount: number, currency: Currency): boolean;
}
```

```ts
// src/core/order-processor/errors.ts
export class PaymentDeclined extends Error {} // Determined by the module's implementation
```

```ts
// src/core/order-processor/implementation.ts
import type { Currency } from "../domain/currency";
import type { OrderProcessor } from "./interface";
import type { OrderProcessorPort } from "./port";

export function newOrderProcessor(
  port: OrderProcessorPort,
  currency: Currency,
): OrderProcessor { // Constructor for the module
  return function processOrder(order) { // Implementation of the module
    /* ... */
  };
}
```

```ts
// src/core/order-processor/index.ts
export type { Order, OrderItem, Receipt } from "./data";
export { PaymentDeclined } from "./errors";
export { newOrderProcessor } from "./implementation";
export type { OrderProcessor } from "./interface";
export type { OrderProcessorPort } from "./port";
```

An adapter imports the port from the module's export file:

```ts
// src/infrastructure/adapters/order-processor/stripe/adapter.ts
import type { OrderProcessorPort } from "../../../../core/order-processor";      // good: the export file
import type { OrderProcessorPort } from "../../../../core/order-processor/port"; // bad: reaches behind the export file
import { newPaypalAdapter } from "../paypal";                                    // bad: an adapter importing another adapter
import type { SyncFetch } from "./fetch";                                        // good: its view of the library, declared in its own package

export function newStripeAdapter(apiKey: string, fetch: SyncFetch): OrderProcessorPort { // Constructor: takes the library's function
  return { // Adapter for the module's port
    charge(amount, currency) { /* ... */ },
  };
}
```

Nothing in `core` imports from `infrastructure`:

```ts
// src/core/order-processor/implementation.ts
import { newStripeAdapter } from "../../infrastructure/adapters/order-processor/stripe"; // bad: the core importing an adapter
```

## Example 2: the factory packages, a driving adapter and its startup code

The adapter's export file exports its constructor and its view, which is what its adapter factory and its tests need:

```ts
// src/infrastructure/adapters/order-processor/stripe/index.ts
export { newStripeAdapter } from "./adapter";
export type { SyncFetch } from "./fetch";
```

The adapter factory passes the library objects the adapter takes and calls its constructor. It lives under `infrastructure/factories/adapters`, at the adapter's path:

```ts
// src/infrastructure/factories/adapters/order-processor/stripe/factory.ts
import syncFetch from "sync-fetch";
import type { OrderProcessorPort } from "../../../../../core/order-processor";
import { newStripeAdapter } from "../../../../adapters/order-processor/stripe";

export function createStripeAdapter(apiKey: string): OrderProcessorPort { // An adapter factory: returns the port the adapter satisfies
  return newStripeAdapter(apiKey, syncFetch); // The library's function satisfies the view by shape
}
```

```ts
// src/infrastructure/factories/adapters/order-processor/stripe/index.ts
export { createStripeAdapter } from "./factory";
```

The module factory chooses an adapter by calling its adapter factory, and never imports from an adapter's package:

```ts
// src/infrastructure/factories/modules/order-processor/factory.ts
import type { Currency } from "../../../../core/domain/currency";
import { newOrderProcessor } from "../../../../core/order-processor";
import type { OrderProcessor } from "../../../../core/order-processor";
import { createPaypalAdapter } from "../../adapters/order-processor/paypal";
import { createStripeAdapter } from "../../adapters/order-processor/stripe";
import { newStripeAdapter } from "../../../adapters/order-processor/stripe"; // bad: a module factory reaching into an adapter's package

export function createOrderProcessor(
  provider: "stripe" | "paypal",
  apiKey: string,
  currency: Currency,
): OrderProcessor { // A module factory for the order processor module
  switch (provider) {
    case "stripe":
      return newOrderProcessor(createStripeAdapter(apiKey), currency);
    case "paypal":
      return newOrderProcessor(createPaypalAdapter(apiKey), currency);
  }
}
```

```ts
// src/infrastructure/factories/modules/order-processor/index.ts
export { createOrderProcessor } from "./factory";
```

The entrypoint here is Node itself, running the startup file; the project does not write it. What it reaches is a driving adapter: the function `newCli` returns has the shape of a command, a function given the command-line arguments. Its constructor takes the module interface it drives, and it translates between the arguments and data carriers:

```ts
// src/infrastructure/adapters/main/cli/adapter.ts
import type { Order, OrderProcessor } from "../../../../core/order-processor";

export function newCli(
  orderProcessor: OrderProcessor, // Its constructor takes the interface it drives
): (argv: string[]) => void { // A driving adapter: the shape the entrypoint calls, a command
  return function cli(argv) { // calls no module's constructor and no factory, imports no other adapter
    const [id, ...costs] = argv;
    const order: Order = {
      id,
      items: costs.map((cost, i) => ({ name: `item_${i}`, cost: Number(cost) })),
    }; // translate the outside world into a data carrier
    const receipt = orderProcessor(order);
    console.log(`total=${receipt.total} ${receipt.currency}`);
  };
}
```

Like every adapter, it has an adapter factory:

```ts
// src/infrastructure/factories/adapters/main/cli/factory.ts
import type { OrderProcessor } from "../../../../../core/order-processor";
import { newCli } from "../../../../adapters/main/cli";

export function createCli(orderProcessor: OrderProcessor): (argv: string[]) => void { // Returns the entrypoint's port: a command
  return newCli(orderProcessor);
}
```

The startup code reads the environment, calls the factories, and hands the driving adapter to the entrypoint. For Node running a file, that means calling it with the arguments:

```ts
// src/infrastructure/startup/main.ts
import { createCli } from "../factories/adapters/main/cli";
import { createOrderProcessor } from "../factories/modules/order-processor";

const cli = createCli(createOrderProcessor("stripe", process.env.STRIPE_API_KEY!, "GBP"));
cli(process.argv.slice(2));
```

With a framework as the entrypoint, the startup code registers the driving adapters with it instead, and the framework defines the port. For Fastify, a driving adapter is a plugin, and the startup code registers each with the app:

```ts
// src/infrastructure/startup/fastify.ts
import Fastify from "fastify";
import { createOrdersRoutes } from "../factories/adapters/fastify/orders-routes";
import { createOrderProcessor } from "../factories/modules/order-processor";

const app = Fastify(); // The entrypoint, configured
app.register(createOrdersRoutes(createOrderProcessor("stripe", process.env.STRIPE_API_KEY!, "GBP")));
await app.listen({ port: 3000 });
```

When the entity a driving adapter drives holds one request's resources, the driving adapter takes the opening of it, and the startup code satisfies it with a module factory. A `PhoneCall` module whose entity talks to a caller over their socket is built per call:

```ts
// src/infrastructure/adapters/fastify/call-routes/adapter.ts
import type { FastifyPluginAsync } from "fastify";
import type { WebSocket } from "ws";
import type { CallDetails, PhoneCall } from "../../../../core/phone-call";

export type OpenPhoneCall = (socket: WebSocket, call: CallDetails) => PhoneCall; // Opens the entity per request: a function returning the module's interface

export function newCallRoutes(path: string, openCall: OpenPhoneCall): FastifyPluginAsync { // Its constructor: returns the plugin Fastify calls
  return async (app) => {
    app.get<{ Querystring: { ddiNo: string } }>(path, { websocket: true }, async (socket, request) => {
      await openCall(socket, { ddiNumber: request.query.ddiNo }).run();
    });
  };
}
```

```ts
// src/infrastructure/startup/fastify.ts
app.register(fastifyWebsocket);
app.register(createCallRoutes("/call", createPhoneCall)); // The adapter factory, given the phone call's module factory as the opener
```

## Example 3: a module that needs another module

A `Checkout` module is built on top of order processing. It does not import `core/order-processor`. It declares its own data carriers and a port with the shape it needs:

```ts
// src/core/checkout/data.ts
export interface CartLine {
  readonly product: string;
  readonly price: number;
}

export interface Cart {
  readonly id: string;
  readonly lines: readonly CartLine[];
}
```

```ts
// src/core/checkout/port.ts
import type { Cart } from "./data";

export interface CheckoutPort { // The checkout module's port
  pay(cart: Cart): number;
}
```

An adapter under `infrastructure/adapters/checkout` satisfies the port by calling an `OrderProcessor` through its interface, translating between the two modules' data carriers:

```ts
// src/infrastructure/adapters/checkout/order-processor/adapter.ts
import type { CheckoutPort } from "../../../../core/checkout";
import type { Order, OrderProcessor } from "../../../../core/order-processor";

export function newOrderProcessorAdapter(
  orderProcessor: OrderProcessor, // Constructor takes the interface, never the implementation
): CheckoutPort { // Adapter connecting checkout to the order processor module
  return {
    pay(cart) {
      const order: Order = {
        id: cart.id,
        items: cart.lines.map((line) => ({ name: line.product, cost: line.price })),
      }; // checkout's data, translated into the order processor's
      return orderProcessor(order).total;
    },
  };
}
```

The adapter's factory gets an `OrderProcessor` from the order processor's module factory package and passes it to the adapter's constructor:

```ts
// src/infrastructure/factories/adapters/checkout/order-processor/factory.ts
import type { CheckoutPort } from "../../../../../core/checkout";
import { newOrderProcessorAdapter } from "../../../../adapters/checkout/order-processor";
import { createOrderProcessor } from "../../../modules/order-processor";

export function createOrderProcessorAdapter(apiKey: string): CheckoutPort { // An adapter factory for the connecting adapter
  return newOrderProcessorAdapter(createOrderProcessor("stripe", apiKey, "GBP"));
}
```

The checkout module factory gets its adapter from that factory:

```ts
// src/infrastructure/factories/modules/checkout/factory.ts
import { newCheckout } from "../../../../core/checkout";
import type { Checkout } from "../../../../core/checkout";
import { createOrderProcessorAdapter } from "../../adapters/checkout/order-processor";

export function createCheckout(apiKey: string): Checkout { // A module factory for the checkout module
  return newCheckout(createOrderProcessorAdapter(apiKey));
}
```

## Example 4: the tests

```
tests/
  unit/
    core/
      order-processor/
        order-processor.test.ts    # constructs with newOrderProcessor and a fake adapter
        adapters.ts                # newFakeOrderProcessorAdapter: the test adapter for this module's port
      checkout/
        checkout.test.ts
        adapters.ts                # newFakeCheckoutAdapter
    infrastructure/
      adapters/
        order-processor/
          stripe/
            stripe.test.ts         # constructs with newStripeAdapter and a fake of its view
            adapters.ts            # newFakeFetch: the test adapter for the view
        checkout/
          order-processor/
            order-processor.test.ts  # constructs with newOrderProcessorAdapter and a fake order processor
            adapters.ts              # newFakeOrderProcessor
        main/
          cli/
            cli.test.ts            # constructs with newCli and a fake order processor
            adapters.ts            # newFakeOrderProcessor
  integration/
    infrastructure/
      adapters/
        order-processor/
          postgres/
            postgres.test.ts       # runs the postgres adapter against a real database
  e2e/
    main.test.ts                   # where the project has end-to-end tests: runs infrastructure/startup/main.ts
```

A test imports the entity under test from its export file and its test adapters from beside itself:

```ts
// tests/unit/core/order-processor/order-processor.test.ts
import { newOrderProcessor } from "../../../../src/core/order-processor";
import { newFakeOrderProcessorAdapter } from "./adapters";
```

The tests for `Checkout` do not import `newFakeOrderProcessorAdapter` or anything else from the order processor's tests. `Checkout` has its own port, so its tests have their own test adapter. The two `newFakeOrderProcessor` test adapters are duplicated for the same reason.

## Example 5: legacy code

A project has a legacy `billing.ts` that already talks to Stripe. A new `OrderProcessor` module reaches it through its port, with an adapter named for it:

```
src/
  billing.ts                     # legacy code, left where it is
  core/
    order-processor/
  infrastructure/
    adapters/
      order-processor/
        legacy-billing/
          index.ts
          adapter.ts             # newLegacyBillingAdapter: satisfies OrderProcessorPort by calling billing.ts
    factories/
      adapters/
        order-processor/
          legacy-billing/
            index.ts
            factory.ts           # createLegacyBillingAdapter
```

```ts
// src/infrastructure/adapters/order-processor/legacy-billing/adapter.ts
import { takePayment } from "../../../../billing";
import type { OrderProcessorPort } from "../../../../core/order-processor";

export function newLegacyBillingAdapter(): OrderProcessorPort {
  return {
    charge(amount, currency) {
      return takePayment(amount / 100, currency) === "ok";
    },
  };
}
```

Legacy code that needs an `OrderProcessor` calls the factory and drives the interface:

```ts
// src/views.ts, legacy code
import { createOrderProcessor } from "./infrastructure/factories/modules/order-processor"; // good: a module factory
import { newOrderProcessor } from "./core/order-processor";                                // bad: the constructor
```

## Example 6: a domain interface and its adapters

A `ToolRunner` module runs the tools a model calls. A tool means the same thing to every entity that uses it, so `AgentTool` is a domain interface, and every tool is an adapter of it:

```
src/
  core/
    domain/
      agent-tool.ts              # AgentTool: a domain interface
    tool-runner/
      port.ts                    # ToolRunnerPort: tools(), using AgentTool
  infrastructure/
    adapters/
      agent-tool/                # the adapters of the AgentTool domain interface
        order-processor/
          adapter.ts             # newPlaceOrderTool: places an order through the OrderProcessor module
          params.ts              # placeOrderParams: the zod schema of its arguments
        structured/
          adapter.ts             # newStructuredTool: validates arguments against a zod schema
      tool-runner/
        toolset/
          adapter.ts             # newToolset: offers the runner the tools it is given
    factories/
      adapters/
        agent-tool/
          order-processor/       # createPlaceOrderTool
          structured/            # createStructuredTool
        tool-runner/
          toolset/               # createToolset
      modules/
        tool-runner/             # createToolRunner
```

The domain interface has a shape and no behaviour. The module's one port uses it:

```ts
// src/core/domain/agent-tool.ts
export interface AgentTool { // A domain interface: the same meaning to every entity that uses it
  readonly name: string;
  readonly description: string;
  readonly parameters: Readonly<Record<string, unknown>>; // the JSON Schema of execute's arguments
  execute(args: Record<string, unknown>): Promise<string>;
}
```

```ts
// src/core/tool-runner/port.ts
import type { AgentTool } from "../domain/agent-tool";

export interface ToolRunnerPort { // The module's one port; it uses the domain interface
  tools(): readonly AgentTool[];
}
```

Each tool is an adapter of the domain interface, under `infrastructure/adapters/agent-tool`. One connects the order processor module, so it is named for that module:

```ts
// src/infrastructure/adapters/agent-tool/order-processor/adapter.ts
import type { AgentTool } from "../../../../core/domain/agent-tool";
import type { OrderProcessor } from "../../../../core/order-processor";

export function newPlaceOrderTool(orderProcessor: OrderProcessor): AgentTool { // Connects the order processor module
  return {
    name: "place_order",
    description: "Place the caller's order.",
    parameters: { type: "object", properties: { id: { type: "string" }, costs: { type: "array" } } },
    async execute(args) {
      const costs = args.costs as number[];
      const receipt = orderProcessor({
        id: args.id as string,
        items: costs.map((cost, i) => ({ name: `item_${i}`, cost })),
      });
      return `Charged ${receipt.total} ${receipt.currency}.`;
    },
  };
}
```

A generic adapter of the same interface is reused by wrapping, never by importing. `newStructuredTool` validates the arguments against a zod schema, then passes them to the tool it wraps:

```ts
// src/infrastructure/adapters/agent-tool/structured/adapter.ts
import { z } from "zod";
import type { AgentTool } from "../../../../core/domain/agent-tool";

export function newStructuredTool(
  params: z.ZodObject,
  inner: AgentTool, // The tool it wraps, typed as the interface
): AgentTool { // A generic adapter of the domain interface
  return {
    name: inner.name,
    description: inner.description,
    parameters: z.toJSONSchema(params), // The schema comes from the zod schema, not the inner tool
    async execute(args) {
      const parsed = params.safeParse(args);
      if (!parsed.success) {
        return JSON.stringify(parsed.error.issues); // handled: the model reads which arguments to correct
      }
      return inner.execute(parsed.data);
    },
  };
}
```

The factories combine them. A tool's factory gets the generic adapter from its factory, and the module factory gets the toolset from its own:

```ts
// src/infrastructure/factories/adapters/agent-tool/order-processor/factory.ts
import type { AgentTool } from "../../../../../core/domain/agent-tool";
import type { OrderProcessor } from "../../../../../core/order-processor";
import { newPlaceOrderTool, placeOrderParams } from "../../../../adapters/agent-tool/order-processor";
import { createStructuredTool } from "../structured";

export function createPlaceOrderTool(orderProcessor: OrderProcessor): AgentTool { // Returns the domain interface it satisfies
  return createStructuredTool(placeOrderParams, newPlaceOrderTool(orderProcessor));
}
```

```ts
// src/infrastructure/factories/modules/tool-runner/factory.ts
import type { OrderProcessor } from "../../../../core/order-processor";
import { newToolRunner } from "../../../../core/tool-runner";
import type { ToolRunner } from "../../../../core/tool-runner";
import { createPlaceOrderTool } from "../../adapters/agent-tool/order-processor";
import { createToolset } from "../../adapters/tool-runner/toolset";

export function createToolRunner(orderProcessor: OrderProcessor): ToolRunner {
  return newToolRunner(createToolset([createPlaceOrderTool(orderProcessor)]));
}
```

The tests for `ToolRunner` pass a fake port returning fake `AgentTool`s; each tool adapter is tested on its own, through the domain interface, under `tests/unit/infrastructure/adapters/agent-tool/<adapter-name>`.

## Checking the rules

ESLint's `no-restricted-imports` can enforce the export file by banning paths that reach past a package's `index.ts`. It can also keep constructors to the factories: constructors are named `new…`, so a pattern on `src/core` with `importNamePattern: "^new"` bans importing them, with an override switching the rule off for `src/infrastructure/factories/modules` and `tests`. The same pattern on `src/infrastructure/adapters`, switched off for `src/infrastructure/factories/adapters` and `tests`, keeps adapter constructors to their adapter factories. [dependency-cruiser](https://github.com/sverweij/dependency-cruiser) can enforce the import rules with `forbidden` rules: one stops `src/core` importing from `src/infrastructure`, one stops `src/core/domain` importing from any module's package, and one stops a module's package importing another module's package. A `forbidden` rule on npm dependencies can also stop `src/core` importing any third-party library.
