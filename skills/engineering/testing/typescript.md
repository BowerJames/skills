# Testing in TypeScript

The sections in [SKILL.md](SKILL.md) are language-agnostic. This file is its TypeScript spelling. The vocabulary's TypeScript shapes — interfaces, function types, object literals — live in [codebase-design's typescript.md](../codebase-design/typescript.md).

## What makes a good test

We will look at a good test for an `OrderProcessor` module. Consider the setup:

```ts
interface PaymentPort { // Port defined for the module
  charge(amount: number): boolean;
}

type OrderProcessor = (order: Order) => Receipt;

function newOrderProcessor(
  payment: PaymentPort, // The port is passed to the constructor of the OrderProcessor module
): OrderProcessor {
  return function processOrder(order) {
    let total = 0;
    for (const item of order.items) {
      total += item.cost;
    }
    const paymentSuccessful = payment.charge(total);
    if (!paymentSuccessful) {
      throw new Error(/* ... */);
    }
    return { total };
  };
}
```

The test adapter — a working fake, no mock library:

```ts
interface FakePaymentAdapter extends PaymentPort { // a test adapter: decides what comes in, records what goes out
  charges: number[];
}

function newFakePaymentAdapter(): FakePaymentAdapter {
  const charges: number[] = [];
  return {
    charges,
    charge(amount) {        // satisfies the port by shape
      charges.push(amount); // records what goes out through the port
      return true;          // decides what comes in
    },
  };
}
```

The test:

```ts
import { expect, test } from "vitest";

function orderWith(prices: number[]): Order { // a harness helper: construction lives in the kit, not the test
  return {
    id: "order_id",
    items: prices.map((price, i) => ({ name: `Item ${i}`, cost: price })),
  };
}

function setup(payment: PaymentPort): OrderProcessor { // Use a helper to wrap the module constructor so future changes to the constructor design can be localised in the test file
  return newOrderProcessor(payment);
}

test.each([
  { prices: [100], expectedCharge: 100 },      // worked examples, chosen by hand
  { prices: [100, 40], expectedCharge: 140 },
])("charges the payment port the order total ($prices)", ({ prices, expectedCharge }) => {
  const payment = newFakePaymentAdapter();           // the test adapter for the port
  const orderProcessor = setup(payment);             // constructed through the constructor, never a factory
  orderProcessor(orderWith(prices));                 // driven through the interface
  expect(payment.charges).toEqual([expectedCharge]); // the outcome goes out through the port — seen by the adapter that receives it
});
```

- The name is the behaviour, and the assertions verify exactly it — a name promising one behaviour while the assertions check another is a test carrying two behaviours.
- Each `test.each` case is one condition of the one behaviour: a second condition is a second case, not a second assertion.
- The expected charges are worked examples (`100 + 40 = 140`, by hand) — the test never computes them with source code.

## Testing the infrastructure

An adapter is tested through the port it satisfies and its own constructor. A Stripe adapter that takes its HTTP dependency in the constructor can be unit tested with a test adapter:

```ts
type Http = (request: HttpRequest) => HttpResponse;

function newStripeAdapter(apiKey: string, http: Http): PaymentPort { // everything required to measure its behaviours
  return {
    charge(amount) { /* ... */ },
  };
}
```

```ts
function newFakeHttp() { // a test adapter for what the adapter's constructor takes
  const requests: HttpRequest[] = [];
  const http: Http = (request) => {
    requests.push(request);
    return { status: 200 };
  };
  return { http, requests };
}

test("charge sends the amount in pence", () => {
  const { http, requests } = newFakeHttp();
  const stripe = newStripeAdapter("test-key", http);
  stripe.charge(12.5);                           // driven through the port it satisfies
  expect(requests[0].body.amount).toBe(1250);
});
```

Integration tests and end-to-end tests are kept apart from the unit tests by directory, so the default run needs no environment:

```ts
// tests/integration/infrastructure/adapters/order-processor/postgres/postgres.test.ts
test("saves and loads an order", () => { // runs against the real resource
  const orders = newPostgresAdapter(process.env.POSTGRES_URL!);
  orders.save(orderWith([100]));
  expect(orders.load("order_id").items[0].cost).toBe(100);
});
```

```ts
// tests/e2e/cli.test.ts
test("cli processes an order", () => { // runs an entrypoint with the production factories
  const output = main(["process", "--price", "100"]);
  expect(output).toContain("total=100");
});
```

```json
// package.json
{
  "scripts": {
    "test": "vitest run tests/unit",
    "test:integration": "vitest run tests/integration",
    "test:e2e": "vitest run tests/e2e"
  }
}
```

## Anti-patterns

**Tautological** — the oracle is the source code:

```ts
import { calculateTotal } from "../../../../src/core/order-processor/implementation"; // a source-code helper

test("totals the receipt", () => { // tautological
  const payment = newFakePaymentAdapter();
  const orderProcessor = setup(payment);
  const order = orderWith([100, 40]);
  expect(orderProcessor(order).total).toBe(calculateTotal(order));
  // whatever bug calculateTotal carries, the expectation inherits —
  // the test agrees by construction and can never disagree
});
```

The fix is an oracle independent of the source code: computing the expectation in the test from the input is fine (`prices.reduce((sum, price) => sum + price, 0)`); the strongest form is section 1's hand-picked worked examples.

**Implementation-coupled** — mocking the implementation's internals:

```ts
import { expect, test, vi } from "vitest";
import { sumCosts } from "../../../../src/core/order-processor/totals"; // a helper the implementation calls internally

vi.mock("../../../../src/core/order-processor/totals");                 // reaches inside the implementation

test("sums the prices", () => {
  const payment = newFakePaymentAdapter();
  const orderProcessor = setup(payment);
  vi.mocked(sumCosts).mockReturnValue(100);
  orderProcessor(orderWith([100, 40]));
  expect(sumCosts).toHaveBeenCalledOnce();
});
```

The tell: swap `sumCosts` for a hand-rolled loop — behaviour unchanged, every interface-level test green — and this test goes red. It was never on the outside of the interface. The fix: observation belongs at the interface and the ports — the fake payment adapter's `charges` — not inside the implementation's helpers. (Module mocking and exporting internals just so a test can reach them are this pattern's two common TypeScript shapes.)
