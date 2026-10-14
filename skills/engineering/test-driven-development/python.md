# Test-driven development in Python

Below follows an example for a TDD approach towards building an `OrderProcessor` module. Nothing exists before the first test. Files are placed as [project-structure](../project-structure/SKILL.md) describes.

## First TDD Cycle

**Red**

The first thing we need is a constructor for producing order processors:

```python
# tests/unit/core/order_processor/test_order_processor.py
import pytest

from core.order_processor import OrderProcessor, OrderProcessorImpl

@pytest.fixture
def order_processor() -> OrderProcessor:
    return OrderProcessorImpl()

def test_can_create_order_processor(order_processor):
    pass
```

At this point the tests will fail but they will fail for the 'wrong reason'. They will fail because `OrderProcessorImpl` and `OrderProcessor` will not import. A true red state requires that the type checker passes but the tests are failing. So the module's package is created with its interface, its implementation and its export file:

```python
# core/order_processor/interface.py
from typing import Protocol

class OrderProcessor(Protocol):
    ...
```

```python
# core/order_processor/implementation.py
class OrderProcessorImpl:
    def __init__(self):
        raise NotImplementedError
```

```python
# core/order_processor/__init__.py
from .implementation import OrderProcessorImpl
from .interface import OrderProcessor

__all__ = ["OrderProcessor", "OrderProcessorImpl"]
```

Now the test will fail for the right reason and the red state has been completed.

**Green**

```python
# core/order_processor/implementation.py
class OrderProcessorImpl:
    def __init__(self):
        pass
```

Green state achieved.

## Second TDD Cycle

**Red**

The next behaviour to pin would be that an order processor charges the total of the items in the order. The module has to charge something, so this behaviour needs a port. The module declares it, and a test adapter for it is written in the same cycle:

```python
# tests/unit/core/order_processor/adapters.py
from core.order_processor import PaymentPort

class FakePaymentAdapter(PaymentPort):
    def __init__(self):
        self.charges = []

    def charge(self, amount: float) -> bool:
        self.charges.append(amount)
        return True
```

Then update the tests:

```python
# tests/unit/core/order_processor/test_order_processor.py
import pytest

from core.domain.order import Order, OrderItem
from core.order_processor import OrderProcessor, OrderProcessorImpl

from .adapters import FakePaymentAdapter

@pytest.fixture
def payment():
    return FakePaymentAdapter()

@pytest.fixture
def order_processor(payment) -> OrderProcessor:
    return OrderProcessorImpl(
        payment=payment
    )

def order_with(prices: list[float]):
    order_items = [
        OrderItem(
            name=f"item_{i}",
            cost=price
        ) for i, price in enumerate(prices)
    ]
    return Order(
        id="order_id",
        items=order_items
    )

def test_can_create_order_processor(
    order_processor
):
    pass

@pytest.mark.parametrize("prices, expected_charge", [
    ([100], 100),
    ([100, 40], 140),
])
def test_charges_total_of_all_order_items(
    prices,
    expected_charge,
    payment,
    order_processor
):
    order = order_with(prices)
    order_processor(order)
    assert payment.charges == [expected_charge]
```

Now the type checker needs to be made to pass. `Order` and `Receipt` cross the module's interface, so they are data carriers in the domain package:

```python
# core/domain/order.py
from dataclasses import dataclass

@dataclass(frozen=True)
class OrderItem:
    name: str
    cost: float

@dataclass(frozen=True)
class Order:
    id: str
    items: list[OrderItem]
```

```python
# core/domain/receipt.py
from dataclasses import dataclass

@dataclass(frozen=True)
class Receipt:
    total: float
```

The port goes in the module's ports file and the interface gains its method:

```python
# core/order_processor/ports.py
from typing import Protocol

class PaymentPort(Protocol):
    def charge(self, amount: float) -> bool:
        ...
```

```python
# core/order_processor/interface.py
from typing import Protocol

from core.domain.order import Order
from core.domain.receipt import Receipt

class OrderProcessor(Protocol):
    def __call__(self, order: Order) -> Receipt:
        ...
```

```python
# core/order_processor/implementation.py
from core.domain.order import Order
from core.domain.receipt import Receipt

from .ports import PaymentPort

class OrderProcessorImpl:
    def __init__(self, payment: PaymentPort):
        self.payment = payment

    def __call__(self, order: Order) -> Receipt:
        raise NotImplementedError
```

`PaymentPort` is added to the export file's imports and `__all__`.

**Green**

```python
# core/order_processor/implementation.py
from core.domain.order import Order
from core.domain.receipt import Receipt

from .ports import PaymentPort

class OrderProcessorImpl:
    def __init__(self, payment: PaymentPort):
        self.payment = payment

    def __call__(self, order: Order) -> Receipt:
        total = sum(
            [
                item.cost for item in order.items
            ]
        )
        self.payment.charge(total)
        return Receipt(
            total=total
        )
```

This completes the second cycle. And the cycles would continue until the required behaviours are all pinned.

## After the cycles

The module now exists in the core and has only ever been constructed with a test adapter. The Stripe adapter for `PaymentPort`, the factory that passes it to the constructor and the entrypoint are written next: see [project-structure](../project-structure/SKILL.md) for where they go and [factory-pattern](../factory-pattern/SKILL.md) for what the factory does.
