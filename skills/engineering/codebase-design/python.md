# Codebase Design in Python

The vocabulary in [SKILL.md](SKILL.md) is language-agnostic. This file is its Python spelling.

## Vocabulary → Python

| Term | Python |
|---|---|
| **Interface** | for classes, a `typing.Protocol`; for functions, a type — `Callable[[Order], Receipt]` |
| **Port** | for classes, a `typing.Protocol`; for functions, a type — `Callable[[], datetime]` |
| **Adapter** | a class or function satisfying the port's shape |
| **Domain interface** | a `typing.Protocol` in the domain package — `AgentTool` |
| **Driven adapter** | an adapter satisfying a module's port — `StripeAdapter` for `OrderProcessorPort` |
| **Driving adapter** | an adapter satisfying the shape its entrypoint calls — a FastAPI `APIRouter`, a command `Callable[[list[str]], None]` |
| **Entrypoint** | the framework or runtime that calls the program, never written in it — FastAPI, Click, Celery, or Python running a file as `__main__` |
| **View** | a `typing.Protocol` in an edge adapter's package, describing the part of a library's object it uses — needed only when the library has no test hook (`httpx` has one: `httpx.MockTransport`) |
| **Constructor** | when the implementation is a class, its `__init__`; when the implementation is a function, the function that takes the port and returns it |
| **Data carrier** | a frozen `dataclass`, or a type alias such as a `Literal` |
| **Error** | an `Exception` subclass |

A constructor and a factory are told apart by name. A class's constructor is the class itself (`OrderProcessorImpl`). A function constructor is named for what it returns (`call_router`). A factory is `create_…` (`create_order_processor`).

## Module

Below illustrates an example of an `OrderProcessor` module in Python.

The interface of a module can be spelled as a `typing.Protocol` or as a type. Both of the below define the same interface; pick one.

```python
from typing import Protocol

class OrderProcessor(Protocol): # Interface for the module, as a Protocol
    def __call__(self, order: Order) -> Receipt: ...
```

```python
from typing import Callable

type OrderProcessor = Callable[[Order], Receipt] # Interface for the module, as a type
```

The rest of the module is the same whichever spelling is used.

```python
from typing import Literal, Protocol

type Currency = Literal["GBP", "USD", "EUR"] # Data carrier: its only invariant is the one its type expresses

class OrderProcessorPort(Protocol): # The module's one port: everything it needs from outside itself
    def charge(self, amount: int, currency: Currency) -> bool: ...

class PaymentDeclined(Exception): # An error the module's implementation determines, so the module owns it
    ...

class OrderProcessorImpl: # Implementation of the module
    def __init__(
        self,
        port: OrderProcessorPort, # the port
        currency: Currency # configuration
    ): # Constructor for the module
        self.port = port
        self.currency = currency

    def __call__(self, order: Order) -> Receipt:
        total = sum(item.cost for item in order.items)
        if not self.port.charge(total, self.currency):
            raise PaymentDeclined(order.id)
        return Receipt(total=total, currency=self.currency)

class StripeAdapter(OrderProcessorPort): # Adapter for the port
    def __init__(self, api_key: str, http: httpx.Client): ... # Constructor for the stripe adapter

    def charge(self, amount: int, currency: Currency) -> bool: ...

class PaypalAdapter(OrderProcessorPort): # Adapter for the port
    def __init__(self, api_key: str, http: httpx.Client): ... # Constructor for the paypal adapter

    def charge(self, amount: int, currency: Currency) -> bool: ...

def create_stripe_adapter(api_key: str) -> OrderProcessorPort: # An adapter factory for the stripe adapter
    return StripeAdapter(api_key, httpx.Client())

def create_paypal_adapter(api_key: str) -> OrderProcessorPort: # An adapter factory for the paypal adapter
    return PaypalAdapter(api_key, httpx.Client())

def create_order_processor(
        provider: Literal["stripe", "paypal"],
        api_key: str,
        currency: Currency
    ) -> OrderProcessor: # A module factory for the order processor module
    if provider == "stripe":
        port = create_stripe_adapter(api_key)
    elif provider == "paypal":
        port = create_paypal_adapter(api_key)
    return OrderProcessorImpl(port, currency)
```

Matching is structural: any class with a matching `charge` satisfies `OrderProcessorPort`. An adapter inherits from the port it satisfies anyway (`class StripeAdapter(OrderProcessorPort)`). The inheritance is a reminder, not a requirement: it links the adapter to the port it is intended for, so a developer reading the adapter can see the dependency.

## Errors

The adapter handles what it can decide and lets everything else propagate:

```python
class StripeAdapter(OrderProcessorPort):
    def __init__(self, api_key: str, http: httpx.Client):
        self.api_key = api_key
        self.http = http

    def charge(self, amount: int, currency: Currency) -> bool:
        response = self.http.post(...) # a network error propagates unchanged
        if response.status_code == 402:
            return False # handled: a decline is an answer the port can give
        return True
```

Wrapping hides the error's origin:

```python
    def charge(self, amount: int, currency: Currency) -> bool:
        try:
            response = self.http.post(...)
        except httpx.ConnectError as error:
            raise PaymentFailed() from error # bad: wraps an error the adapter did not determine
```
