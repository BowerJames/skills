# Factory Pattern in TypeScript

The rules in [SKILL.md](SKILL.md) are language-agnostic. This file is their TypeScript spelling. The vocabulary's TypeScript shapes — interfaces, function types, object literals — live in [codebase-design's typescript.md](../codebase-design/typescript.md).

The examples build on this `OrderProcessor` module and its adapters:

```ts
interface PaymentPort { // Port defined for the module
  charge(amount: number): boolean;
}

type OrderProcessor = (order: Order) => Receipt; // Interface for the module

function newOrderProcessor(payment: PaymentPort, currency = "GBP"): OrderProcessor { // Constructor for the module
  return function processOrder(order) { // Implementation of the module
    /* ... */
  };
}

function newStripeAdapter(apiKey: string, http: Http): PaymentPort { // Constructor for the stripe adapter
  return {
    charge(amount) { /* ... */ },
  };
}

function newPaypalAdapter(apiKey: string, http: Http): PaymentPort { // Constructor for the paypal adapter
  return {
    charge(amount) { /* ... */ },
  };
}
```

## Rule 1: a factory returns the interface

Bad example:

```ts
function createOrderProcessor(
  type: "stripe" | "paypal",
) { // This is not a factory for the OrderProcessor module: with no declared return type, callers get whatever type the constructor happens to return, not a promise of the module interface.
  switch (type) {
    case "stripe":
      return newOrderProcessor(newStripeAdapter(/* ... */));
    case "paypal":
      return newOrderProcessor(newPaypalAdapter(/* ... */));
  }
}
```

Good example:

```ts
function createOrderProcessor(
  type: "stripe" | "paypal",
): OrderProcessor { // This is now a factory for the OrderProcessor module since it declares the return type as the interface for the module.
  switch (type) {
    case "stripe":
      return newOrderProcessor(newStripeAdapter(/* ... */));
    case "paypal":
      return newOrderProcessor(newPaypalAdapter(/* ... */));
  }
}
```

## Rule 2: a factory holds no behaviour

Bad example:

```ts
function createOrderProcessor(type: "stripe" | "paypal"): OrderProcessor {
  const payment = type === "stripe" ? newStripeAdapter(/* ... */) : newPaypalAdapter(/* ... */);
  const orderProcessor = newOrderProcessor(payment);

  return function withRetry(order) { // Retrying is a behaviour, and no test adapter can reach it here.
    for (let attempt = 0; attempt < 3; attempt++) {
      try {
        return orderProcessor(order);
      } catch {
        continue;
      }
    }
    throw new Error("payment failed");
  };
}
```

Retrying belongs behind a constructor: in the implementation of the module, or in an adapter that wraps another adapter. The factory then only chooses and constructs:

```ts
function createOrderProcessor(type: "stripe" | "paypal"): OrderProcessor {
  const payment = type === "stripe" ? newStripeAdapter(/* ... */) : newPaypalAdapter(/* ... */);
  return newOrderProcessor(newRetryingAdapter(payment, 3));
}
```

## Rule 3: one constructor, many factories

Each factory is one way of supplying the single constructor:

```ts
function createStripeOrderProcessor(): OrderProcessor {
  return newOrderProcessor(newStripeAdapter(/* ... */));
}

function createPaypalOrderProcessor(): OrderProcessor {
  return newOrderProcessor(newPaypalAdapter(/* ... */));
}
```

Bad example, a second way of constructing added beside the implementation:

```ts
// core/order-processor/implementation.ts
export function newOrderProcessor(payment: PaymentPort): OrderProcessor { /* ... */ }

export function newStripeOrderProcessor(): OrderProcessor { // A second constructor: the module's package now knows about an adapter.
  return newOrderProcessor(newStripeAdapter(/* ... */));
}
```

## Rule 4: a factory never calls another module's constructor

A `Checkout` module has a port satisfied by an adapter that calls an `OrderProcessor`.

Bad example:

```ts
function createCheckout(): Checkout {
  const orderProcessor = newOrderProcessor(newStripeAdapter(/* ... */)); // Calls another module's constructor, so how an OrderProcessor is built is now known in two places.
  return newCheckout(newOrderProcessorAdapter(orderProcessor));
}
```

Good examples:

```ts
function createCheckout(): Checkout {
  const orderProcessor = createOrderProcessor("stripe"); // Another module's factory
  return newCheckout(newOrderProcessorAdapter(orderProcessor));
}
```

```ts
function createCheckout(orderProcessor: OrderProcessor): Checkout { // The entity is taken as a parameter
  return newCheckout(newOrderProcessorAdapter(orderProcessor));
}
```

## Rule 5: a factory is where resources are first reached

The factory reads the environment and passes what it reads to the constructors:

```ts
function createOrderProcessor(
  type: "stripe" | "paypal", // A parameter that selects between choices
): OrderProcessor {
  const currency = process.env.CURRENCY!; // Configuration for the module, read here and not in the constructor
  switch (type) {
    case "stripe":
      return newOrderProcessor(
        newStripeAdapter(
          process.env.STRIPE_API_KEY!, // The factory reaches the resource and passes the value on
          stripe.newRealHttp(),        // The factory constructs what the adapter's own port needs, from the adapter's package
        ),
        currency,
      );
    case "paypal":
      return newOrderProcessor(
        newPaypalAdapter(process.env.PAYPAL_API_KEY!, paypal.newRealHttp()),
        currency,
      );
  }
}
```

What a factory takes depends on its use case. This one takes the adapter and supplies only the configuration:

```ts
function createOrderProcessor(
  payment: PaymentPort, // An adapter chosen by the caller
): OrderProcessor {
  return newOrderProcessor(payment, process.env.CURRENCY!);
}
```
