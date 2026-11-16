# 04 — Functions: Notes

> **Why this matters for an FDE:** functions turn a one-off script into reusable, testable pieces like `fetch_orders()`, `clean_row()` and `send_report()`. Customers inherit your code; small named functions make it understandable.

---

## 1. Defining and calling functions

```python
def greet(name):
    """Return a greeting for a customer."""
    return f"Hello, {name}!"

message = greet("Acme")
print(message)   # -> Hello, Acme!
```

- `def` + name + parameters + `:` + indented body.
- **Parameters** are the names in the definition; **arguments** are the values you pass in.
- A function does nothing until you **call** it with `()`.

**Good function habits**
- One job per function. If the name needs "and", split it.
- Name it with a verb: `load_csv`, `normalize_email`, `is_valid`.
- Keep it short enough to read without scrolling.

---

## 2. Positional, keyword, default args

```python
def create_user(name, role="viewer", active=True):
    return {"name": name, "role": role, "active": active}

print(create_user("Jane"))                    # positional, defaults used
# -> {'name': 'Jane', 'role': 'viewer', 'active': True}
print(create_user("Raj", "admin"))            # positional
print(create_user("Li", active=False))        # keyword: skip 'role'
```

- **Positional:** matched by order.
- **Keyword:** matched by name (clearer, order doesn't matter).
- **Default:** used when the caller doesn't pass it. Defaults must come after non-defaults.

**Force keyword-only** with a bare `*` (makes calls self-documenting):

```python
def transfer(amount, *, dry_run):
    return f"{'would move' if dry_run else 'moved'} {amount}"

print(transfer(100, dry_run=True))   # -> would move 100
# transfer(100, True)  -> TypeError: must name it
```

⚠️ **The mutable default trap** (a classic interview question):

```python
def add_tag(tag, tags=[]):        # BAD: the same list is reused every call
    tags.append(tag)
    return tags

print(add_tag("a"))   # -> ['a']
print(add_tag("b"))   # -> ['a', 'b']   ← leaked from last call!

def add_tag_fixed(tag, tags=None):  # GOOD
    if tags is None:
        tags = []
    tags.append(tag)
    return tags
```

---

## 3. *args and **kwargs

- `*args` collects extra **positional** args into a **tuple**.
- `**kwargs` collects extra **keyword** args into a **dict**.

```python
def total(*amounts):
    return sum(amounts)

print(total(10, 20, 30))   # -> 60

def log_event(event, **fields):
    details = ", ".join(f"{k}={v}" for k, v in fields.items())
    print(f"[{event}] {details}")

log_event("login", user="jane", ip="10.0.0.1")
# -> [login] user=jane, ip=10.0.0.1
```

**Unpacking when calling** (the reverse direction):

```python
nums = [1, 2, 3]
print(total(*nums))              # -> 6

params = {"user": "raj", "ip": "1.1.1.1"}
log_event("logout", **params)    # -> [logout] user=raj, ip=1.1.1.1
```

🔧 **FDE use:** wrapping a library call and passing options through: `def get(url, **kwargs): return requests.get(url, timeout=10, **kwargs)`.

---

## 4. Return values

- `return` sends a value back and **ends** the function immediately.
- No `return` (or bare `return`) → the function returns `None`.
- Return several values as a tuple.

```python
def parse_name(full):
    first, _, last = full.partition(" ")
    return first, last

first, last = parse_name("Jane Doe")
print(first, last)   # -> Jane Doe
```

**Early return** keeps code flat (a "guard clause"):

```python
def discount(price, code):
    if not code:
        return price            # nothing to do, leave early
    if code == "VIP":
        return price * 0.8
    return price

print(discount(100, "VIP"))   # -> 80.0
```

⚠️ **Common mistake:** using `print()` inside a function when you meant `return`. `print` shows a value; `return` lets the caller *use* it.

---

## 5. Scope

**Scope** = where a name is visible. Python looks names up in this order (**LEGB**):

1. **L**ocal: inside the current function
2. **E**nclosing: inside outer functions (for nested functions)
3. **G**lobal: top level of the file
4. **B**uilt-in: `print`, `len`, ...

```python
rate = 0.1              # global

def apply_tax(amount):
    tax = amount * rate  # 'tax' is local; 'rate' is read from global
    return amount + tax

print(apply_tax(100))   # -> 110.0
# print(tax)            # NameError: 'tax' only exists inside the function
```

Assigning to a name inside a function makes it **local** unless you say `global`. Avoid `global`; pass values in and return results instead.

**Closures**: an inner function remembers variables from its enclosing function:

```python
def make_multiplier(factor):
    def multiply(x):
        return x * factor
    return multiply

double = make_multiplier(2)
print(double(21))   # -> 42
```

---

## 6. Lambdas, map, filter, sorted(key=)

A **lambda** is a tiny anonymous function for one expression:

```python
square = lambda x: x * x
print(square(5))   # -> 25
```

Mostly used as a **key** for sorting/min/max:

```python
customers = [
    {"name": "Acme", "mrr": 1200},
    {"name": "Globex", "mrr": 300},
    {"name": "Initech", "mrr": 900},
]

top = sorted(customers, key=lambda c: c["mrr"], reverse=True)
print([c["name"] for c in top])   # -> ['Acme', 'Initech', 'Globex']

biggest = max(customers, key=lambda c: c["mrr"])
print(biggest["name"])            # -> Acme
```

**map / filter** apply a function to each item (comprehensions are usually clearer):

```python
names = ["  acme ", "globex"]
print(list(map(str.strip, names)))                 # -> ['acme', 'globex']
print(list(filter(lambda c: c["mrr"] > 500, customers)))  # big customers
# same as: [c for c in customers if c["mrr"] > 500]
```

Functions are **objects**: you can pass them around, store them in dicts, return them:

```python
handlers = {"csv": lambda p: f"reading CSV {p}", "json": lambda p: f"reading JSON {p}"}
print(handlers["csv"]("data.csv"))   # -> reading CSV data.csv
```

---

## 7. Type hints and docstrings

**Type hints** say what types a function expects and returns. Python does **not** enforce them at runtime, but editors and tools (`mypy`, `pyright`) use them to catch bugs.

```python
def average(values: list[float]) -> float:
    """Return the mean of `values`.

    Args:
        values: a non-empty list of numbers.

    Returns:
        The arithmetic mean.

    Raises:
        ValueError: if `values` is empty.
    """
    if not values:
        raise ValueError("values must not be empty")
    return sum(values) / len(values)

print(average([1, 2, 3]))   # -> 2.0
help(average)               # prints the docstring
```

Common hints: `int`, `str`, `list[str]`, `dict[str, int]`, `str | None` (value may be None), `-> None` (returns nothing). More in topic 09.

**Docstring** = the triple-quoted string on the first line. Say *what* the function does, its arguments, return value, and errors.

---

## Recap

- `def name(params): ... return value`. Call with `name(args)`.
- Defaults go last; never use a mutable default (`[]`, `{}`). Use `None`.
- `*args` → tuple, `**kwargs` → dict; `*` / `**` also unpack when calling.
- Names inside a function are local (LEGB rule).
- `sorted(items, key=lambda x: ...)` is the everyday lambda use.
- Add type hints + a docstring to every function you hand to a customer.

## Practice ideas

1. `normalize_email(email: str) -> str` that strips and lower-cases.
2. `summarize(*amounts, currency="USD")` returning `"3 payments, total USD 60.00"`.
3. Sort a list of tickets by priority then by created date (`key=lambda t: (t["priority"], t["created"])`).
4. Write `make_counter()` that returns a function counting how many times it's called.
