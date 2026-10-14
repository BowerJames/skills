# Testing in Python

The sections in [SKILL.md](SKILL.md) are language-agnostic. This file is its Python spelling. The vocabulary's Python shapes — Protocol and Callable mechanics — live in [codebase-design's python.md](../codebase-design/python.md).

## What makes a good test

We will look at a good test for an `OrderProcessor` module. Consider the setup:

```python
class PaymentPort(Protocol): # Port defined for the module
    def charge(self, amount: float) -> bool: ...

type OrderProcessor = Callable[[Order], Receipt]

class OrderProcessorImpl:
    def __init__(
        self,
        payment: PaymentPort # The port is passed to the constructor of the OrderProcessor module
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

The test adapter — a working fake, no mock library:

```python
class FakePaymentAdapter:                    # a test adapter: decides what comes in, records what goes out
    def __init__(self) -> None:
        self.charges: list[float] = []

    def charge(self, amount: float) -> bool: # satisfies the port by shape — no inheritance
        self.charges.append(amount)          # records what goes out through the port
        return True                          # decides what comes in
```

The test:

```python
def order_with(prices: list[float]) -> Order:  # a harness helper: construction lives in the kit, not the test
    return Order(id="order_id", items=[OrderItem(name=f"item_{i}", cost=price) for i, price in enumerate(prices)])

def setup(
    payment: PaymentPort
) -> OrderProcessor: # Use a helper to wrap the module constructor so future changes to the constructor design can be localised in the test file
    return OrderProcessorImpl(
        payment
    )

@pytest.mark.parametrize("prices, expected_charge", [
    ([100], 100),                            # worked examples, chosen by hand
    ([100, 40], 140),
])
def test_charges_the_payment_port_the_order_total(prices, expected_charge):
    payment = FakePaymentAdapter()               # the test adapter for the port
    order_processor = setup(payment)             # constructed through the constructor, never a factory
    order_processor(order_with(prices))          # driven through the interface
    assert payment.charges == [expected_charge]  # the outcome goes out through the port — seen by the adapter that receives it
```

- The name is the behaviour, and the assertions verify exactly it — a name promising one behaviour while the assertions check another is a test carrying two behaviours.
- Each parametrize case is one condition of the one behaviour: a second condition is a second case, not a second assertion.
- The expected charges are worked examples (`100 + 40 = 140`, by hand) — the test never computes them with source code.

## Testing the infrastructure

An adapter is tested through the port it satisfies and its own constructor. A `StripeAdapter` that takes its HTTP dependency in the constructor can be unit tested with a test adapter:

```python
type HTTP = Callable[[Request], Response]

class StripeAdapter(PaymentPort):
    def __init__(self, api_key: str, http: HTTP): ... # everything required to measure its behaviours

    def charge(self, amount: float) -> bool: ...
```

```python
class FakeHTTP:                                  # a test adapter for what the adapter's constructor takes
    def __init__(self) -> None:
        self.requests: list[Request] = []

    def __call__(self, request: Request) -> Response:
        self.requests.append(request)
        return Response(status=200)

def test_charge_sends_the_amount_in_pence():
    http = FakeHTTP()
    stripe = StripeAdapter(api_key="test-key", http=http)
    stripe.charge(12.50)                         # driven through the port it satisfies
    assert http.requests[0].body["amount"] == 1250
```

Integration tests and end-to-end tests are kept apart from the unit tests with markers, so the default run needs no environment:

```python
@pytest.mark.integration                         # runs against the real resource
def test_saves_and_loads_an_order(postgres_url):
    orders = PostgresAdapter(postgres_url)
    orders.save(order_with([100]))
    assert orders.load("order_id").items[0].cost == 100

@pytest.mark.e2e                                 # runs an entrypoint with the production factories
def test_cli_processes_an_order(capsys):
    main(["process", "--price", "100"])
    assert "total=100" in capsys.readouterr().out
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = ["integration", "e2e"]
addopts = "-m 'not integration and not e2e'"
```

## Anti-patterns

**Tautological** — the oracle is the source code:

```python
from core.order_processor.implementation import calculate_total   # a source-code helper

def test_totals_the_receipt():      # tautological
    payment = FakePaymentAdapter()
    order_processor = setup(payment)
    order = order_with([100, 40])
    assert order_processor(order).total == calculate_total(order)
    # whatever bug calculate_total carries, the expectation inherits —
    # the test agrees by construction and can never disagree
```

The fix is an oracle independent of the source code: computing the expectation in the test from the input is fine (`sum(prices)`); the strongest form is section 1's hand-picked worked examples.

**Implementation-coupled** — patching the implementation's internals:

```python
from unittest.mock import patch

@patch("core.order_processor.implementation.sum")    # reaches inside the implementation
def test_sums_the_prices(fake_sum):
    payment = FakePaymentAdapter()
    order_processor = setup(payment)
    fake_sum.return_value = 100
    order_processor(order_with([100, 40]))
    fake_sum.assert_called_once()
```

The tell: swap `sum()` for a hand-rolled loop — behaviour unchanged, every interface-level test green — and this test goes red. It was never on the outside of the interface. The fix: observation belongs at the interface and the ports — the fake payment adapter's `charges` — not inside the implementation's helpers. (Patching internals and asserting private attributes are this pattern's two common Python shapes.)
