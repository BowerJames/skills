# Factory Pattern in Python

The rules in [SKILL.md](SKILL.md) are language-agnostic. This file is their Python spelling. The vocabulary's Python shapes — Protocol and Callable mechanics — live in [codebase-design's python.md](../codebase-design/python.md).

The examples build on this `OrderProcessor` module and its adapters:

```python
class PaymentPort(Protocol): # Port defined for the module
    def charge(self, amount: float) -> bool: ...

class OrderProcessor(Protocol): # Interface for the module
    def __call__(self, order: Order) -> Receipt: ...

class OrderProcessorImpl: # Implementation of the module
    def __init__(self, payment: PaymentPort, currency: str = "GBP"): ... # Constructor for the module

    def __call__(self, order: Order) -> Receipt: ...

class StripeAdapter(PaymentPort): # Adapter for the payment port
    def __init__(self, api_key: str, http: HTTP): ...

    def charge(self, amount: float) -> bool: ...

class PaypalAdapter(PaymentPort): # Adapter for the payment port
    def __init__(self, api_key: str, http: HTTP): ...

    def charge(self, amount: float) -> bool: ...
```

## Rule 1: a factory returns the interface

Bad example:

```python
def create_order_processor(
        type: Literal["stripe", "paypal"]
    ) -> OrderProcessorImpl: # This is not a factory for the OrderProcessor module as it declares the return type as the implementation of the module, not the module interface.
    if type == "stripe":
        payment = StripeAdapter(...)
    elif type == "paypal":
        payment = PaypalAdapter(...)
    return OrderProcessorImpl(payment)
```

Good example:

```python
def create_order_processor(
        type: Literal["stripe", "paypal"]
    ) -> OrderProcessor: # This is now a factory for the OrderProcessor module since it declares the return type as the interface for the module.
    if type == "stripe":
        payment = StripeAdapter(...)
    elif type == "paypal":
        payment = PaypalAdapter(...)
    return OrderProcessorImpl(payment)
```

## Rule 2: a factory holds no behaviour

Bad example:

```python
def create_order_processor(
        type: Literal["stripe", "paypal"]
    ) -> OrderProcessor:
    if type == "stripe":
        payment = StripeAdapter(...)
    elif type == "paypal":
        payment = PaypalAdapter(...)
    order_processor = OrderProcessorImpl(payment)

    def with_retry(order: Order) -> Receipt: # Retrying is a behaviour, and no test adapter can reach it here.
        for _ in range(3):
            try:
                return order_processor(order)
            except PaymentFailed:
                continue
        raise PaymentFailed

    return with_retry
```

Retrying belongs behind a constructor: in `OrderProcessorImpl`, or in an adapter that wraps another adapter. The factory then only chooses and constructs:

```python
def create_order_processor(
        type: Literal["stripe", "paypal"]
    ) -> OrderProcessor:
    if type == "stripe":
        payment = StripeAdapter(...)
    elif type == "paypal":
        payment = PaypalAdapter(...)
    return OrderProcessorImpl(RetryingAdapter(payment, attempts=3))
```

## Rule 3: one constructor, many factories

Each factory is one way of supplying the single constructor:

```python
def create_stripe_order_processor() -> OrderProcessor:
    return OrderProcessorImpl(StripeAdapter(...))

def create_paypal_order_processor() -> OrderProcessor:
    return OrderProcessorImpl(PaypalAdapter(...))
```

Bad example, a second way of constructing added to the implementation:

```python
class OrderProcessorImpl:
    def __init__(self, payment: PaymentPort): ...

    @classmethod
    def with_stripe(cls) -> "OrderProcessorImpl": # A second constructor: the implementation now knows about an adapter.
        return cls(StripeAdapter(...))
```

## Rule 4: a factory never calls another module's constructor

A `Checkout` module has a port satisfied by an adapter that calls an `OrderProcessor`.

Bad example:

```python
def create_checkout() -> Checkout:
    order_processor = OrderProcessorImpl(StripeAdapter(...)) # Calls another module's constructor, so how an OrderProcessor is built is now known in two places.
    return CheckoutImpl(OrderProcessorAdapter(order_processor))
```

Good examples:

```python
def create_checkout() -> Checkout:
    order_processor = create_order_processor("stripe") # Another module's factory
    return CheckoutImpl(OrderProcessorAdapter(order_processor))
```

```python
def create_checkout(order_processor: OrderProcessor) -> Checkout: # The entity is taken as a parameter
    return CheckoutImpl(OrderProcessorAdapter(order_processor))
```

## Rule 5: a factory is where resources are first reached

The factory reads the environment and passes what it reads to the constructors:

```python
def create_order_processor(
        type: Literal["stripe", "paypal"] # A parameter that selects between choices
    ) -> OrderProcessor:
    if type == "stripe":
        payment = StripeAdapter(
            api_key=os.environ["STRIPE_API_KEY"], # The factory reaches the resource and passes the value on
            http=stripe.RealHTTP() # The factory constructs what the adapter's own port needs, from the adapter's package
        )
    elif type == "paypal":
        payment = PaypalAdapter(
            api_key=os.environ["PAYPAL_API_KEY"],
            http=paypal.RealHTTP()
        )
    return OrderProcessorImpl(
        payment,
        currency=os.environ["CURRENCY"] # Configuration for the module, read here and not in the constructor
    )
```

What a factory takes depends on its use case. This one takes the adapter and supplies only the configuration:

```python
def create_order_processor(
        payment: PaymentPort # An adapter chosen by the caller
    ) -> OrderProcessor:
    return OrderProcessorImpl(payment, currency=os.environ["CURRENCY"])
```
