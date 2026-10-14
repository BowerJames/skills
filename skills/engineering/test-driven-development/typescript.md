# Test-driven development in TypeScript

Below follows an example for a TDD approach towards building an `OrderProcessor` module. Nothing exists before the first test. Files are placed as [project-structure](../project-structure/SKILL.md) describes.

## First TDD Cycle

**Red**

The first thing we need is a constructor for producing order processors:

```ts
// tests/unit/core/order-processor/order-processor.test.ts
import { test } from "vitest";

import { newOrderProcessor } from "../../../../src/core/order-processor";

function setup() {
  return { orderProcessor: newOrderProcessor() };
}

test("can create an order processor", () => {
  setup();
});
```

At this point the tests will fail but they will fail for the 'wrong reason'. They will fail because `newOrderProcessor` does not exist to import. A true red state requires that the type checker passes but the tests are failing. Vitest does not type check, so `tsc --noEmit` is the compile step and is run alongside the tests. So the module's package is created with its interface, its implementation and its export file:

```ts
// src/core/order-processor/interface.ts
export interface OrderProcessor {}
```

```ts
// src/core/order-processor/implementation.ts
import type { OrderProcessor } from "./interface";

export function newOrderProcessor(): OrderProcessor {
  throw new Error("not implemented");
}
```

```ts
// src/core/order-processor/index.ts
export { newOrderProcessor } from "./implementation";
export type { OrderProcessor } from "./interface";
```

Now the test will fail for the right reason and the red state has been completed.

**Green**

```ts
// src/core/order-processor/implementation.ts
import type { OrderProcessor } from "./interface";

export function newOrderProcessor(): OrderProcessor {
  return {};
}
```

Green state achieved.

## Second TDD Cycle

**Red**

The next behaviour to pin would be that an order processor charges the total of the items in the order. The module has to charge something, so this behaviour needs a port. The module declares it, and a test adapter for it is written in the same cycle:

```ts
// tests/unit/core/order-processor/adapters.ts
import type { PaymentPort } from "../../../../src/core/order-processor";

export interface FakePaymentAdapter extends PaymentPort {
  charges: number[];
}

export function newFakePaymentAdapter(): FakePaymentAdapter {
  const charges: number[] = [];
  return {
    charges,
    charge(amount) {
      charges.push(amount);
      return true;
    },
  };
}
```

Then update the tests:

```ts
// tests/unit/core/order-processor/order-processor.test.ts
import { expect, test } from "vitest";

import type { Order } from "../../../../src/core/domain/order";
import { newOrderProcessor } from "../../../../src/core/order-processor";
import { newFakePaymentAdapter } from "./adapters";

function setup() {
  const payment = newFakePaymentAdapter();
  const orderProcessor = newOrderProcessor(payment);
  return { payment, orderProcessor };
}

function orderWith(prices: number[]): Order {
  return {
    id: "order_id",
    items: prices.map((price, i) => ({ name: `item_${i}`, cost: price })),
  };
}

test("can create an order processor", () => {
  setup();
});

test.each([
  { prices: [100], expectedCharge: 100 },
  { prices: [100, 40], expectedCharge: 140 },
])("charges total of all order items ($prices)", ({ prices, expectedCharge }) => {
  const { payment, orderProcessor } = setup();
  const order = orderWith(prices);
  orderProcessor(order);
  expect(payment.charges).toEqual([expectedCharge]);
});
```

Now the type checker needs to be made to pass. `Order` and `Receipt` cross the module's interface, so they are data carriers in the domain package:

```ts
// src/core/domain/order.ts
export interface OrderItem {
  readonly name: string;
  readonly cost: number;
}

export interface Order {
  readonly id: string;
  readonly items: readonly OrderItem[];
}
```

```ts
// src/core/domain/receipt.ts
export interface Receipt {
  readonly total: number;
}
```

The port goes in the module's ports file and the interface gains its call signature:

```ts
// src/core/order-processor/ports.ts
export interface PaymentPort {
  charge(amount: number): boolean;
}
```

```ts
// src/core/order-processor/interface.ts
import type { Order } from "../domain/order";
import type { Receipt } from "../domain/receipt";

export interface OrderProcessor {
  (order: Order): Receipt;
}
```

```ts
// src/core/order-processor/implementation.ts
import type { OrderProcessor } from "./interface";
import type { PaymentPort } from "./ports";

export function newOrderProcessor(payment: PaymentPort): OrderProcessor {
  return function processOrder(order) {
    throw new Error("not implemented");
  };
}
```

`PaymentPort` is added to the export file as a type export.

**Green**

```ts
// src/core/order-processor/implementation.ts
import type { OrderProcessor } from "./interface";
import type { PaymentPort } from "./ports";

export function newOrderProcessor(payment: PaymentPort): OrderProcessor {
  return function processOrder(order) {
    const total = order.items.reduce((sum, item) => sum + item.cost, 0);
    payment.charge(total);
    return { total };
  };
}
```

This completes the second cycle. And the cycles would continue until the required behaviours are all pinned.

## After the cycles

The module now exists in the core and has only ever been constructed with a test adapter. The Stripe adapter for `PaymentPort`, the factory that passes it to the constructor and the entrypoint are written next: see [project-structure](../project-structure/SKILL.md) for where they go and [factory-pattern](../factory-pattern/SKILL.md) for what the factory does.
