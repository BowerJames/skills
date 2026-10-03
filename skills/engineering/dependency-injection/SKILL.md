---
name: dependency-injection
description: Discipline for wiring modules together. Covers coding against interfaces, declaring seams at construction, pure construction, the injectables/creatables split, and scoped injectables. Use when wiring dependencies, structuring initialisation, designing module graphs, or fixing modules that create or fetch what they need.
---

# Prerequisites

- **[codebase-design](../codebase-design/SKILL.md)** — read it first. This skill uses its vocabulary (module, interface, implementation, seam, adapter, resource) exactly.

# Dependency Injection

Dependency injection is passing a module its dependencies instead of letting it create or fetch them: a module receives **Adapters** across its **Seams** rather than reaching for **Resources** itself. The aim is that every resource sits behind a seam, every seam is declared in the **Interface**, and construction stays pure.

## Glossary

Use these terms exactly. The prerequisite terms (module, interface, implementation, seam, adapter, resource) are defined in [codebase-design](../codebase-design/SKILL.md); these are this skill's own.

**Construction** — the act of assembling a module so it is ready for use: a class constructor, a factory function, or simply entering a function.

**Use** — the phase where the assembled module does its work. Construction wires; use works.

**Injectable** — anything provided at construction to configure behaviour: a plain value (a bool, a timeout) or an **Adapter** at a **Seam**. It decides *how* the module works — which adapter satisfies each seam, which **Resource** is reached, which policy applies. Injectables never vary between calls and carry no per-call state; a module injectable wraps a resource or composes other injectables.

_Avoid_: service (overloaded), singleton (a lifetime, not a role), helper.

**Creatable** — anything passed via parameters to carry data: a plain value (a number, a string), an entity, a message, a DTO. It decides *what* the module works on. Data varies per call, so creatables cross interfaces as parameters and returns, never sitting at seams — created anywhere, cheaply, short-lived. Their behaviour, if any, is pure: answers about themselves, touching no resource.

_Avoid_: model (overloaded), data bag.

**Scope** — the span over which an injectable is shared: the runtime, a request, a session, an entity. Each scope composes its injectables once, at its boundary. The runtime's scope is composed at the **composition root**; narrower scopes compose where their runtime data arrives.

**Composition point** — the only place in its scope that knows the concretes: where that scope's graph is assembled and its seams satisfied. The **composition root** is the widest composition point, near `main`; narrower ones sit at scope boundaries.

## Rules

### 1. Code against interfaces, not implementations

Declare a dependency's type as an **Interface**, not a concrete class. The concrete thing is an **Adapter**; the seam exists only where the interface is declared.

```python
class PaymentGateway(Protocol):  # the interface at the seam
    def charge(self, amount: float) -> bool:
        ...

class StripeGateway(PaymentGateway):  # an adapter satisfying it
    def __init__(self, api_key: str, requester):
        ...

    def charge(self, amount: float) -> bool:  # what processOrder needs
        ...

    def refund(self, amount: float) -> bool:  # part of the adapter's full surface,
        ...                                   # invisible at the seam

# Coupled: no seam — the caller is welded to one adapter
def processOrder(order, gateway: StripeGateway):
    ...

# Decoupled: PaymentGateway is the interface at the seam
# Any adapter — Stripe in production, a fake in tests — can satisfy it
def processOrder(order, gateway: PaymentGateway):
    ...
```

The interface belongs to the consumer: declare the operations the caller needs, which is often a subset of the adapter's full surface. A consumer forced to see `refund` when it only ever charges is seeing someone else's interface.

### 2. Declare seams at construction

The seam must be visible in the module's interface, and construction is where the interface declares it. A dependency in the signature is a fact every caller and test can see and must satisfy; a dependency fetched inside the body is a hidden resource — present in behaviour, absent from the interface.

- **Injection at construction makes the invalid state unrepresentable.** A module cannot be assembled without its adapters.
- **Service locator** (asking a global registry or container for dependencies) removes the seam from the interface entirely: the module works only when the ambient world is arranged just so. Injection inverts this — dependencies arrive; the module never fetches.
- **Setter injection** half-hides it: the module exists in an unready state until some caller remembers the setter. The interface lies about what is required.

This is the testability rule "accept dependencies, don't create them" applied at the level of the signature. Construction is partial application: bind the injectables, and what remains is a stable function from creatables to results.

### 3. Keep construction pure and side-effect free

Construction wires; use works. Construction is pure when the same adapters in produce the same module out and nothing else happens: no I/O, no network, no clock reads, no thread spawning, no world queries. Validating argument shapes is construction; reading a config file is not — config, the filesystem, and the clock are all **Resources**, and construction that touches one has dependencies its interface doesn't declare.

Pure construction buys three things:

- The whole graph can be assembled in a test, instantly, with the world absent.
- Construction is deterministic: if wiring fails, it fails at the composition root, not deep inside a call.
- Effects live in methods, where the interface says they happen.

### 4. Separate injectables from creatables

Everything a module receives is an injectable or a creatable, and the two never swap roles: injectables are provided at construction to configure behaviour; creatables are passed via parameters to carry data.

Rules of the split:

- Module injectables may depend on injectables. Creatables may hold only data, and other creatables.
- Never inject a creatable. It's data: create it, or receive it as a method argument.
- A parameter that never varies between calls is an injectable that missed construction: promote it.
- A dependency that varies per call is a creatable at this scope: demote it to a parameter, or open a narrower scope and bind it there.
- Never create an injectable inside a working method. Creation at a scope's boundary is composition: the boundary declares the runtime data it needs, wires the scope's graph from that data plus wider-scoped adapters, and then may use what it composed. Creation inside a working method hides the resource behind an ordinary call.
- Respect scope direction: an injectable may depend only on injectables of the same or wider scope. A wider module holding a narrower one is a captive — it outlives its data, and every later request sees the first entity's config.

When they mix, both rot: a creatable that takes injectables becomes a service-in-disguise, dragging its dependencies through every construction site; an injectable holding mutable per-call state becomes untestable. The test question: does it decide how the module behaves (an injectable — provide it at construction), or what the module works on (a creatable — pass it as a parameter)?

## Composition points

The composition root, near `main`, assembles the runtime's graph: every concrete adapter is created there, and every seam is satisfied there. It is the only code that knows all the concretes; everything downstream sees interfaces.

```python
def main():
    http = HttpRequests()                      # adapter for the network resource
    clock = SystemClock()                      # adapter for the clock resource
    gateway = StripeGateway(API_KEY, http)     # injectable
    orders = OrderService(gateway, clock)      # injectable composing injectables
    App(orders).run()

class OrderService:                            # an injectable: construction only wires
    def __init__(self, gateway: PaymentGateway, clock: Clock):
        self.gateway = gateway
        self.clock = clock

    def place(self, cart) -> Order:            # work happens in methods
        order = Order(cart, self.clock.now())  # Order is a creatable, created here
        return order if self.gateway.charge(order.total) else ...
```

A narrower scope composes at its boundary, where runtime data arrives:

```python
class EntityEndpoints:                                # root-scoped
    def __init__(self, config_source: ConfigSource):  # adapter, chosen at the root
        self.config_source = config_source

    def on_get(self, entity_id: EntityId):            # scope boundary: the creatable arrives
        loader = EntityConfigLoader(entity_id, self.config_source)  # scoped injectable
        return EntityService(loader).handle()         # compose the scope, then use it
```

Composition needs two ingredients: dependencies (adapters, chosen at the root) and data (creatables, arriving at runtime). The root has only dependencies; when construction needs runtime data, composition moves to the scope where that data arrives — but the root still chooses the adapters. What crosses a scope boundary is either an already-wired adapter or a factory that binds the runtime data to it.

One composition point per scope. A DI framework or container is optional: it only automates these functions. The discipline lives in the modules, not the framework.

## Relationships

- A dependency declared in a signature is an **Interface** at a **Seam**; the thing passed in is an **Adapter**.
- A module injectable wraps a **Resource** or composes injectables; a creatable is data that crosses interfaces.
- A composition point is a module whose **Implementation** is pure wiring; the composition root is the widest.
- A scope's composition binds two ingredients: adapters chosen at the root, and creatables arriving at the boundary.
- Pure construction keeps every resource behind a seam, wiring included.

## Rejected framings

- **"DI needs a framework."** DI is two disciplines — declare seams, pass adapters — doable in any language with function parameters. Frameworks automate the composition root; they don't carry the discipline.
- **Injecting everything.** Only injectables are injected. Creatables are created. Treating every object as a dependency swells the graph and turns data into wiring.
- **Service locator as DI.** A locator inverts the inversion: the module fetches again, the seam vanishes from the interface, and every test must arrange the ambient registry first.
