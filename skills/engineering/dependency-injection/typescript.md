# Dependency Injection in TypeScript

The rules in [SKILL.md](SKILL.md) are language-agnostic. This file is their TypeScript spelling. The vocabulary's TypeScript shapes — interfaces, function types, object literals — live in [codebase-design's typescript.md](../codebase-design/typescript.md).

# The constructor

## Rule 1: a constructor takes the module's port and its configuration

Bad examples for an `OrderProcessor` module:

```ts
import { newStripeAdapter } from "../../infrastructure/adapters/order-processor/stripe";

function newOrderProcessor(
  port: ReturnType<typeof newStripeAdapter>, // The constructor names an adapter's type, which includes its implementation.
): OrderProcessor {
  /* ... */
}
```

```ts
import type { PaymentGateway } from "../payment-gateway";

function newOrderProcessor(
  port: PaymentGateway, // The constructor names another module's interface, so a change to that module reaches this one.
): OrderProcessor {
  /* ... */
}
```

```ts
interface PaymentPort {
  charge(amount: number, currency: Currency): boolean;
}

type ClockPort = () => Date;

function newOrderProcessor(
  payment: PaymentPort,
  clock: ClockPort, // A second port: the module's needs are split across two interfaces.
): OrderProcessor {
  /* ... */
}
```

```ts
interface OrderProcessorPort { // The port is copied from what Stripe offers, not from what the module calls.
  charge(amount: number, currency: Currency): boolean;
  refund(amount: number, currency: Currency): boolean;
  listCharges(): Charge[];
}
```

Good example for an `OrderProcessor` module:

```ts
interface OrderProcessorPort { // The module's one port, declared by the module, holding only what it calls.
  charge(amount: number, currency: Currency): boolean;
  now(): Date;
}

function newOrderProcessor(
  port: OrderProcessorPort, // The port: any adapter satisfying it can be passed.
  currency: Currency,       // Configuration: a value, passed as its concrete type.
): OrderProcessor {
  /* ... */
}
```

A `Refunds` module that also needs Stripe declares its own port. Each module has its own adapter, and the two adapters are thin wrappers around the same resource:

```ts
interface RefundsPort { // The port of the refunds module
  refund(amount: number, currency: Currency): boolean;
}
```

```ts
// infrastructure/adapters/order-processor/stripe
function newStripeAdapter(apiKey: string, http: Http): OrderProcessorPort {
  return {
    charge(amount, currency) { /* ... */ },
    now() { /* ... */ }, // a whole adapter: it satisfies every method of the port
  };
}

// infrastructure/adapters/refunds/stripe
function newStripeAdapter(apiKey: string, http: Http): RefundsPort {
  return {
    refund(amount, currency) { /* ... */ },
  };
}
```

An adapter's constructor can take anything, but like every constructor it must take everything required to measure the adapter's behaviours:

```ts
function newStripeAdapter(apiKey: string): OrderProcessorPort {
  const http = newRealHttp(); // bad: created inside, so a test cannot measure what is sent
  /* ... */
}
```

```ts
function newStripeAdapter(
  apiKey: string,
  http: Http, // good: passed in, so a test can pass its own and measure what is sent
): OrderProcessorPort {
  /* ... */
}
```

## Rule 2: the port and configuration go in the constructor, not the interface

Avoid for an `OrderProcessor` module:

```ts
type OrderProcessor = (order: Order, port: OrderProcessorPort) => Receipt;

function newOrderProcessor(): OrderProcessor {
  return function processOrder(
    order,
    port, // The port that changes the capability of the order processor is in the interface, so every caller must supply it.
  ) {
    /* ... */
  };
}
```

Prefer for an `OrderProcessor` module:

```ts
type OrderProcessor = (order: Order) => Receipt;

function newOrderProcessor(
  port: OrderProcessorPort, // The port is passed to the constructor, so the interface takes only the order.
  currency: Currency,
): OrderProcessor {
  return function processOrder(order) {
    const total = order.items.reduce((sum, item) => sum + item.cost, 0);
    if (!port.charge(total, currency)) {
      throw new PaymentDeclined(order.id);
    }
    return { total, currency };
  };
}
```

## Rule 3: a constructor accepts what it needs, and reaches no resource

Avoid a constructor or implementation that creates or fetches what it needs:

```ts
function newOrderProcessor(): OrderProcessor {
  const apiKey = process.env.STRIPE_API_KEY!; // fetches a resource
  const port = newStripeAdapter(apiKey, newRealHttp()); // creates its own adapter
  const currency = process.env.CURRENCY as Currency;     // fetches its own configuration
  return function processOrder(order) {
    const placedAt = new Date();              // fetches a resource
    /* ... */
  };
}
```

Prefer a constructor that accepts all of it, with the clock reached through the port:

```ts
function newOrderProcessor(
  port: OrderProcessorPort, // the port, which also tells the time
  currency: Currency,       // configuration
): OrderProcessor {
  return function processOrder(order) {
    const placedAt = port.now();
    /* ... */
  };
}
```

An adapter's constructor keeps what it is given. It reaches its resource only when called:

```ts
function newPostgresAdapter(connectionString: string): OrderProcessorPort {
  const pool = new Pool({ connectionString });
  void pool.connect(); // bad: constructing the adapter opens a connection
  /* ... */
}
```

```ts
function newPostgresAdapter(pool: Pool): OrderProcessorPort { // good: the pool was opened by whatever called the factory
  /* ... */
}
```

# The factory

The examples build on the `OrderProcessor` module above and two adapters for its port:

```ts
function newStripeAdapter(apiKey: string, http: Http): OrderProcessorPort { /* ... */ }

function newPaypalAdapter(apiKey: string, http: Http): OrderProcessorPort { /* ... */ }
```

## Rule 4: a factory returns the interface

Bad example:

```ts
function createOrderProcessor(
  provider: "stripe" | "paypal",
  apiKey: string,
  currency: Currency,
) { // This is not a factory for the OrderProcessor module: with no declared return type, callers get whatever type the constructor happens to return, not a promise of the module interface.
  /* ... */
}
```

Good example:

```ts
function createOrderProcessor(
  provider: "stripe" | "paypal",
  apiKey: string,
  currency: Currency,
): OrderProcessor { // This is a factory for the OrderProcessor module since it declares the return type as the interface for the module.
  switch (provider) {
    case "stripe":
      return newOrderProcessor(newStripeAdapter(apiKey, newRealHttp()), currency);
    case "paypal":
      return newOrderProcessor(newPaypalAdapter(apiKey, newRealHttp()), currency);
  }
}
```

## Rule 5: a factory only wires

Bad example:

```ts
function createOrderProcessor(apiKey: string, currency: Currency): OrderProcessor {
  const orderProcessor = newOrderProcessor(newStripeAdapter(apiKey, newRealHttp()), currency);

  return function withRetry(order) { // Retrying is a behaviour, and no test adapter can reach it here.
    for (let attempt = 0; attempt < 3; attempt++) {
      try {
        return orderProcessor(order);
      } catch {
        continue;
      }
    }
    throw new Error(/* ... */);
  };
}
```

Retrying belongs behind a constructor: in the implementation of the module, or in an adapter that wraps another adapter of the same port. The factory then only chooses and constructs:

```ts
function createOrderProcessor(apiKey: string, currency: Currency): OrderProcessor {
  const port = newRetryingAdapter(newStripeAdapter(apiKey, newRealHttp()), 3);
  return newOrderProcessor(port, currency);
}
```

## Rule 6: a factory reaches no resource

Bad example:

```ts
function createOrderProcessor(): OrderProcessor {
  return newOrderProcessor(
    newStripeAdapter(
      process.env.STRIPE_API_KEY!, // The factory reads the environment.
      newRealHttp(),
    ),
    process.env.CURRENCY as Currency,
  );
}
```

Good example, with the values passed in by whatever calls the factory:

```ts
function createOrderProcessor(apiKey: string, currency: Currency): OrderProcessor {
  return newOrderProcessor(
    newStripeAdapter(
      apiKey,
      newRealHttp(), // Constructing reaches no resource: the http function only reaches the network when called.
    ),
    currency,
  );
}
```

```ts
// the program's startup code, an entrypoint or a settings loader: whichever the design chooses
const orderProcessor = createOrderProcessor(
  process.env.STRIPE_API_KEY!,
  process.env.CURRENCY as Currency,
);
```

A connection is opened outside the factory too, and passed in:

```ts
function createOrderProcessor(pool: Pool, currency: Currency): OrderProcessor {
  return newOrderProcessor(newPostgresAdapter(pool), currency);
}
```

## Rule 7: one constructor, many factories

Each factory is one way of supplying the single constructor:

```ts
function createStripeOrderProcessor(apiKey: string, currency: Currency): OrderProcessor {
  return newOrderProcessor(newStripeAdapter(apiKey, newRealHttp()), currency);
}

function createInMemoryOrderProcessor(currency: Currency): OrderProcessor {
  return newOrderProcessor(newInMemoryAdapter(), currency);
}
```

Bad example, a second way of constructing added beside the implementation:

```ts
// core/order-processor/implementation.ts
export function newOrderProcessor(port: OrderProcessorPort, currency: Currency): OrderProcessor { /* ... */ }

export function newStripeOrderProcessor(apiKey: string): OrderProcessor { // A second constructor: the module's package now knows about an adapter.
  return newOrderProcessor(newStripeAdapter(apiKey, newRealHttp()), "GBP");
}
```

## Rule 8: a factory never calls another module's constructor

A `Checkout` module's port is satisfied by an adapter that calls an `OrderProcessor`.

Bad example:

```ts
function createCheckout(apiKey: string): Checkout {
  const orderProcessor = newOrderProcessor(newStripeAdapter(apiKey, newRealHttp()), "GBP"); // Calls another module's constructor, so how an OrderProcessor is built is now known in two places.
  return newCheckout(newOrderProcessorAdapter(orderProcessor));
}
```

Good examples:

```ts
function createCheckout(apiKey: string): Checkout {
  const orderProcessor = createStripeOrderProcessor(apiKey, "GBP"); // Another module's factory
  return newCheckout(newOrderProcessorAdapter(orderProcessor));
}
```

```ts
function createCheckout(orderProcessor: OrderProcessor): Checkout { // The entity is taken as a parameter
  return newCheckout(newOrderProcessorAdapter(orderProcessor));
}
```
