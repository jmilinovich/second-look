# Test Patterns

Patterns to reach for when writing or rewriting tests. Python/pytest examples, but the principles are language-agnostic.

## Arrange, Act, Assert (AAA)

Structure every test in three distinct sections, separated by blank lines:

```python
def test_discount_applies_to_cart_subtotal():
    # Arrange
    cart = Cart()
    cart.add(Item(name="shirt", price=20))
    cart.add(Item(name="hat", price=10))
    discount = PercentageDiscount(0.10)

    # Act
    result = discount.apply(cart)

    # Assert
    assert result.total == 27.00
```

Why: it makes the test's intent scannable. A reader knows exactly where the inputs are, where the behavior-under-test is exercised, and where the expectation lives. When Arrange grows past a few lines, extract a helper or fixture. When you find yourself writing a second Act, split the test.

The BDD "Given-When-Then" framing is the same idea under a different name. Either works.

## Table-driven tests with `pytest.mark.parametrize`

When you'd otherwise write N nearly-identical test functions, use parametrize. Each row runs as its own test, fails individually, and adding a new case is one line.

```python
import pytest
from tax import calculate_tax

@pytest.mark.parametrize("subtotal,rate,expected", [
    (100.00, 0.10, 10.00),      # basic case
    (0.00,   0.10, 0.00),       # zero subtotal
    (100.00, 0.00, 0.00),       # zero rate
    (99.99,  0.0725, 7.249),    # common real-world rate
    (-50.00, 0.10, 0.00),       # negative subtotal clamps to 0
])
def test_calculate_tax(subtotal, rate, expected):
    assert calculate_tax(subtotal, rate) == pytest.approx(expected)
```

Use this whenever you're testing the same function with different inputs. Signs you should parametrize:

- You have three or more test functions with similar setup and one thing that varies.
- You're tempted to loop inside a single test over cases (don't — you lose individual failure reporting).
- The expected behavior is fundamentally a table (inputs → outputs).

Give each case an `id` when the parameters aren't self-describing:

```python
@pytest.mark.parametrize("input,expected", [
    pytest.param("", "", id="empty"),
    pytest.param("a", "A", id="single_char"),
    pytest.param("hello", "Hello", id="word"),
    pytest.param("hello world", "Hello World", id="multi_word"),
])
def test_title_case(input, expected):
    assert title_case(input) == expected
```

Caveat: parametrize values are untyped at collection time, so mypy won't catch type errors in the table. Worth it for most cases, but keep tables small enough to eyeball.

## Mock at boundaries, not internals

A mock is a port stub — it replaces something that would otherwise require I/O, network, or unpredictable state. Mocks belong at the edges of your system.

**Good boundaries to mock:**
- Database clients
- HTTP clients to external services
- Message queues, pubsub
- Filesystem (when the real FS would be slow or non-deterministic)
- Time (`datetime.now`, `time.time`)
- Random number generators

**Bad things to mock:**
- Your own modules calling each other
- Pure functions
- Data classes or simple value objects
- Anything whose real implementation is fast, deterministic, and has no side effects

Example of the right level of mocking:

```python
def test_create_user_sends_welcome_email(monkeypatch):
    # Arrange
    sent_emails = []
    def fake_send(to, subject, body):
        sent_emails.append({"to": to, "subject": subject})
    monkeypatch.setattr("app.email.send", fake_send)

    # Act
    user = create_user(email="john@example.com", name="John")

    # Assert — assert on observable outcome, not on how it was implemented
    assert user.id is not None
    assert len(sent_emails) == 1
    assert sent_emails[0]["to"] == "john@example.com"
```

Notice: we don't assert *that* `app.email.send` was called, or *how many times*, or in what order relative to other calls. We assert the *outcome*: one welcome email went to the right address. If `create_user` is later refactored to batch emails differently, this test still passes as long as the user gets one welcome email.

Compare to the over-mocked version, which is brittle:

```python
# BAD — asserts on implementation, not behavior
def test_create_user_sends_welcome_email(mocker):
    mock_db = mocker.patch("app.db.insert_user")
    mock_email = mocker.patch("app.email.send")
    mock_logger = mocker.patch("app.logger.info")

    create_user(email="john@example.com", name="John")

    mock_db.assert_called_once()
    mock_email.assert_called_once_with("john@example.com", ANY, ANY)
    mock_logger.assert_called_with("User created: john@example.com")
```

This test breaks if you change the log message, if you decide to batch inserts, or if you add any other side effect. None of those are behavior changes — they're implementation.

## Test naming

Name tests after the behavior being verified, from the caller's perspective:

Good:
- `test_returns_zero_for_tax_exempt_items`
- `test_cart_rejects_negative_quantities`
- `test_login_fails_when_password_is_empty`
- `test_search_returns_empty_list_when_no_matches`

Bad:
- `test_calculate_tax` (which case?)
- `test_cart_1` (numbered, meaningless)
- `test_login_error` (what error?)
- `test_search` (what behavior?)

A good name reads like a spec: if the test passes, this sentence is true about the code. If it fails, the failure line in CI tells you what contract was violated, without needing to open the test.

## Fixtures for shared setup (but not too much)

`pytest.fixture` is great for shared setup that's genuinely shared. It's also a trap: over-fixturing creates tests where you can't tell what the inputs are without jumping through three files.

Rule of thumb: inline the arrange step when it's short and specific to the test. Use a fixture when the same setup appears in 3+ tests.

```python
@pytest.fixture
def authenticated_client():
    client = TestClient(app)
    token = create_test_token(user_id="test-user-1")
    client.headers["Authorization"] = f"Bearer {token}"
    return client

def test_get_profile(authenticated_client):
    response = authenticated_client.get("/profile")
    assert response.status_code == 200
```

Avoid "god fixtures" that set up huge amounts of state. If a fixture has grown past ~20 lines or sets up things unrelated to each other, split it.

## Regression tests are gold

When a bug is found and fixed, write a test that would have caught it, before fixing. Then confirm the test fails, apply the fix, confirm it passes.

These tests are often the highest-signal tests in the suite — they encode real-world failure modes that slipped through every other layer.

Name them descriptively and, when useful, link the bug:

```python
def test_cart_total_handles_fractional_cents():
    """Regression for #2341: floating-point arithmetic was dropping $0.01 on
    orders with many items due to naive accumulation."""
    cart = Cart()
    for _ in range(100):
        cart.add(Item(price=0.99))
    assert cart.total == 99.00
```
