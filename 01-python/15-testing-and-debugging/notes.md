# 15 — Testing & Debugging: Notes

> **Why this matters for an FDE:** you'll change integrations while customers depend on them. Tests let you change code without fear, and debugging skills let you fix a production issue on a customer call in minutes instead of hours.

---

## 1. Reading tracebacks

When Python crashes it prints a **traceback**. Read it **bottom-up**:

```text
Traceback (most recent call last):
  File "/app/sync.py", line 42, in <module>
    main()
  File "/app/sync.py", line 37, in main
    total = compute_total(orders)
  File "/app/sync.py", line 12, in compute_total
    return sum(o["amount"] for o in orders)
  File "/app/sync.py", line 12, in <genexpr>
    return sum(o["amount"] for o in orders)
               ~^^^^^^^^^^
KeyError: 'amount'
```

1. **Last line** = what went wrong: `KeyError: 'amount'` (some order has no `amount` key).
2. **The frame just above** = where: `sync.py`, line 12, in `compute_total`.
3. **Further up** = how we got there (the call chain).
4. Skip frames inside libraries (`site-packages/...`) until you reach **your** code.

Then ask: *what was the data at that moment?* (Usually: one record is shaped differently from what you assumed.)

---

## 2. Debugging techniques: print, breakpoint(), VS Code debugger

**1. Print debugging**: fast and fine for small things:

```python
order = {"id": 7}
print(f"{order=}")             # -> order={'id': 7}
print(repr("trailing space "))  # repr shows hidden whitespace -> 'trailing space '
```

**2. `breakpoint()`**: pauses and opens the **pdb** debugger in the terminal:

```python
def compute_total(orders):
    breakpoint()          # execution stops here
    return sum(o["amount"] for o in orders)
```

| pdb command | Does |
|-------------|------|
| `p expr` / `pp expr` | print / pretty-print a value |
| `n` | next line |
| `s` | step into a function |
| `c` | continue until next breakpoint |
| `l` | show code around current line |
| `w` | where am I (stack) |
| `u` / `d` | move up/down the stack |
| `q` | quit |

Post-mortem: `python3 -m pdb script.py` drops you into the debugger at the crash.

**3. VS Code debugger** (most comfortable):
1. Click left of a line number to set a red breakpoint.
2. Run → Start Debugging (F5) → "Python File".
3. Inspect variables in the side panel, hover over names, use the Debug Console to run expressions.
4. Step over (F10), step into (F11), continue (F5). Conditional breakpoints: right-click → "Edit breakpoint" → `order["id"] == 7`.

**Debugging method**
1. **Reproduce** reliably (smallest input that fails).
2. **Read** the error and traceback fully.
3. **Form a hypothesis**, check it (print/debugger), don't guess-and-edit randomly.
4. **Fix**, then add a **test** so it never comes back.

---

## 3. logging for debugging

(Set-up details in topic 14.) Logs are how you debug code running at the customer, where you can't attach a debugger.

```python
import logging

logging.basicConfig(level=logging.DEBUG, format="%(asctime)s %(levelname)s %(name)s: %(message)s")
log = logging.getLogger(__name__)

def process(record: dict) -> None:
    log.debug("processing record id=%s keys=%s", record.get("id"), sorted(record))
    if "amount" not in record:
        log.warning("record %s missing amount; skipped", record.get("id"))
        return
    log.info("record %s ok", record["id"])

process({"id": 1, "amount": 10})
process({"id": 2})
```

- **DEBUG**: detailed internals (off in production by default).
- **INFO**: normal milestones ("synced 120 records").
- **WARNING**: something odd but handled.
- **ERROR/exception**: an operation failed (`log.exception` adds the traceback).
- Include **ids and counts** in messages so you can find the exact record later. Never log secrets.

---

## 4. pytest basics

`python3 -m pip install pytest`. Tests are plain functions starting with `test_`, in files named `test_*.py`, using plain `assert`.

```python
# pricing.py
def apply_discount(price: float, percent: float) -> float:
    if not 0 <= percent <= 100:
        raise ValueError("percent must be 0-100")
    return round(price * (1 - percent / 100), 2)
```

```python
# test_pricing.py
import pytest
from pricing import apply_discount

def test_ten_percent_off():
    assert apply_discount(100, 10) == 90

def test_zero_discount_returns_same_price():
    assert apply_discount(59.99, 0) == 59.99

def test_invalid_percent_raises():
    with pytest.raises(ValueError, match="0-100"):
        apply_discount(100, 150)

@pytest.mark.parametrize("price, percent, expected", [
    (100, 50, 50),
    (80, 25, 60),
    (10, 100, 0),
])
def test_many_cases(price, percent, expected):
    assert apply_discount(price, percent) == expected
```

```bash
pytest                  # find and run all tests
pytest -v               # verbose: each test name
pytest -k discount      # only tests matching a name
pytest -x               # stop at first failure
pytest --lf             # re-run only last failures
```

Floats: compare with `pytest.approx`: `assert 0.1 + 0.2 == pytest.approx(0.3)`.

**Fixtures**: reusable setup, injected by parameter name:

```python
import pytest

@pytest.fixture
def sample_orders():
    return [{"id": 1, "amount": 10}, {"id": 2, "amount": 15}]

def test_total(sample_orders):
    assert sum(o["amount"] for o in sample_orders) == 25

def test_writes_report(tmp_path):            # built-in fixture: a temp folder per test
    out = tmp_path / "report.csv"
    out.write_text("id,total\n")
    assert out.read_text().startswith("id")
```

Shared fixtures go in `conftest.py` (pytest finds it automatically).

---

## 5. Writing good test cases

A good test is **small, fast, independent and readable**. Use the **Arrange → Act → Assert** shape:

```python
def test_normalize_email_strips_and_lowercases():
    # Arrange
    raw = "  Jane.Doe@ACME.com "
    # Act
    result = raw.strip().lower()
    # Assert
    assert result == "jane.doe@acme.com"
```

**What to test**
- The **happy path** (normal input).
- **Edge cases:** empty list, `None`, zero, one item, very large values, unicode, duplicate rows.
- **Error cases:** bad input raises the right exception with a helpful message.
- **Every bug you fix**: write the failing test first, then fix (a regression test).

**Habits**
- Test name = the behaviour: `test_skips_rows_without_email`, not `test_1`.
- One behaviour per test; no test depends on another's leftovers.
- Don't test Python or libraries. Test **your** logic.
- Keep pure logic (cleaning, mapping, calculations) separate from I/O so it's easy to test.

---

## 6. Mocking external services

Tests must not call real APIs, send emails or hit production databases. Replace those parts with **fakes/mocks**.

```python
# client.py
import requests

def get_customer_name(customer_id: int) -> str:
    resp = requests.get(f"https://api.example.com/customers/{customer_id}", timeout=10)
    resp.raise_for_status()
    return resp.json()["name"]
```

```python
# test_client.py
from unittest.mock import patch, Mock
import client

def test_get_customer_name():
    fake_resp = Mock()
    fake_resp.json.return_value = {"name": "Acme"}
    fake_resp.raise_for_status.return_value = None

    with patch("client.requests.get", return_value=fake_resp) as fake_get:
        assert client.get_customer_name(42) == "Acme"

    fake_get.assert_called_once()
    assert "/customers/42" in fake_get.call_args.args[0]
```

- **Patch where the name is used** (`client.requests.get`), not where it's defined.
- pytest's `monkeypatch` fixture is great for env vars: `monkeypatch.setenv("API_KEY", "test")`.
- Libraries: `responses` / `respx` (fake HTTP for requests/httpx), `freezegun` (fake "now"), and an in-memory SQLite for DB logic.
- Better design = less mocking: pass the client/session in as a parameter (dependency injection), then pass a fake in tests.

**Coverage**: `pip install pytest-cov` → `pytest --cov=mypackage --cov-report=term-missing` shows untested lines. Aim for meaningful coverage of logic, not 100% for its own sake.

---

## Recap

- Read tracebacks bottom-up; find the first frame in **your** code; inspect the data.
- Debug with `print(f"{x=}")`, `breakpoint()`/pdb, or the VS Code debugger (F5); reproduce → hypothesise → verify → fix → test.
- Log with levels, ids and counts; never secrets.
- pytest: `test_*` functions + `assert`, `pytest.raises`, `parametrize`, fixtures (`tmp_path`, `conftest.py`).
- Test happy path, edge cases, errors, and every fixed bug.
- Mock external services (`unittest.mock.patch`, `monkeypatch`, `responses`/`respx`).

## Practice ideas

1. Write tests (with `parametrize`) for the email normaliser and CSV cleaner from earlier topics.
2. Put a bug in a function on purpose, find it only with the VS Code debugger, then add a regression test.
3. Mock `requests.get` to test your API client for 200, 404 and 500 responses.
4. Run `pytest --cov` on your mini project and cover the most important untested branch.
