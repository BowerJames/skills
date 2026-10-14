# Codebase Design in TypeScript

The vocabulary in [SKILL.md](SKILL.md) is language-agnostic. This file is its TypeScript spelling.

The style throughout: **no classes**. Entities are functions and object literals returned by their constructors; they satisfy an `interface` structurally.

## Vocabulary → TypeScript

| Term | TypeScript |
|---|---|
| **Interface** | for object shapes, an `interface`; for functions, a type — `() => Date` |
| **Port** | for object shapes, an `interface`; for functions, a type — `() => Date` |
| **Adapter** | an object literal or function satisfying the port's shape |
| **Constructor** | the function that takes the ports and returns the entity |

With no classes, a constructor and a factory are both functions. They are told apart by name: a constructor is `new…` (`newOrderProcessor`), a factory is `create…` (`createOrderProcessor`).

## Module

Below illustrates an example of an `OrderProcessor` module in TypeScript.

The interface of a module can be spelled as an `interface` or as a type. Both of the below define the same interface; pick one.

```ts
interface OrderProcessor { // Interface for the module, as an interface
  (order: Order): Receipt;
}
```

```ts
type OrderProcessor = (order: Order) => Receipt; // Interface for the module, as a type
```

The rest of the module is the same whichever spelling is used.

```ts
interface PaymentPort { // Port defined for the module
  charge(amount: number): boolean;
}

function newOrderProcessor(
  payment: PaymentPort,
): OrderProcessor { // Constructor for the module
  return function processOrder(order) { // Implementation of the module
    /* ... */
  };
}

function newStripeAdapter(apiKey: string, http: Http): PaymentPort { // Constructor for the stripe adapter
  return { // Adapter for the payment port
    charge(amount) { /* ... */ },
  };
}

function newPaypalAdapter(apiKey: string, http: Http): PaymentPort { // Constructor for the paypal adapter
  return { // Adapter for the payment port
    charge(amount) { /* ... */ },
  };
}

function createOrderProcessor(
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

Matching is structural: the object literals returned by `newStripeAdapter` and `newPaypalAdapter` both satisfy `PaymentPort`. The declared return type checks each literal where it is built, so drift is a compile error at that point. No `class` and no `implements` are needed.
