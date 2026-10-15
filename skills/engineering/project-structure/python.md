# Project Structure in Python

The rules in [SKILL.md](SKILL.md) are language-agnostic. This file is their Python spelling.

## Vocabulary → Python

| Term | Python |
|---|---|
| **Package** | a directory with an `__init__.py` |
| **Export file** | the package's `__init__.py`, re-exporting names and listing them in `__all__` |
| **Data carrier** | a frozen `dataclass`, or a type alias such as a `Literal` |
| **Domain package** | `core/domain`, with one file per concept — `core/domain/currency.py`, `core/domain/agent_tool.py` |
| **Domain interface** | a `typing.Protocol` in its own file in `core/domain` — `AgentTool` in `core/domain/agent_tool.py` |
| **Module factory package** | `infrastructure/factories/modules/<module_name>`, with the factories in `factory.py` |
| **Adapter factory package** | `infrastructure/factories/adapters/<module_name>/<adapter_name>`, or `<entrypoint_name>/<adapter_name>` for a driving adapter, or `<interface_name>/<adapter_name>` for an adapter of a domain interface, with the factories in `factory.py` |
| **Entrypoint** | the framework or runtime that calls the program, never written in it: FastAPI, Click, Celery, or Python running a file as `__main__`, named `main` |
| **Driving adapter** | a class in `infrastructure/adapters/<entrypoint_name>/<adapter_name>` — `Cli` in `infrastructure/adapters/main/cli/adapter.py` |
| **Startup code** | a file in `infrastructure/startup`, one per entrypoint — `infrastructure/startup/main.py` with its `if __name__ == "__main__":` block |

Python calls every `.py` file a "module". In these skills **module** keeps its codebase-design meaning, so this file says "file" for a `.py` file.

Files behind the export file carry no `_` prefix. The export file decides what is public: a name is public if `__init__.py` re-exports it and lists it in `__all__`, and private otherwise.

## Example 1: a module and its adapters

An `OrderProcessor` module charges for an order through its `OrderProcessorPort`. A Stripe adapter and a PayPal adapter each satisfy that port.

```
src/
  core/
    domain/
      __init__.py
      currency.py              # Currency
    order_processor/
      __init__.py              # the export file
      interface.py             # OrderProcessor
      port.py                  # OrderProcessorPort
      data.py                  # Order, OrderItem, Receipt
      errors.py                # PaymentDeclined
      implementation.py        # OrderProcessorImpl; could be several files
  infrastructure/
    adapters/
      order_processor/
        stripe/
          __init__.py          # the export file
          adapter.py           # StripeAdapter: takes an httpx.Client
        paypal/
          __init__.py          # the export file
          adapter.py           # PaypalAdapter: takes an httpx.Client
      main/                    # the driving adapters of Python's __main__
        cli/
          __init__.py          # the export file
          adapter.py           # Cli
    factories/
      modules/
        order_processor/
          __init__.py          # the export file
          factory.py           # create_order_processor
      adapters/
        order_processor/
          stripe/
            __init__.py        # the export file
            factory.py         # create_stripe_adapter
          paypal/
            __init__.py        # the export file
            factory.py         # create_paypal_adapter
        main/
          cli/
            __init__.py        # the export file
            factory.py         # create_cli
    startup/
      main.py                  # the startup code for Python's __main__
```

`Currency` means the same thing to every module and belongs to none, so it is in the domain package. `Order` and `Receipt` appear in the `OrderProcessor` interface, so they are in the module's data file:

```python
# core/domain/currency.py
from typing import Literal

type Currency = Literal["GBP", "USD", "EUR"]
```

```python
# core/order_processor/data.py
from dataclasses import dataclass

from core.domain.currency import Currency

@dataclass(frozen=True)
class OrderItem:
    name: str
    cost: int # in the currency's minor unit

@dataclass(frozen=True)
class Order:
    id: str
    items: tuple[OrderItem, ...]

@dataclass(frozen=True)
class Receipt:
    total: int
    currency: Currency
```

The interface, the port, the errors and the implementation each have their own file:

```python
# core/order_processor/interface.py
from typing import Protocol

from .data import Order, Receipt

class OrderProcessor(Protocol): # Interface for the module
    def __call__(self, order: Order) -> Receipt: ...
```

```python
# core/order_processor/port.py
from typing import Protocol

from core.domain.currency import Currency

class OrderProcessorPort(Protocol): # The module's port
    def charge(self, amount: int, currency: Currency) -> bool: ...
```

```python
# core/order_processor/errors.py
class PaymentDeclined(Exception): # Determined by the module's implementation
    ...
```

```python
# core/order_processor/implementation.py
from core.domain.currency import Currency

from .data import Order, Receipt
from .errors import PaymentDeclined
from .port import OrderProcessorPort

class OrderProcessorImpl: # Implementation of the module
    def __init__(
        self,
        port: OrderProcessorPort,
        currency: Currency
    ): # Constructor for the module
        self.port = port
        self.currency = currency

    def __call__(self, order: Order) -> Receipt:
        ...
```

```python
# core/order_processor/__init__.py
from .data import Order, OrderItem, Receipt
from .errors import PaymentDeclined
from .implementation import OrderProcessorImpl
from .interface import OrderProcessor
from .port import OrderProcessorPort

__all__ = [
    "Order",
    "OrderItem",
    "OrderProcessor",
    "OrderProcessorImpl",
    "OrderProcessorPort",
    "PaymentDeclined",
    "Receipt",
]
```

An adapter imports the port from the module's export file:

```python
# infrastructure/adapters/order_processor/stripe/adapter.py
from core.domain.currency import Currency
from core.order_processor import OrderProcessorPort                # good: the export file
from core.order_processor.port import OrderProcessorPort           # bad: reaches behind the export file
from infrastructure.adapters.order_processor.paypal import ...     # bad: an adapter importing another adapter

import httpx                                                       # good: the library whose client it takes

class StripeAdapter(OrderProcessorPort): # Adapter for the module's port
    def __init__(self, api_key: str, http: httpx.Client): ... # Constructor: takes the library's own client

    def charge(self, amount: int, currency: Currency) -> bool: ...
```

Nothing in `core` imports from `infrastructure`:

```python
# core/order_processor/implementation.py
from infrastructure.adapters.order_processor.stripe import StripeAdapter   # bad: the core importing an adapter
```

## Example 2: the factory packages, a driving adapter and its startup code

The adapter's export file exports its constructor, which is what its adapter factory needs:

```python
# infrastructure/adapters/order_processor/stripe/__init__.py
from .adapter import StripeAdapter

__all__ = ["StripeAdapter"]
```

The adapter factory builds the library objects the adapter takes and calls its constructor. It lives under `infrastructure/factories/adapters`, at the adapter's path:

```python
# infrastructure/factories/adapters/order_processor/stripe/factory.py
import httpx

from core.order_processor import OrderProcessorPort
from infrastructure.adapters.order_processor.stripe import StripeAdapter

def create_stripe_adapter(api_key: str) -> OrderProcessorPort: # An adapter factory: returns the port the adapter satisfies
    return StripeAdapter(api_key, httpx.Client()) # Building the client reaches no resource: it connects when called
```

```python
# infrastructure/factories/adapters/order_processor/stripe/__init__.py
from .factory import create_stripe_adapter

__all__ = ["create_stripe_adapter"]
```

The module factory chooses an adapter by calling its adapter factory, and never imports from an adapter's package:

```python
# infrastructure/factories/modules/order_processor/factory.py
from typing import Literal

from core.domain.currency import Currency
from core.order_processor import OrderProcessor, OrderProcessorImpl
from infrastructure.factories.adapters.order_processor.paypal import create_paypal_adapter
from infrastructure.factories.adapters.order_processor.stripe import create_stripe_adapter
from infrastructure.adapters.order_processor.stripe import StripeAdapter   # bad: a module factory reaching into an adapter's package

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

```python
# infrastructure/factories/modules/order_processor/__init__.py
from .factory import create_order_processor

__all__ = ["create_order_processor"]
```

The entrypoint here is Python itself, running the startup file as `__main__`; the project does not write it. What it reaches is a driving adapter: `Cli` has the shape of a command, a callable given the command-line arguments. Its constructor takes the module interface it drives, and it translates between the arguments and data carriers:

```python
# infrastructure/adapters/main/cli/adapter.py
from core.order_processor import Order, OrderItem, OrderProcessor

class Cli: # A driving adapter: calls no module's constructor and no factory, imports no other adapter
    def __init__(self, order_processor: OrderProcessor): # Its constructor takes the interface it drives
        self.order_processor = order_processor

    def __call__(self, argv: list[str]) -> None: # The shape the entrypoint calls: a command
        order = Order(
            id=argv[0],
            items=tuple(OrderItem(name=f"item_{i}", cost=int(cost)) for i, cost in enumerate(argv[1:]))
        ) # translate the outside world into a data carrier
        receipt = self.order_processor(order)
        print(f"total={receipt.total} {receipt.currency}")
```

Like every adapter, it has an adapter factory:

```python
# infrastructure/factories/adapters/main/cli/factory.py
from collections.abc import Callable

from core.order_processor import OrderProcessor
from infrastructure.adapters.main.cli import Cli

def create_cli(order_processor: OrderProcessor) -> Callable[[list[str]], None]: # Returns the entrypoint's port: a command
    return Cli(order_processor)
```

The startup code reads the environment, calls the factories, and hands the driving adapter to the entrypoint. For Python's `__main__`, that means calling it with the arguments:

```python
# infrastructure/startup/main.py
import os
import sys

from infrastructure.factories.adapters.main.cli import create_cli
from infrastructure.factories.modules.order_processor import create_order_processor

if __name__ == "__main__": # The startup code: run with `python -m infrastructure.startup.main`
    cli = create_cli(
        create_order_processor(
            provider="stripe",
            api_key=os.environ["STRIPE_API_KEY"],
            currency="GBP"
        )
    )
    cli(sys.argv[1:])
```

With a framework as the entrypoint, the startup code registers the driving adapters with it instead, and the framework defines the port. For FastAPI, a driving adapter is a function that returns an `APIRouter`, and the startup code includes each in the app:

```python
# infrastructure/startup/fastapi.py
import os

from fastapi import FastAPI

from infrastructure.factories.adapters.fastapi.orders_router import create_orders_router
from infrastructure.factories.modules.order_processor import create_order_processor

app = FastAPI() # The entrypoint, configured: uvicorn runs infrastructure.startup.fastapi:app
app.include_router(
    create_orders_router(
        create_order_processor(provider="stripe", api_key=os.environ["STRIPE_API_KEY"], currency="GBP")
    )
)
```

When the entity a driving adapter drives holds one request's resources, the driving adapter takes the opening of it, and the startup code satisfies it with a module factory. A `PhoneCall` module whose entity talks to a caller over their socket is built per call:

```python
# infrastructure/adapters/fastapi/call_router/router.py
from collections.abc import Callable

from fastapi import APIRouter, WebSocket

from core.phone_call import CallDetails, PhoneCall

type OpenPhoneCall = Callable[[WebSocket, CallDetails], PhoneCall] # Opens the entity per request: a function returning the module's interface

def call_router(*, path: str, open_call: OpenPhoneCall) -> APIRouter: # Its constructor: returns the APIRouter FastAPI calls
    router = APIRouter()

    @router.websocket(path)
    async def call(websocket: WebSocket, ddiNo: str) -> None:
        await open_call(websocket, CallDetails(ddi_number=ddiNo)).run()

    return router
```

```python
# infrastructure/startup/fastapi.py
app.include_router(
    create_call_router(path="/call", open_call=create_phone_call) # The adapter factory, given the phone call's module factory as the opener
)
```

## Example 3: a module that needs another module

A `Checkout` module is built on top of order processing. It does not import `core/order_processor`. It declares its own data carriers and a port with the shape it needs:

```python
# core/checkout/data.py
from dataclasses import dataclass

@dataclass(frozen=True)
class CartLine:
    product: str
    price: int

@dataclass(frozen=True)
class Cart:
    id: str
    lines: tuple[CartLine, ...]
```

```python
# core/checkout/port.py
from typing import Protocol

from .data import Cart

class CheckoutPort(Protocol): # The checkout module's port
    def pay(self, cart: Cart) -> int: ...
```

An adapter under `infrastructure/adapters/checkout` satisfies the port by calling an `OrderProcessor` through its interface, translating between the two modules' data carriers:

```python
# infrastructure/adapters/checkout/order_processor/adapter.py
from core.checkout import Cart, CheckoutPort
from core.order_processor import Order, OrderItem, OrderProcessor

class OrderProcessorAdapter(CheckoutPort): # Adapter connecting checkout to the order processor module
    def __init__(self, order_processor: OrderProcessor): # Constructor takes the interface, never the implementation
        self.order_processor = order_processor

    def pay(self, cart: Cart) -> int:
        order = Order(
            id=cart.id,
            items=tuple(OrderItem(name=line.product, cost=line.price) for line in cart.lines)
        ) # checkout's data, translated into the order processor's
        return self.order_processor(order).total
```

The adapter's factory gets an `OrderProcessor` from the order processor's module factory package and passes it to the adapter's constructor:

```python
# infrastructure/factories/adapters/checkout/order_processor/factory.py
from core.checkout import CheckoutPort
from infrastructure.adapters.checkout.order_processor import OrderProcessorAdapter
from infrastructure.factories.modules.order_processor import create_order_processor

def create_order_processor_adapter(api_key: str) -> CheckoutPort: # An adapter factory for the connecting adapter
    return OrderProcessorAdapter(create_order_processor("stripe", api_key, "GBP"))
```

The checkout module factory gets its adapter from that factory:

```python
# infrastructure/factories/modules/checkout/factory.py
from core.checkout import Checkout, CheckoutImpl
from infrastructure.factories.adapters.checkout.order_processor import create_order_processor_adapter

def create_checkout(api_key: str) -> Checkout: # A module factory for the checkout module
    return CheckoutImpl(create_order_processor_adapter(api_key))
```

## Example 4: the tests

```
tests/
  unit/
    core/
      order_processor/
        test_order_processor.py    # constructs OrderProcessorImpl with FakeOrderProcessorAdapter
        adapters.py                # FakeOrderProcessorAdapter: the test adapter for this module's port
      checkout/
        test_checkout.py
        adapters.py                # FakeCheckoutAdapter
    infrastructure/
      adapters/
        order_processor/
          stripe/
            test_stripe.py         # constructs StripeAdapter with an httpx.Client on httpx.MockTransport
        checkout/
          order_processor/
            test_order_processor.py  # constructs OrderProcessorAdapter with FakeOrderProcessor
            adapters.py              # FakeOrderProcessor
        main/
          cli/
            test_cli.py            # constructs Cli with FakeOrderProcessor
            adapters.py            # FakeOrderProcessor
  integration/
    infrastructure/
      adapters/
        order_processor/
          postgres/
            test_postgres.py       # runs PostgresAdapter against a real database
  e2e/
    test_main.py                   # where the project has end-to-end tests: runs infrastructure/startup/main.py
```

A test imports the entity under test from its export file and its test adapters from beside itself:

```python
# tests/unit/core/order_processor/test_order_processor.py
from core.order_processor import OrderProcessor, OrderProcessorImpl

from .adapters import FakeOrderProcessorAdapter
```

The tests for `Checkout` do not import `FakeOrderProcessorAdapter` or anything else from the order processor's tests. `Checkout` has its own port, so its tests have their own test adapter. The two `FakeOrderProcessor` test adapters are duplicated for the same reason.

## Example 5: legacy code

A project has a legacy `billing.py` that already talks to Stripe. A new `OrderProcessor` module reaches it through its port, with an adapter named for it:

```
src/
  billing.py                     # legacy code, left where it is
  core/
    order_processor/
  infrastructure/
    adapters/
      order_processor/
        legacy_billing/
          __init__.py
          adapter.py             # LegacyBillingAdapter: satisfies OrderProcessorPort by calling billing.py
    factories/
      adapters/
        order_processor/
          legacy_billing/
            __init__.py
            factory.py           # create_legacy_billing_adapter
```

```python
# infrastructure/adapters/order_processor/legacy_billing/adapter.py
import billing

from core.domain.currency import Currency
from core.order_processor import OrderProcessorPort

class LegacyBillingAdapter(OrderProcessorPort):
    def charge(self, amount: int, currency: Currency) -> bool:
        return billing.take_payment(amount / 100, currency) == "ok"
```

Legacy code that needs an `OrderProcessor` calls the factory and drives the interface:

```python
# src/views.py, legacy code
from infrastructure.factories.modules.order_processor import create_order_processor   # good: a module factory
from core.order_processor import OrderProcessorImpl                                  # bad: the constructor
```

## Example 6: a domain interface and its adapters

A `ToolRunner` module runs the tools a model calls. A tool means the same thing to every entity that uses it, so `AgentTool` is a domain interface, and every tool is an adapter of it:

```
src/
  core/
    domain/
      agent_tool.py              # AgentTool: a domain interface
    tool_runner/
      port.py                    # ToolRunnerPort: tools(), using AgentTool
  infrastructure/
    adapters/
      agent_tool/                # the adapters of the AgentTool domain interface
        order_processor/
          adapter.py             # PlaceOrderTool: places an order through the OrderProcessor module
          params.py              # PlaceOrderParams: the pydantic model of its arguments
        structured/
          adapter.py             # StructuredTool: validates arguments against a pydantic model
      tool_runner/
        toolset/
          adapter.py             # Toolset: offers the runner the tools it is given
    factories/
      adapters/
        agent_tool/
          order_processor/       # create_place_order_tool
          structured/            # create_structured_tool
        tool_runner/
          toolset/               # create_toolset
      modules/
        tool_runner/             # create_tool_runner
```

The domain interface has a shape and no behaviour. The module's one port uses it:

```python
# core/domain/agent_tool.py
from collections.abc import Mapping
from typing import Any, Protocol

class AgentTool(Protocol): # A domain interface: the same meaning to every entity that uses it
    @property
    def name(self) -> str: ...
    @property
    def description(self) -> str: ...
    @property
    def parameters(self) -> Mapping[str, Any]: ... # the JSON Schema of execute's arguments
    async def execute(self, arguments: dict[str, Any]) -> str: ...
```

```python
# core/tool_runner/port.py
from collections.abc import Sequence
from typing import Protocol

from core.domain.agent_tool import AgentTool

class ToolRunnerPort(Protocol): # The module's one port; it uses the domain interface
    def tools(self) -> Sequence[AgentTool]: ...
```

Each tool is an adapter of the domain interface, under `infrastructure/adapters/agent_tool`. One connects the order processor module, so it is named for that module:

```python
# infrastructure/adapters/agent_tool/order_processor/adapter.py
from typing import Any

from core.domain.agent_tool import AgentTool
from core.order_processor import Order, OrderItem, OrderProcessor

class PlaceOrderTool(AgentTool): # An adapter of the domain interface, connecting the order processor module
    name = "place_order"
    description = "Place the caller's order."
    parameters = {"type": "object", "properties": {"id": {"type": "string"}, "costs": {"type": "array"}}}

    def __init__(self, order_processor: OrderProcessor): # Takes the module's interface, never its implementation
        self.order_processor = order_processor

    async def execute(self, arguments: dict[str, Any]) -> str:
        order = Order(
            id=arguments["id"],
            items=tuple(OrderItem(name=f"item_{i}", cost=cost) for i, cost in enumerate(arguments["costs"]))
        )
        receipt = self.order_processor(order)
        return f"Charged {receipt.total} {receipt.currency}."
```

A generic adapter of the same interface is reused by wrapping, never by importing. `StructuredTool` validates the arguments against a pydantic model, then passes them to the tool it wraps:

```python
# infrastructure/adapters/agent_tool/structured/adapter.py
from collections.abc import Mapping
from typing import Any

from pydantic import BaseModel, ValidationError

from core.domain.agent_tool import AgentTool

class StructuredTool(AgentTool): # A generic adapter of the domain interface
    def __init__(self, params: type[BaseModel], inner: AgentTool): # The tool it wraps, typed as the interface
        self.params = params
        self.inner = inner

    @property
    def name(self) -> str:
        return self.inner.name

    @property
    def description(self) -> str:
        return self.inner.description

    @property
    def parameters(self) -> Mapping[str, Any]:
        return self.params.model_json_schema() # The schema comes from the model, not the inner tool

    async def execute(self, arguments: dict[str, Any]) -> str:
        try:
            valid = self.params.model_validate(arguments)
        except ValidationError as error:
            return error.json() # handled: the model reads which arguments to correct
        return await self.inner.execute(valid.model_dump())
```

The factories combine them. A tool's factory gets the generic adapter from its factory, and the module factory gets the toolset from its own:

```python
# infrastructure/factories/adapters/agent_tool/order_processor/factory.py
from core.domain.agent_tool import AgentTool
from core.order_processor import OrderProcessor
from infrastructure.adapters.agent_tool.order_processor import PlaceOrderParams, PlaceOrderTool
from infrastructure.factories.adapters.agent_tool.structured import create_structured_tool

def create_place_order_tool(order_processor: OrderProcessor) -> AgentTool: # Returns the domain interface it satisfies
    return create_structured_tool(params=PlaceOrderParams, inner=PlaceOrderTool(order_processor))
```

```python
# infrastructure/factories/modules/tool_runner/factory.py
from core.order_processor import OrderProcessor
from core.tool_runner import ToolRunner, ToolRunnerImpl
from infrastructure.factories.adapters.agent_tool.order_processor import create_place_order_tool
from infrastructure.factories.adapters.tool_runner.toolset import create_toolset

def create_tool_runner(order_processor: OrderProcessor) -> ToolRunner:
    return ToolRunnerImpl(create_toolset([create_place_order_tool(order_processor)]))
```

The tests for `ToolRunner` pass a fake port returning fake `AgentTool`s; each tool adapter is tested on its own, through the domain interface, under `tests/unit/infrastructure/adapters/agent_tool/<adapter_name>`.

## Checking the rules

[import-linter](https://import-linter.readthedocs.io/) can enforce the import rules: a `layers` contract places `infrastructure` above `core`, a `forbidden` contract stops `core.domain` importing from any module's package, and an `independence` contract stops the packages of modules importing each other. A `forbidden` contract with `include_external_packages` can also stop `core` importing any third-party library.

import-linter works on files, not names, so it cannot tell an import of `OrderProcessorImpl` from an import of `OrderProcessor`. Ruff's `banned-api` setting (rule `TID251`) can: ban each constructor by its full name, `core.order_processor.OrderProcessorImpl`, and switch the rule off with `per-file-ignores` for `src/infrastructure/factories/modules/` and `tests/`. Ban each adapter's constructor the same way, with the rule switched off for `src/infrastructure/factories/adapters/` and `tests/`.
