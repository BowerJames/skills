# Dependency Injection in Python

The rules in [SKILL.md](SKILL.md) are language-agnostic. This file is their Python spelling. The vocabulary's Python shapes — Protocol and Callable mechanics — live in [codebase-design's python.md](../codebase-design/python.md).

## Rule 1: a constructor depends on the module's own ports

Bad examples for an `OrderProcessor` module:

```python
from infrastructure.adapters.order_processor.stripe import StripeAdapter

class OrderProcessorImpl:
    def __init__(
        self,
        payment: StripeAdapter # The constructor names an adapter's type, which includes its implementation.
    ): ...

    def __call__(self, order: Order) -> Receipt: ...
```

```python
from core.payment_gateway import PaymentGateway

class OrderProcessorImpl:
    def __init__(
        self,
        payment: PaymentGateway # The constructor names another module's interface, so a change to that module reaches this one.
    ): ...

    def __call__(self, order: Order) -> Receipt: ...
```

```python
class PaymentPort(Protocol): # The port is copied from what Stripe offers, not from what the module calls.
    def charge(self, amount: float) -> bool: ...
    def refund(self, amount: float) -> bool: ...
    def list_charges(self) -> list[Charge]: ...
```

Good example for an `OrderProcessor` module:

```python
class PaymentPort(Protocol): # The port is declared by the module and holds only what the module calls.
    def charge(self, amount: float) -> bool: ...

class OrderProcessorImpl:
    def __init__(
        self,
        payment: PaymentPort, # A port: any adapter satisfying it can be passed.
        currency: str # Configuration: a value, passed as its concrete type.
    ): ...

    def __call__(self, order: Order) -> Receipt: ...
```

A `Refunds` module that also needs Stripe declares its own port. Each module has its own adapter, and the two adapters are thin wrappers around the same logic:

```python
class RefundPort(Protocol): # Port defined for the refunds module
    def refund(self, amount: float) -> bool: ...
```

```python
class StripeAdapter(PaymentPort): # infrastructure/adapters/order_processor/stripe
    def __init__(self, api_key: str, http: HTTP): ...

    def charge(self, amount: float) -> bool: ...

class StripeAdapter(RefundPort): # infrastructure/adapters/refunds/stripe
    def __init__(self, api_key: str, http: HTTP): ...

    def refund(self, amount: float) -> bool: ...
```

An adapter's constructor can take anything, but like every constructor it must take everything required to measure the adapter's behaviours:

```python
class StripeAdapter(PaymentPort):
    def __init__(self, api_key: str):
        self.http = RealHTTP() # bad: created inside, so a test cannot measure what is sent
```

```python
class StripeAdapter(PaymentPort):
    def __init__(
        self,
        api_key: str,
        http: HTTP # good: passed in, so a test can pass its own and measure what is sent
    ): ...
```

## Rule 2: ports and configuration go in the constructor, not the interface

Avoid for an `OrderProcessor` module:

```python
class PaymentPort(Protocol):
    def charge(self, amount: float) -> bool: ...

type OrderProcessor = Callable[[Order, PaymentPort], Receipt]

class OrderProcessorImpl:
    def __call__(
        self,
        order: Order,
        payment: PaymentPort # The port that changes the capability of the order processor is in the interface, so every caller must supply it.
    ) -> Receipt:
        total = 0
        for item in order.items:
            total += item.cost
        payment_successful = payment.charge(total)
        if not payment_successful:
            raise ...
        return Receipt(
            total=total
        )
```

Prefer for an `OrderProcessor` module:

```python
class PaymentPort(Protocol):
    def charge(self, amount: float) -> bool: ...

type OrderProcessor = Callable[[Order], Receipt]

class OrderProcessorImpl:
    def __init__(
        self,
        payment: PaymentPort # The port is passed to the constructor, so the interface takes only the order.
    ):
        self.payment = payment

    def __call__(self, order: Order) -> Receipt:
        total = 0
        for item in order.items:
            total += item.cost
        payment_successful = self.payment.charge(total)
        if not payment_successful:
            raise ...
        return Receipt(
            total=total
        )
```

Avoid a constructor or implementation that creates or fetches what it needs:

```python
class OrderProcessorImpl:
    def __init__(self):
        api_key = os.environ["STRIPE_API_KEY"]   # fetches a resource
        self.payment = StripeAdapter(api_key, RealHTTP())  # creates its own adapter
        self.currency = os.environ["CURRENCY"]   # fetches its own configuration

    def __call__(self, order: Order) -> Receipt:
        placed_at = datetime.now()               # fetches a resource
        ...
```

Prefer a constructor that accepts all of it:

```python
type ClockPort = Callable[[], datetime]

class OrderProcessorImpl:
    def __init__(
        self,
        payment: PaymentPort, # a port
        clock: ClockPort, # a port, spelled as a type
        currency: str # configuration
    ):
        self.payment = payment
        self.clock = clock
        self.currency = currency

    def __call__(self, order: Order) -> Receipt:
        placed_at = self.clock()
        ...
```
