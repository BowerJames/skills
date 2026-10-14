# Dependency Injection in TypeScript

The rules in [SKILL.md](SKILL.md) are language-agnostic. This file is their TypeScript spelling. The vocabulary's TypeScript shapes — interfaces, function types, object literals — live in [codebase-design's typescript.md](../codebase-design/typescript.md).

## Rule 1: a constructor depends on the module's own ports

Bad examples for an `OrderProcessor` module:

```ts
import { newStripeAdapter } from "../../infrastructure/adapters/order-processor/stripe";

function newOrderProcessor(
  payment: ReturnType<typeof newStripeAdapter>, // The constructor names an adapter's type, which includes its implementation.
): OrderProcessor {
  /* ... */
}
```

```ts
import type { PaymentGateway } from "../payment-gateway";

function newOrderProcessor(
  payment: PaymentGateway, // The constructor names another module's interface, so a change to that module reaches this one.
): OrderProcessor {
  /* ... */
}
```

```ts
interface PaymentPort { // The port is copied from what Stripe offers, not from what the module calls.
  charge(amount: number): boolean;
  refund(amount: number): boolean;
  listCharges(): Charge[];
}
```

Good example for an `OrderProcessor` module:

```ts
interface PaymentPort { // The port is declared by the module and holds only what the module calls.
  charge(amount: number): boolean;
}

function newOrderProcessor(
  payment: PaymentPort, // A port: any adapter satisfying it can be passed.
  currency: string,     // Configuration: a value, passed as its concrete type.
): OrderProcessor {
  /* ... */
}
```

A `Refunds` module that also needs Stripe declares its own port. Each module has its own adapter, and the two adapters are thin wrappers around the same logic:

```ts
interface RefundPort { // Port defined for the refunds module
  refund(amount: number): boolean;
}
```

```ts
// infrastructure/adapters/order-processor/stripe
function newStripeAdapter(apiKey: string, http: Http): PaymentPort {
  return {
    charge(amount) { /* ... */ },
  };
}

// infrastructure/adapters/refunds/stripe
function newStripeAdapter(apiKey: string, http: Http): RefundPort {
  return {
    refund(amount) { /* ... */ },
  };
}
```

An adapter's constructor can take anything, but like every constructor it must take everything required to measure the adapter's behaviours:

```ts
function newStripeAdapter(apiKey: string): PaymentPort {
  const http = newRealHttp(); // bad: created inside, so a test cannot measure what is sent
  /* ... */
}
```

```ts
function newStripeAdapter(
  apiKey: string,
  http: Http, // good: passed in, so a test can pass its own and measure what is sent
): PaymentPort {
  /* ... */
}
```

## Rule 2: ports and configuration go in the constructor, not the interface

Avoid for an `OrderProcessor` module:

```ts
interface PaymentPort {
  charge(amount: number): boolean;
}

type OrderProcessor = (order: Order, payment: PaymentPort) => Receipt;

function newOrderProcessor(): OrderProcessor {
  return function processOrder(
    order,
    payment, // The port that changes the capability of the order processor is in the interface, so every caller must supply it.
  ) {
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

Prefer for an `OrderProcessor` module:

```ts
interface PaymentPort {
  charge(amount: number): boolean;
}

type OrderProcessor = (order: Order) => Receipt;

function newOrderProcessor(
  payment: PaymentPort, // The port is passed to the constructor, so the interface takes only the order.
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

Avoid a constructor or implementation that creates or fetches what it needs:

```ts
function newOrderProcessor(): OrderProcessor {
  const apiKey = process.env.STRIPE_API_KEY!; // fetches a resource
  const payment = newStripeAdapter(apiKey, newRealHttp()); // creates its own adapter
  const currency = process.env.CURRENCY!;     // fetches its own configuration
  return function processOrder(order) {
    const placedAt = new Date();              // fetches a resource
    /* ... */
  };
}
```

Prefer a constructor that accepts all of it:

```ts
type ClockPort = () => Date;

function newOrderProcessor(
  payment: PaymentPort, // a port
  clock: ClockPort,     // a port, spelled as a type
  currency: string,     // configuration
): OrderProcessor {
  return function processOrder(order) {
    const placedAt = clock();
    /* ... */
  };
}
```
