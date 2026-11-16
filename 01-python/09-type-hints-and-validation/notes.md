# 09 — Type Hints & Data Validation: Notes

> **Why this matters for an FDE:** customer data and third-party APIs lie. Fields go missing, numbers arrive as strings, dates come in five formats. Type hints catch your own bugs before running; **pydantic** catches *their* bad data at the boundary with clear error messages. Modern AI/LLM tooling (FastAPI, tool calling, structured outputs) is built on pydantic too.

---

## 1. Type hints basics

Type hints describe what types variables, parameters and return values should be. Python **ignores them at runtime**; they're for you, your editor, and type checkers.

```python
name: str = "Acme"
seats: int = 250
ratio: float = 0.75
active: bool = True

def price(seats: int, per_seat: float = 30.0) -> float:
    return seats * per_seat

print(price(10))   # -> 300.0
print(price("10", 2))  # -> 1010  runs, but wrong! ("10" * 2). A type checker flags this.
```

**Collections** (Python 3.9+ uses the built-in names):

```python
ids: list[int] = [1, 2, 3]
prices: dict[str, float] = {"pro": 30.0}
point: tuple[int, int] = (1, 2)
row: tuple[str, ...] = ("a", "b", "c")   # any length
unique: set[str] = {"a"}
```

**Optional and unions:**

```python
def find_email(user_id: int) -> str | None:   # may return None
    return None

value: int | str = 5          # either type
```

`-> None` means "returns nothing".

---

## 2. Typing module essentials

```python
from typing import Any, Literal, TypedDict, Callable, Iterator
from collections.abc import Iterable

Plan = Literal["starter", "pro", "enterprise"]   # only these exact values

def set_plan(plan: Plan) -> None: ...

class UserDict(TypedDict):          # the shape of a dict (e.g. JSON from an API)
    id: int
    email: str

def first_email(users: list[UserDict]) -> str:
    return users[0]["email"]

def apply(func: Callable[[int], int], value: int) -> int:   # a function as argument
    return func(value)

def numbers() -> Iterator[int]:     # generators
    yield 1

def total(values: Iterable[float]) -> float:   # accept any iterable, not just list
    return sum(values)

Headers = dict[str, str]            # type alias for readability
anything: Any = 42                  # escape hatch: no checking (use sparingly)
```

Tips:
- Accept **general** types in parameters (`Iterable`, `Mapping`), return **specific** ones (`list`, `dict`).
- `Any` turns checking off. Prefer `object` or a real type when you can.

---

## 3. Static type checking (mypy / pyright)

A **type checker** reads your code without running it and reports mismatches.

```bash
python3 -m pip install mypy
mypy my_script.py
```

```python
# bad.py
def double(x: int) -> int:
    return x * 2

double("5")    # mypy: Argument 1 has incompatible type "str"; expected "int"
```

- VS Code's Pylance uses **pyright** and shows these as red squiggles live. Turn on "basic" type checking in settings.
- Add hints gradually: start with function signatures; the checker infers most locals.

🔧 **FDE use:** when you hand a codebase to a customer team, hints are living documentation: they can see what `sync_accounts(...)` expects without reading its body.

---

## 4. Pydantic models

**pydantic** validates and converts data at runtime using type hints. Install: `python3 -m pip install pydantic`.

```python
from datetime import datetime
from pydantic import BaseModel, Field, ValidationError

class Customer(BaseModel):
    id: int
    name: str
    email: str
    seats: int = Field(ge=1, le=10_000)    # constraints
    plan: str = "starter"                  # default
    created_at: datetime

raw = {
    "id": "42",                             # string → converted to int
    "name": "Acme",
    "email": "ops@acme.com",
    "seats": 250,
    "created_at": "2026-11-17T10:00:00Z",  # ISO string → datetime
}

c = Customer(**raw)          # or Customer.model_validate(raw)
print(c.id, type(c.id))      # -> 42 <class 'int'>
print(c.created_at.year)     # -> 2026
print(c.model_dump())        # back to a dict
print(c.model_dump_json())   # straight to a JSON string
```

**Clear errors on bad data:**

```python
try:
    Customer(id="abc", name="X", email="x@y.com", seats=0, created_at="yesterday")
except ValidationError as e:
    print(e.error_count(), "errors")   # -> 3 errors
    for err in e.errors():
        print(err["loc"], err["msg"])
# -> ('id',) Input should be a valid integer, unable to parse string as an integer
# -> ('seats',) Input should be greater than or equal to 1
# -> ('created_at',) Input should be a valid datetime ...
```

**Nested models and lists**, which match nested JSON:

```python
class LineItem(BaseModel):
    sku: str
    qty: int
    price: float

class Order(BaseModel):
    id: int
    items: list[LineItem]

    @property
    def total(self) -> float:
        return sum(i.qty * i.price for i in self.items)

order = Order.model_validate({"id": 1, "items": [{"sku": "A", "qty": 2, "price": 9.5}]})
print(order.total)   # -> 19.0
```

Pydantic vs dataclass: **dataclass** = a lightweight container for data you already trust. **Pydantic** = data from outside (APIs, files, users, LLM output) that must be checked and converted.

---

## 5. Custom validators and settings

**Field validators**: your own cleaning/validation rules:

```python
from pydantic import BaseModel, field_validator

class Contact(BaseModel):
    email: str
    country: str

    @field_validator("email")
    @classmethod
    def normalize_email(cls, v: str) -> str:
        v = v.strip().lower()
        if "@" not in v:
            raise ValueError("not an email address")
        return v

    @field_validator("country")
    @classmethod
    def upper_country(cls, v: str) -> str:
        return v.strip().upper()

print(Contact(email="  JANE@Acme.com ", country="pk"))
# -> email='jane@acme.com' country='PK'
```

**Settings from environment variables** with `pydantic-settings` (`pip install pydantic-settings`): typed config with defaults, read from env / `.env`:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")
    api_base_url: str = "https://api.example.com"
    api_key: str                 # required: fails fast if missing
    timeout_seconds: float = 10.0

# settings = Settings()   # reads API_KEY, TIMEOUT_SECONDS ... from env
```

🔧 **FDE use:** validate every payload at the edge of your system (incoming webhook, CSV row, API response, LLM JSON output). Invalid data fails loudly with the exact field and reason, instead of a mysterious `KeyError` three functions later.

---

## Recap

- Hints (`x: int`, `-> str | None`, `list[dict[str, int]]`) document intent; Python doesn't enforce them.
- `Literal`, `TypedDict`, `Callable`, `Iterable` cover most real needs.
- Run `mypy` / enable Pylance type checking to catch bugs before runtime.
- `pydantic.BaseModel` validates + converts external data; `ValidationError` tells you exactly what's wrong.
- `field_validator` for custom rules; `pydantic-settings` for typed config from env vars.

## Practice ideas

1. Add hints to every function from topic 04 and run `mypy` until it's clean.
2. Model a webhook payload (`event`, `timestamp`, nested `data`) in pydantic and feed it good and bad samples.
3. Validate each row of a CSV with a pydantic model; write valid rows to one file and errors to another.
4. Build a `Settings` class for an API client and load it from a `.env` file.
