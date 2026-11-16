# 01 — Basics: Notes

> **Why this matters for an FDE:** every script you write for a customer (a data fix, an API call, a quick report) is built from these pieces. Get them solid and everything later is easier.

---

## 1. Install Python & run a script

Python is an **interpreter**: it reads your `.py` file top to bottom and runs each line.

```bash
python3 --version              # check it's installed (3.11+ is good)
python3 hello.py               # run a file
python3                        # open the interactive REPL (exit with Ctrl+D)
```

A script is just a text file:

```python
# hello.py
print("Hello, FDE!")
```

```text
$ python3 hello.py
Hello, FDE!
```

**The REPL** (`>>>` prompt) is great for testing one line quickly. Scripts are for anything you want to keep.

⚠️ **Common mistakes**
- Running `python` instead of `python3`: on many Linux/Mac machines `python` is missing or is Python 2.
- Running from the wrong folder: `python3 hello.py` only works if `hello.py` is in your current directory (`pwd` to check).

🔧 **FDE use:** you'll often SSH into a customer box and the first thing to check is `python3 --version`.

---

## 2. Variables and naming

A **variable** is a name that points to a value. You create one with `=`.

```python
customer_name = "Acme Corp"
seats = 250
is_active = True

seats = seats + 10   # re-assign: the name now points to a new value
print(seats)         # -> 260
```

**Naming rules**
- Letters, digits, `_`; cannot start with a digit; case-sensitive (`Name` ≠ `name`).
- Python style (PEP 8): `snake_case` for variables and functions, `UPPER_CASE` for constants, `PascalCase` for classes.

```python
MAX_RETRIES = 3          # constant (by convention: Python won't stop you changing it)
api_base_url = "https://api.example.com"
```

Pick names that explain *what* the value is: `invoice_total` beats `x`.

⚠️ **Common mistakes**
- Shadowing built-ins: `list = [1, 2]` or `str = "a"` breaks `list()` / `str()` later.
- Using a variable before assigning it → `NameError`.

---

## 3. Data types: int, float, str, bool, None

Every value has a **type**. Check it with `type()`.

| Type | Example | Use for |
|------|---------|---------|
| `int` | `42`, `-7`, `1_000_000` | counts, IDs |
| `float` | `3.14`, `0.1` | measurements, ratios |
| `str` | `"hello"`, `'hi'` | text |
| `bool` | `True`, `False` | yes/no flags |
| `NoneType` | `None` | "no value yet / missing" |

```python
print(type(42))        # -> <class 'int'>
print(type(3.5))       # -> <class 'float'>
print(type("hi"))      # -> <class 'str'>
print(type(None))      # -> <class 'NoneType'>
```

**Floats are approximate:**

```python
print(0.1 + 0.2)          # -> 0.30000000000000004
print(round(0.1 + 0.2, 2))  # -> 0.3
```

For money, use `decimal.Decimal` (or store cents as `int`).

**Truthiness:** in an `if`, these count as `False`: `0`, `0.0`, `""`, `None`, `[]`, `{}`, `set()`. Everything else is `True`.

**Checking for None:** always use `is`:

```python
email = None
if email is None:
    print("missing email")   # -> missing email
```

---

## 4. print() and input()

`print()` writes to the screen. It accepts many values and a few options:

```python
print("a", "b", "c")             # -> a b c
print("a", "b", sep="-")         # -> a-b
print("no newline", end="")      # stays on the same line
print()                          # just a newline
```

`input()` reads a line from the user. **It always returns a `str`.**

```python
age_text = input("Age? ")    # user types 30
age = int(age_text)          # convert before doing math
print(age + 1)               # -> 31
```

⚠️ **Common mistake:** `input("Age? ") + 1` → `TypeError` (you can't add a `str` and an `int`).

🔧 **FDE use:** real scripts rarely use `input()`. They take values from command-line arguments or environment variables (topic 14). But it's perfect while learning.

---

## 5. Arithmetic, comparison, logical operators

**Arithmetic**

```python
print(7 + 2)    # -> 9
print(7 - 2)    # -> 5
print(7 * 2)    # -> 14
print(7 / 2)    # -> 3.5   (true division: always a float)
print(7 // 2)   # -> 3     (floor division)
print(7 % 2)    # -> 1     (remainder: great for "every Nth item")
print(2 ** 10)  # -> 1024  (power)
```

Shortcuts: `x += 1`, `x -= 1`, `x *= 2`, ...

**Comparison** → returns a `bool`

```python
print(5 == 5, 5 != 3, 5 > 3, 5 <= 5)   # -> True True True True
print(1 < 5 < 10)                      # -> True  (chaining works!)
```

**Logical:** `and`, `or`, `not`

```python
plan = "enterprise"
seats = 600
print(plan == "enterprise" and seats > 500)  # -> True
print(not True)                              # -> False
```

`or` is often used for defaults:

```python
region = "" or "us-east-1"
print(region)   # -> us-east-1
```

⚠️ **Common mistakes**
- `=` (assign) vs `==` (compare).
- `if x == 1 or 2:` is always True. Write `if x == 1 or x == 2:` or `if x in (1, 2):`.

---

## 6. String methods and f-strings

Strings are **immutable** sequences of characters. Methods return a *new* string.

```python
s = "  Acme Corp  "
print(s.strip())           # -> Acme Corp   (remove surrounding whitespace)
print(repr(s.lower()))     # -> '  acme corp  '
print(repr(s.upper()))     # -> '  ACME CORP  '
print(s.strip().replace("Corp", "Inc"))  # -> Acme Inc

email = "jane.doe@acme.com"
print(email.split("@"))        # -> ['jane.doe', 'acme.com']
print(email.endswith(".com"))  # -> True
print("@" in email)            # -> True
print(email.find("@"))         # -> 8   (index, or -1 if missing)
print(",".join(["a", "b", "c"]))  # -> a,b,c
```

**Indexing and slicing** (start at 0, end is exclusive):

```python
word = "Python"
print(word[0])     # -> P
print(word[-1])    # -> n   (negative = from the end)
print(word[0:3])   # -> Pyt
print(word[::-1])  # -> nohtyP  (reversed)
print(len(word))   # -> 6
```

**f-strings**: the modern way to build strings:

```python
name, total = "Acme", 1234.5
print(f"{name} owes ${total:,.2f}")   # -> Acme owes $1,234.50
print(f"{0.4567:.1%}")                # -> 45.7%
print(f"{'id':<6}|{'name':>8}|")      # -> id    |    name|
print(f"{total=}")                    # -> total=1234.5  (handy for debugging)
```

Multi-line strings use triple quotes: `"""..."""`.

⚠️ **Common mistake:** `s.strip()` alone does nothing visible. Strings are immutable, so you must save the result: `s = s.strip()`.

🔧 **FDE use:** cleaning customer data is mostly `strip()`, `lower()`, `split()`, `replace()`.

---

## 7. Type conversion

Convert between types with the type's name:

```python
print(int("42"))        # -> 42
print(float("3.5"))     # -> 3.5
print(repr(str(99)))    # -> '99'
print(int(3.9))         # -> 3     (truncates, does NOT round)
print(round(3.9))       # -> 4
print(bool(""))         # -> False
print(bool("False"))    # -> True  (non-empty string!)
```

Bad input raises `ValueError`:

```python
try:
    int("12abc")
except ValueError as e:
    print("bad number:", e)   # -> bad number: invalid literal for int() with base 10: '12abc'
```

(We cover `try/except` properly in topic 05.)

⚠️ **Common mistakes**
- `int("3.5")` fails. Do `int(float("3.5"))`.
- `bool("False")` is `True`. To parse yes/no text, compare strings: `text.lower() in ("true", "yes", "1")`.

---

## Recap

- Run scripts with `python3 file.py`; experiment in the REPL.
- Variables are names → values; use `snake_case`.
- Core types: `int`, `float`, `str`, `bool`, `None`. Check with `type()`, test None with `is None`.
- `input()` returns `str`; convert before math.
- `/` gives float, `//` floor, `%` remainder, `**` power.
- Strings are immutable; use methods + f-strings (`f"{x:,.2f}"`).

## Practice ideas

1. Ask for a name and birth year; print the age in an f-string.
2. Given `" JANE.Doe@Acme.COM "`, print a cleaned, lower-case email and its domain.
3. Convert a list of price strings `["10.5", "3", "7.25"]` into a total, formatted as money.
