# Project Structure in TypeScript

The rules in [SKILL.md](SKILL.md) are language-agnostic. This file is their TypeScript spelling.

## Vocabulary → TypeScript

| Term | TypeScript |
|---|---|
| **Package** | a directory with an `index.ts` |
| **Export file** | the package's `index.ts`, re-exporting names |
| **Data carrier** | an `interface` or `type` of plain, `readonly` data |
| **Domain package** | `core/domain`, with one file per concept — `core/domain/order.ts` |
| **Factory package** | `infrastructure/factories/<module-name>`, with the factories in `factory.ts` |
| **Entrypoint** | a file in `infrastructure/entrypoints` — `infrastructure/entrypoints/cli.ts` |

**Package** here is a directory in one project, not an npm package. An ES module is a file; in these skills **module** keeps its codebase-design meaning, so this file says "file" for a `.ts` file.

Directories are kebab-case, so `<module_name>` in the rules is spelled `order-processor`.

## Example 1: a module and its adapters

An `OrderProcessor` module charges for an order through a `PaymentPort`. Stripe and PayPal each satisfy that port.

```
src/
  core/
    domain/
      order.ts                 # Order, OrderItem
      receipt.ts               # Receipt
    order-processor/
      index.ts                 # the export file
      interface.ts             # OrderProcessor
      implementation.ts        # newOrderProcessor; could be several files
      ports.ts                 # PaymentPort
  infrastructure/
    adapters/
      order-processor/
        stripe/
          index.ts             # the export file
          adapter.ts           # newStripeAdapter
          http.ts              # Http, newRealHttp: the adapter's own port and what satisfies it
        paypal/
          index.ts             # the export file
          adapter.ts           # newPaypalAdapter
          http.ts              # Http, newRealHttp
    factories/
      order-processor/
        index.ts               # the export file
        factory.ts             # createOrderProcessor
    entrypoints/
      cli.ts
```

The interface, the implementation and the ports each have their own file:

```ts
// src/core/order-processor/interface.ts
import type { Order } from "../domain/order";
import type { Receipt } from "../domain/receipt";

export type OrderProcessor = (order: Order) => Receipt; // Interface for the module
```

```ts
// src/core/order-processor/ports.ts
export interface PaymentPort { // Port defined for the module
  charge(amount: number): boolean;
}
```

```ts
// src/core/order-processor/implementation.ts
import type { OrderProcessor } from "./interface";
import type { PaymentPort } from "./ports";

export function newOrderProcessor(
  payment: PaymentPort,
): OrderProcessor { // Constructor for the module
  return function processOrder(order) { // Implementation of the module
    /* ... */
  };
}
```

```ts
// src/core/order-processor/index.ts
export { newOrderProcessor } from "./implementation";
export type { OrderProcessor } from "./interface";
export type { PaymentPort } from "./ports";
```

`Order` and `Receipt` appear in the `OrderProcessor` interface, so they are defined in the domain package from the start.

An adapter imports the port from the module's export file:

```ts
// src/infrastructure/adapters/order-processor/stripe/adapter.ts
import type { PaymentPort } from "../../../../core/order-processor";       // good: the export file
import type { PaymentPort } from "../../../../core/order-processor/ports"; // bad: reaches behind the export file
import { newPaypalAdapter } from "../paypal";                              // bad: an adapter importing another adapter
import type { Http } from "./http";                                        // good: the adapter's own port, defined in its own package

export function newStripeAdapter(apiKey: string, http: Http): PaymentPort { // Constructor for the stripe adapter
  return { // Adapter for the payment port
    charge(amount) { /* ... */ },
  };
}
```

Nothing in `core` imports from `infrastructure`:

```ts
// src/core/order-processor/implementation.ts
import { newStripeAdapter } from "../../infrastructure/adapters/order-processor/stripe"; // bad: the core importing an adapter
```

## Example 2: the factory package and an entrypoint

The factory that chooses an adapter imports the adapters, so it lives in `infrastructure/factories/order-processor`, not in `core/order-processor`:

```ts
// src/infrastructure/factories/order-processor/factory.ts
import { newOrderProcessor } from "../../../core/order-processor";
import type { OrderProcessor } from "../../../core/order-processor";
import { newPaypalAdapter } from "../../adapters/order-processor/paypal";
import { newStripeAdapter } from "../../adapters/order-processor/stripe";

export function createOrderProcessor(
  type: "stripe" | "paypal",
): OrderProcessor { // A factory for the order processor module
  switch (type) {
    case "stripe":
      return newOrderProcessor(newStripeAdapter(/* ... */));
    case "paypal":
      return newOrderProcessor(newPaypalAdapter(/* ... */));
  }
}
```

```ts
// src/infrastructure/factories/order-processor/index.ts
export { createOrderProcessor } from "./factory";
```

The entrypoint asks the factory for an entity and drives it through its interface. It calls no constructor and imports no adapter:

```ts
// src/infrastructure/entrypoints/cli.ts
import type { Order } from "../../core/domain/order";
import { createOrderProcessor } from "../factories/order-processor";

export function main(argv: string[]): void {
  const orderProcessor = createOrderProcessor("stripe");
  const order: Order = { /* ... */ };  // translate the outside world into a data carrier
  const receipt = orderProcessor(order);
  console.log(receipt);
}
```

## Example 3: a module that needs another module

A `Checkout` module is built on top of order processing. It does not import `core/order-processor`. It declares a port with the shape it needs:

```ts
// src/core/checkout/ports.ts
import type { Order } from "../domain/order";
import type { Receipt } from "../domain/receipt";

export interface OrderProcessingPort { // Port defined for the checkout module
  process(order: Order): Receipt;
}
```

An adapter under `infrastructure/adapters/checkout` satisfies the port by calling an `OrderProcessor` through its interface:

```ts
// src/infrastructure/adapters/checkout/order-processor/adapter.ts
import type { OrderProcessingPort } from "../../../../core/checkout";
import type { OrderProcessor } from "../../../../core/order-processor";

export function newOrderProcessorAdapter(
  orderProcessor: OrderProcessor, // Constructor takes the interface, never the implementation
): OrderProcessingPort { // Adapter connecting checkout to the order processor module
  return {
    process(order) {
      return orderProcessor(order);
    },
  };
}
```

The checkout factory gets an `OrderProcessor` from the order processor's factory package and passes it to the adapter:

```ts
// src/infrastructure/factories/checkout/factory.ts
import { newCheckout } from "../../../core/checkout";
import type { Checkout } from "../../../core/checkout";
import { newOrderProcessorAdapter } from "../../adapters/checkout/order-processor";
import { createOrderProcessor } from "../order-processor";

export function createCheckout(): Checkout { // A factory for the checkout module
  const orderProcessor = createOrderProcessor("stripe");
  return newCheckout(newOrderProcessorAdapter(orderProcessor));
}
```

## Example 4: the tests

```
tests/
  unit/
    core/
      order-processor/
        order-processor.test.ts    # constructs with newOrderProcessor and a fake payment adapter
        adapters.ts                # newFakePaymentAdapter: test adapters for this module's ports
      checkout/
        checkout.test.ts
        adapters.ts                # newFakeOrderProcessingAdapter
    infrastructure/
      adapters/
        order-processor/
          stripe/
            stripe.test.ts         # constructs with newStripeAdapter and a fake http
            adapters.ts            # newFakeHttp
  integration/
    infrastructure/
      adapters/
        order-processor/
          postgres/
            postgres.test.ts       # runs the postgres adapter against a real database
  e2e/
    cli.test.ts                    # runs the cli entrypoint with the production factories
```

A test imports the entity under test from its export file and its test adapters from beside itself:

```ts
// tests/unit/core/order-processor/order-processor.test.ts
import { newOrderProcessor } from "../../../../src/core/order-processor";
import { newFakePaymentAdapter } from "./adapters";
```

The tests for `Checkout` do not import `newFakePaymentAdapter` or anything else from the order processor's tests. `Checkout` has its own port, so its tests have their own test adapter.

## Checking the rules

ESLint's `no-restricted-imports` can enforce the export file by banning paths that reach past a package's `index.ts`. It can also keep constructors to the factories: constructors are named `new…`, so a pattern on `src/core` with `importNamePattern: "^new"` bans importing them, and an override switches the rule off for `src/infrastructure/factories` and `tests`. [dependency-cruiser](https://github.com/sverweij/dependency-cruiser) can enforce the import rules with `forbidden` rules: one stops `src/core` importing from `src/infrastructure`, one stops `src/core/domain` importing from any module's package, and one stops a module's package importing another module's package. A `forbidden` rule on npm dependencies can also stop `src/core` importing third-party libraries that reach a resource.
