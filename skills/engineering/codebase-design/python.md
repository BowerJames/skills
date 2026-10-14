# Codebase Design in Python

The vocabulary in [SKILL.md](SKILL.md) is language-agnostic. This file is its Python spelling.

## Vocabulary → Python

| Term | Python |
|---|---|
| **Interface** | for classes, a `typing.Protocol`; for functions, a type — `Callable[[], datetime]` |
| **Port** | for classes, a `typing.Protocol`; for functions, a type — `Callable[[], datetime]` |
| **Adapter** | a class or function satisfying the port's shape |
| **Constructor** | when the implementation is a class, its `__init__`; when the implementation is a function, the function that takes the ports and returns it |

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

class PaymentPort(Protocol): # Port defined for the module
    def charge(self, amount: float) -> bool: ...

class OrderProcessorImpl: # Implementation of the module
    def __init__(
        self,
        payment: PaymentPort
    ): # Constructor for the module
        self.payment = payment

    def __call__(self, order: Order) -> Receipt:
        ...

class StripeAdapter(PaymentPort): # Adapter for the payment port
    def __init__(self, api_key: str, http: HTTP): ... # Constructor for the stripe adapter

    def charge(self, amount: float) -> bool: ...

class PaypalAdapter(PaymentPort): # Adapter for the payment port
    def __init__(self, api_key: str, http: HTTP): ... # Constructor for the paypal adapter

    def charge(self, amount: float) -> bool: ...

def create_order_processor(
        type: Literal["stripe", "paypal"]
    ) -> OrderProcessor: # A factory for the order processor module
    if type == "stripe":
        payment = StripeAdapter(...)
    elif type == "paypal":
        payment = PaypalAdapter(...)
    return OrderProcessorImpl(payment)
```

Matching is structural: any class with a matching `charge` satisfies `PaymentPort`. An adapter inherits from the port it satisfies anyway (`class StripeAdapter(PaymentPort)`). The inheritance is a reminder, not a requirement: it links the adapter to the port it is intended for, so a developer reading the adapter can see the dependency.
