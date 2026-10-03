# Dependency Injection in Python

The rules in [SKILL.md](SKILL.md) are language-agnostic. This file is their Python spelling.

## Vocabulary → Python

| Term | Python |
|---|---|
| **Interface** | for classes, a `typing.Protocol`; for functions, a type — `Callable[[], datetime]` |
| **Adapter** | any class or function satisfying the interface's shape |
| **Module** | a class, a module-level function, or a package |
| **Seam** | a typed parameter behaviour varies through: an object interface, a function interface, or a plain value |
| **Resource** | `os.environ`, the filesystem, the clock, the network — reached only behind an adapter |

## Interfaces are types

```python
from typing import Protocol

class PaymentGateway(Protocol):          # the interface at the seam
    def charge(self, amount: float) -> bool: ...

class StripeGateway:                      # an adapter — no inheritance
    def charge(self, amount: float) -> bool: ...
```

Matching is structural: `StripeGateway` satisfies the Protocol by shape alone. Explicit inheritance (`class StripeGateway(PaymentGateway)`) is allowed and documents intent — it lets the type checker flag drift — but it is a declaration, never a requirement.

- Prefer **Protocol over ABC** for seams: ABC is nominal (adapters must inherit), and a seam wants *any* satisfier.
- `@runtime_checkable` only checks that methods exist, not their signatures — it is no substitute for the type checker.

A function's interface is its type: `clock: Callable[[], datetime]` says everything a caller must know. When a function interface needs keyword parameters, defaults, or a name, promote it to a Protocol with `__call__`:

```python
class Clock(Protocol):
    def __call__(self, *, tz: str = "UTC") -> datetime: ...
```

## The shapes of a seam

A seam is anywhere behaviour varies without editing the module. In Python it takes three shapes:

```python
def __init__(self,
             gateway: PaymentGateway,            # object interface: a whole adapter can cross it
             clock: Callable[[], datetime],      # function interface: one function swaps
             verbose: bool):                     # plain value: one bit of behaviour
```

All three are seams; they differ in **Depth**. A Protocol carries a whole interface of variation, a Callable one function's worth, a bool one bit. Fit the seam to the variation: a Protocol behind a binary toggle is ceremony; a bool behind a swappable adapter is a wall.

## Modules don't need classes

A module-level function with a typed parameter is already seam-declaring DI — no class, no framework, no decorators:

```python
def processOrder(order: Order, gateway: PaymentGateway) -> Receipt: ...
```

## Wiring happens in `main()`

Wire the graph in `main()` (or `if __name__ == "__main__":`), invoked from the entry point. Frameworks: the startup hook is where wiring belongs.

- **Import purity**: importing a module wires nothing. No `gateway = StripeGateway(KEY)` at module top level — that is construction at import time: hidden, order-dependent, untestable.
- **`os.environ` is a Resource**: read it in `main()`, pass the values in as plain typed parameters.

## Data carriers are frozen dataclasses

Data crossing interfaces — values, entities, messages — are frozen dataclasses: created anywhere, cheap, immutable, never taking dependencies.

```python
@dataclass(frozen=True)
class Order:
    cart: Cart
    placed_at: datetime
```

## Testing: fixtures wire the graph

A pytest fixture that assembles the graph with fakes is the test doing its own wiring — the same discipline at test scope. A fake is a small class satisfying the Protocol; a seam needs no mock library. `monkeypatch` is a service locator for tests: injecting a fake at the seam beats patching a global every time.

## Gotchas

- **Mutable default arguments** — `def __init__(self, hooks=[])` shares one list across every instance. Defaults must be immutable; use `None` and construct inside.
- **No framework needed.** `dependency-injector` and friends automate wiring; the discipline is stdlib-only — typed parameters and Protocols carry it.
- **Protocol parameter names must match** for structural compatibility — renaming `charge(self, amount)` to `charge(self, amt)` silently breaks the match.
