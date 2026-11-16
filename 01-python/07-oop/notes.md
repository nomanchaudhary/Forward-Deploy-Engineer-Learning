# 07 — OOP (Object-Oriented Programming): Notes

> **Why this matters for an FDE:** SDKs and API clients you use (and build) are classes: `client = Client(api_key)`, `client.orders.list()`. OOP lets you bundle data + behaviour, e.g. a `SalesforceClient` that holds its token and knows how to call the API.

---

## 1. Classes and objects

- A **class** is a blueprint. An **object** (instance) is one thing built from it.
- Everything in Python is already an object: `"hi".upper()` calls a method on a `str` object.

```python
class Customer:
    pass

acme = Customer()        # create an instance
globex = Customer()
print(type(acme))        # -> <class '__main__.Customer'>
print(acme is globex)    # -> False  (two separate objects)
```

**When to use a class:** when you have **data plus functions that operate on that data** and you'll create several of them, or when you need to keep state (a connection, a token, a counter). If it's just a function, keep it a function.

---

## 2. `__init__` and instance attributes

`__init__` runs when you create an object. `self` is the object being created; attributes stored on `self` belong to that instance.

```python
class Customer:
    country_default = "US"            # class attribute: shared by all instances

    def __init__(self, name, seats, country=None):
        self.name = name              # instance attributes: per object
        self.seats = seats
        self.country = country or self.country_default

acme = Customer("Acme", 250)
print(acme.name, acme.seats, acme.country)   # -> Acme 250 US
acme.seats = 300                              # attributes are changeable
```

⚠️ **Common mistakes**
- Forgetting `self` as the first parameter of a method → `TypeError: takes 0 positional arguments but 1 was given`.
- Writing `name = name` instead of `self.name = name` (just a local variable, lost after `__init__`).
- Using a mutable **class** attribute (`tags = []`) → shared by every instance. Create it in `__init__`.

---

## 3. Methods and properties

A **method** is a function defined in a class. It receives the instance as `self`.

```python
class Account:
    def __init__(self, owner, balance=0.0):
        self.owner = owner
        self.balance = balance

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("amount must be positive")
        self.balance += amount
        return self.balance

acct = Account("Acme")
acct.deposit(100)
print(acct.balance)   # -> 100.0
```

**@property**: a method you access like an attribute (computed values, validation):

```python
class Subscription:
    def __init__(self, seats, price_per_seat):
        self.seats = seats
        self.price_per_seat = price_per_seat

    @property
    def mrr(self):
        return self.seats * self.price_per_seat

sub = Subscription(10, 30)
print(sub.mrr)   # -> 300   (no parentheses)
```

**@classmethod**: an alternative constructor (very common for "build from a dict/JSON"):

```python
class User:
    def __init__(self, id, email):
        self.id = id
        self.email = email

    @classmethod
    def from_api(cls, data: dict):
        return cls(id=data["id"], email=data["email"].lower())

u = User.from_api({"id": 1, "email": "JANE@ACME.COM"})
print(u.email)   # -> jane@acme.com
```

**@staticmethod**: a plain function that lives in the class for organisation (no `self`/`cls`).

**Naming convention:** `_private_ish` (single underscore) means "internal, don't touch from outside". Python doesn't enforce it.

---

## 4. Inheritance

A **child class** reuses and extends a **parent class**.

```python
class Notifier:
    def __init__(self, recipient):
        self.recipient = recipient

    def send(self, message):
        raise NotImplementedError

class EmailNotifier(Notifier):
    def send(self, message):
        return f"Email to {self.recipient}: {message}"

class SlackNotifier(Notifier):
    def __init__(self, recipient, channel):
        super().__init__(recipient)     # run the parent's __init__
        self.channel = channel

    def send(self, message):
        return f"Slack #{self.channel} @{self.recipient}: {message}"

for n in [EmailNotifier("jane@acme.com"), SlackNotifier("raj", "alerts")]:
    print(n.send("Deploy finished"))     # same call, different behaviour (polymorphism)
# -> Email to jane@acme.com: Deploy finished
# -> Slack #alerts @raj: Deploy finished
print(isinstance(SlackNotifier("x", "y"), Notifier))   # -> True
```

**Composition over inheritance:** often it's simpler to *hold* another object than to inherit from it ("a ReportJob **has** a Notifier" rather than "**is** a Notifier"). Use inheritance for true "is-a" relationships, and keep hierarchies shallow (1–2 levels).

**Abstract base classes** force children to implement methods:

```python
from abc import ABC, abstractmethod

class DataSource(ABC):
    @abstractmethod
    def fetch(self) -> list[dict]: ...

# DataSource()  -> TypeError: Can't instantiate abstract class
```

---

## 5. Dunder methods (`__str__`, `__repr__`)

"Dunder" = **d**ouble **under**score. They let your objects work with Python's built-in syntax.

```python
class Money:
    def __init__(self, amount, currency="USD"):
        self.amount = amount
        self.currency = currency

    def __repr__(self):            # for developers: unambiguous, used in the REPL/logs
        return f"Money({self.amount!r}, {self.currency!r})"

    def __str__(self):             # for humans: used by print() and f-strings
        return f"{self.currency} {self.amount:,.2f}"

    def __eq__(self, other):       # ==
        return (self.amount, self.currency) == (other.amount, other.currency)

    def __add__(self, other):      # +
        return Money(self.amount + other.amount, self.currency)

m = Money(1200) + Money(34.5)
print(m)         # -> USD 1,234.50
print(repr(m))   # -> Money(1234.5, 'USD')
print(Money(1) == Money(1))   # -> True
```

Other useful ones: `__len__` (`len(x)`), `__iter__` (`for item in x`), `__contains__` (`in`), `__getitem__` (`x[key]`), `__enter__/__exit__` (`with`, topic 08), `__lt__` (sorting).

Tip: **always write a `__repr__`**. Debugging is much easier when logs show `Order(id=7, status='paid')` instead of `<Order object at 0x7f...>`.

---

## 6. dataclasses

For classes that mainly **hold data**, `@dataclass` writes `__init__`, `__repr__` and `__eq__` for you.

```python
from dataclasses import dataclass, field, asdict

@dataclass
class Order:
    id: int
    customer: str
    total: float = 0.0
    tags: list[str] = field(default_factory=list)   # safe mutable default

    @property
    def is_large(self) -> bool:
        return self.total > 1000

o = Order(7, "Acme", 1500)
print(o)             # -> Order(id=7, customer='Acme', total=1500, tags=[])
print(o.is_large)    # -> True
print(asdict(o))     # -> {'id': 7, 'customer': 'Acme', 'total': 1500, 'tags': []}
```

Options:
- `@dataclass(frozen=True)`: immutable (and hashable, so it can go in a set or be a dict key).
- `@dataclass(order=True)`: adds `<`, `>` so instances can be sorted.

When you also need **validation** of incoming data (e.g. from an API), use **pydantic** (topic 09).

---

## Recap

- Class = blueprint; object = instance; `self` = the current instance.
- `__init__` sets instance attributes; put mutable defaults there, never on the class.
- `@property` for computed attributes; `@classmethod` for alternative constructors like `from_api`.
- Inherit for "is-a", compose for "has-a"; call `super().__init__()`.
- Implement `__repr__` and `__str__`; use `@dataclass` for data containers.

## Practice ideas

1. `BankAccount` with deposit/withdraw that raises on overdraft, plus a `__repr__`.
2. A `Shape` base class with `area()`, and `Circle`/`Rectangle` children; sort a list of shapes by area.
3. A `@dataclass Ticket` with `from_dict()` classmethod; load 3 tickets from a list of dicts.
4. A tiny `APIClient` class that stores `base_url` and `token` and has a `build_url(path)` method.
