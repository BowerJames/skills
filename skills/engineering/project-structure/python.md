# Project Structure in Python

The rules in [SKILL.md](SKILL.md) are language-agnostic. This file is their Python spelling.

## Vocabulary → Python

| Term | Python |
|---|---|
| **Package** | a directory with an `__init__.py` |
| **Export file** | the package's `__init__.py`, re-exporting names and listing them in `__all__` |
| **Data carrier** | a frozen `dataclass` |
| **Domain package** | `core/domain`, with one file per concept — `core/domain/order.py` |
| **Factory package** | `infrastructure/factories/<module_name>`, with the factories in `factory.py` |
| **Entrypoint** | a file in `infrastructure/entrypoints` — `infrastructure/entrypoints/cli.py` |

Python calls every `.py` file a "module". In these skills **module** keeps its codebase-design meaning, so this file says "file" for a `.py` file.

Files behind the export file carry no `_` prefix. The export file decides what is public: a name is public if `__init__.py` re-exports it and lists it in `__all__`, and private otherwise.

## Example 1: a module and its adapters

An `OrderProcessor` module charges for an order through a `PaymentPort`. Stripe and PayPal each satisfy that port.

```
src/
  core/
    domain/
      __init__.py
      order.py                 # Order, OrderItem
      receipt.py               # Receipt
    order_processor/
      __init__.py              # the export file
      interface.py             # OrderProcessor
      implementation.py        # OrderProcessorImpl; could be several files
      ports.py                 # PaymentPort
  infrastructure/
    adapters/
      order_processor/
        stripe/
          __init__.py          # the export file
          adapter.py           # StripeAdapter
          http.py              # HTTP, RealHTTP: the adapter's own port and what satisfies it
        paypal/
          __init__.py          # the export file
          adapter.py           # PaypalAdapter
          http.py              # HTTP, RealHTTP
    factories/
      order_processor/
        __init__.py            # the export file
        factory.py             # create_order_processor
    entrypoints/
      cli.py
```

The interface, the implementation and the ports each have their own file:

```python
# core/order_processor/interface.py
from typing import Protocol

from core.domain.order import Order
from core.domain.receipt import Receipt

class OrderProcessor(Protocol): # Interface for the module
    def __call__(self, order: Order) -> Receipt: ...
```

```python
# core/order_processor/ports.py
from typing import Protocol

class PaymentPort(Protocol): # Port defined for the module
    def charge(self, amount: float) -> bool: ...
```

```python
# core/order_processor/implementation.py
from core.domain.order import Order
from core.domain.receipt import Receipt

from .ports import PaymentPort

class OrderProcessorImpl: # Implementation of the module
    def __init__(
        self,
        payment: PaymentPort
    ): # Constructor for the module
        self.payment = payment

    def __call__(self, order: Order) -> Receipt:
        ...
```

```python
# core/order_processor/__init__.py
from .implementation import OrderProcessorImpl
from .interface import OrderProcessor
from .ports import PaymentPort

__all__ = ["OrderProcessor", "OrderProcessorImpl", "PaymentPort"]
```

`Order` and `Receipt` appear in the `OrderProcessor` interface, so they are defined in the domain package from the start.

An adapter imports the port from the module's export file:

```python
# infrastructure/adapters/order_processor/stripe/adapter.py
from core.order_processor import PaymentPort                       # good: the export file
from core.order_processor.ports import PaymentPort                 # bad: reaches behind the export file
from infrastructure.adapters.order_processor.paypal import ...     # bad: an adapter importing another adapter

from .http import HTTP                                             # good: the adapter's own port, defined in its own package

class StripeAdapter(PaymentPort): # Adapter for the payment port
    def __init__(self, api_key: str, http: HTTP): ... # Constructor for the stripe adapter

    def charge(self, amount: float) -> bool: ...
```

Nothing in `core` imports from `infrastructure`:

```python
# core/order_processor/implementation.py
from infrastructure.adapters.order_processor.stripe import StripeAdapter   # bad: the core importing an adapter
```

## Example 2: the factory package and an entrypoint

The factory that chooses an adapter imports the adapters, so it lives in `infrastructure/factories/order_processor`, not in `core/order_processor`:

```python
# infrastructure/factories/order_processor/factory.py
from typing import Literal

from core.order_processor import OrderProcessor, OrderProcessorImpl
from infrastructure.adapters.order_processor.paypal import PaypalAdapter
from infrastructure.adapters.order_processor.stripe import StripeAdapter

def create_order_processor(
        type: Literal["stripe", "paypal"]
    ) -> OrderProcessor: # A factory for the order processor module
    if type == "stripe":
        payment = StripeAdapter(...)
    elif type == "paypal":
        payment = PaypalAdapter(...)
    return OrderProcessorImpl(payment)
```

```python
# infrastructure/factories/order_processor/__init__.py
from .factory import create_order_processor

__all__ = ["create_order_processor"]
```

The entrypoint asks the factory for an entity and drives it through its interface. It calls no constructor and imports no adapter:

```python
# infrastructure/entrypoints/cli.py
from core.domain.order import Order
from infrastructure.factories.order_processor import create_order_processor

def main(argv: list[str]) -> None:
    order_processor = create_order_processor("stripe")
    order = Order(...)                 # translate the outside world into a data carrier
    receipt = order_processor(order)
    print(receipt)
```

## Example 3: a module that needs another module

A `Checkout` module is built on top of order processing. It does not import `core/order_processor`. It declares a port with the shape it needs:

```python
# core/checkout/ports.py
from typing import Protocol

from core.domain.order import Order
from core.domain.receipt import Receipt

class OrderProcessingPort(Protocol): # Port defined for the checkout module
    def process(self, order: Order) -> Receipt: ...
```

An adapter under `infrastructure/adapters/checkout` satisfies the port by calling an `OrderProcessor` through its interface:

```python
# infrastructure/adapters/checkout/order_processor/adapter.py
from core.checkout import OrderProcessingPort
from core.domain.order import Order
from core.domain.receipt import Receipt
from core.order_processor import OrderProcessor

class OrderProcessorAdapter(OrderProcessingPort): # Adapter connecting checkout to the order processor module
    def __init__(self, order_processor: OrderProcessor): # Constructor takes the interface, never the implementation
        self.order_processor = order_processor

    def process(self, order: Order) -> Receipt:
        return self.order_processor(order)
```

The checkout factory gets an `OrderProcessor` from the order processor's factory package and passes it to the adapter:

```python
# infrastructure/factories/checkout/factory.py
from core.checkout import Checkout, CheckoutImpl
from infrastructure.adapters.checkout.order_processor import OrderProcessorAdapter
from infrastructure.factories.order_processor import create_order_processor

def create_checkout() -> Checkout: # A factory for the checkout module
    order_processor = create_order_processor("stripe")
    return CheckoutImpl(OrderProcessorAdapter(order_processor))
```

## Example 4: the tests

```
tests/
  unit/
    core/
      order_processor/
        test_order_processor.py    # constructs OrderProcessorImpl with FakePaymentAdapter
        adapters.py                # FakePaymentAdapter: test adapters for this module's ports
      checkout/
        test_checkout.py
        adapters.py                # FakeOrderProcessingAdapter
    infrastructure/
      adapters/
        order_processor/
          stripe/
            test_stripe.py         # constructs StripeAdapter with FakeHTTP
            adapters.py            # FakeHTTP
  integration/
    infrastructure/
      adapters/
        order_processor/
          postgres/
            test_postgres.py       # runs PostgresAdapter against a real database
  e2e/
    test_cli.py                    # runs the cli entrypoint with the production factories
```

A test imports the entity under test from its export file and its test adapters from beside itself:

```python
# tests/unit/core/order_processor/test_order_processor.py
from core.order_processor import OrderProcessor, OrderProcessorImpl

from .adapters import FakePaymentAdapter
```

The tests for `Checkout` do not import `FakePaymentAdapter` or anything else from the order processor's tests. `Checkout` has its own port, so its tests have their own test adapter.

## Checking the rules

[import-linter](https://import-linter.readthedocs.io/) can enforce the import rules: a `layers` contract places `infrastructure` above `core`, a `forbidden` contract stops `core.domain` importing from any module's package, and an `independence` contract stops the packages of modules importing each other. A `forbidden` contract with `include_external_packages` can also stop `core` importing third-party libraries that reach a resource.

import-linter works on files, not names, so it cannot tell an import of `OrderProcessorImpl` from an import of `OrderProcessor`. Ruff's `banned-api` setting (rule `TID251`) can: ban each constructor by its full name, `core.order_processor.OrderProcessorImpl`, and switch the rule off with `per-file-ignores` for `src/infrastructure/factories/` and `tests/`.
