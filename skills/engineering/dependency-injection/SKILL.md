---
name: dependency-injection
description: Discipline for wiring modules together. Covers coding against interfaces, constructor injection, pure constructors, and the injectables/creatables split. Use when wiring dependencies, structuring constructors, designing module graphs, or fixing modules that create or fetch what they need.
---

# Dependency Injection

Dependency injection is passing a module its dependencies instead of letting it create or fetch them: a module receives **Adapters** across its **Seams** rather than reaching for **Resources** itself. The aim is that every resource sits behind a seam, every seam is declared in the **Interface**, and construction stays pure.

## Prerequisites

- **[codebase-design](../codebase-design/SKILL.md)** — read it first. This skill uses its vocabulary (module, interface, implementation, seam, adapter, resource) exactly.

## Rules

### 1. Code against interfaces, not implementations

Declare a dependency's type as an **Interface**, not a concrete class. The concrete thing is an **Adapter**; the seam exists only where the interface is declared.

```python
# Coupled: no seam — the caller is welded to one adapter
def processOrder(order, gateway: StripeGateway):
    ...

# Decoupled: PaymentGateway is the interface at the seam
# Any adapter — Stripe in production, a fake in tests — can satisfy it
def processOrder(order, gateway: PaymentGateway):
    ...
```

The interface belongs to the consumer: declare the operations the caller needs, which is often a subset of the adapter's full surface. A consumer forced to see `refund` when it only ever charges is seeing someone else's interface.

### 2. Prefer constructor injection

The seam must be visible in the module's interface, and the constructor (or function signature) is where the interface declares it. A dependency in the signature is a fact every caller and test can see and must satisfy; a dependency fetched inside the body is a hidden resource — present in behaviour, absent from the interface.

- **Constructor injection makes the invalid state unrepresentable.** A module cannot exist without its adapters.
- **Service locator** (asking a global registry or container for dependencies) removes the seam from the interface entirely: the module works only when the ambient world is arranged just so. Injection inverts this — dependencies arrive; the module never fetches.
- **Setter injection** half-hides it: the module exists in an unready state until some caller remembers the setter. The interface lies about what is required.

This is the testability rule "accept dependencies, don't create them" applied at the level of the signature.

### 3. Keep constructors pure and side-effect free

A constructor wires; a method works. Construction is pure when the same adapters in produce the same object out and nothing else happens: no I/O, no network, no clock reads, no thread spawning, no world queries. Validating argument shapes is construction; reading a config file is not — config, the filesystem, and the clock are all **Resources**, and a constructor that touches one has dependencies its interface doesn't declare.

Pure construction buys three things:

- The whole graph can be assembled in a test, instantly, with the world absent.
- Construction is deterministic: if wiring fails, it fails at the composition root, not deep inside a call.
- Effects live in methods, where the interface says they happen.

### 4. Separate injectables from creatables

**Injectable** — a module whose job is behaviour: it wraps a **Resource** or composes other injectables, sits at a **Seam**, is created once at the **Composition Root**, and lives for the runtime. It carries no per-call state.

_Avoid_: service (overloaded), singleton (a lifetime, not a role), helper.

**Creatable** — a module whose job is data: a value, entity, message, or DTO. Created anywhere it's needed, cheaply and often, short-lived, holding state and identity. Creatables cross interfaces as parameters and returns; they never sit at seams.

_Avoid_: model (overloaded), data bag (a creatable may have behaviour over its own data — that's allowed).

Rules of the split:

- Injectables may depend on injectables. Creatables may hold only data, and other creatables.
- Never inject a creatable. It's data: create it, or receive it as a method argument.
- Never create an injectable mid-flow. If it touches a resource, it's wired at the composition root; a mid-flow `new` hides the resource behind a constructor call.

When they mix, both rot: a creatable that takes injectables becomes a service-in-disguise, dragging its dependencies through every construction site; an injectable holding mutable per-call state becomes untestable. The test question: does it exist to hold data, or to do work over resources?

## Composition root

The **composition root** is the single module, near `main`, where the object graph is assembled: every concrete adapter is constructed here, and every seam is satisfied here. It is the only code that knows all the concretes; everything downstream sees interfaces.

```python
def main():
    http = HttpRequests()                      # adapter for the network resource
    clock = SystemClock()                      # adapter for the clock resource
    gateway = StripeGateway(API_KEY, http)     # injectable
    orders = OrderService(gateway, clock)      # injectable composing injectables
    App(orders).run()

class OrderService:                            # an injectable: constructor only wires
    def __init__(self, gateway: PaymentGateway, clock: Clock):
        self.gateway = gateway
        self.clock = clock

    def place(self, cart) -> Order:            # work happens in methods
        order = Order(cart, self.clock.now())  # Order is a creatable, created here
        return order if self.gateway.charge(order.total) else ...
```

One composition root per runtime. A DI framework or container is optional: it only automates this function. The discipline lives in the modules, not the framework.

## Relationships

- A dependency declared in a signature is an **Interface** at a **Seam**; the thing passed in is an **Adapter**.
- An injectable wraps a **Resource** or composes injectables; a creatable is data that crosses interfaces.
- The composition root is the one module whose **Implementation** is pure wiring.
- Constructor purity keeps every resource behind a seam, including during construction.

## Rejected framings

- **"DI needs a framework."** DI is two disciplines — declare seams, pass adapters — doable in any language with function parameters. Frameworks automate the composition root; they don't carry the discipline.
- **Injecting everything.** Only injectables are injected. Creatables are created. Treating every object as a dependency swells the graph and turns data into wiring.
- **Service locator as DI.** A locator inverts the inversion: the module fetches again, the seam vanishes from the interface, and every test must arrange the ambient registry first.
