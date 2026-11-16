# 02 — Control Flow: Notes

> **Why this matters for an FDE:** "for every row in the customer's file, if it's broken, fix it, else skip it" is control flow. You'll write this pattern every week.

---

## 1. if / elif / else

Run code only when a condition is true. **Indentation (4 spaces) defines the block.**

```python
seats = 120

if seats >= 500:
    tier = "enterprise"
elif seats >= 50:
    tier = "business"
else:
    tier = "starter"

print(tier)   # -> business
```

- Python checks conditions **top to bottom** and runs only the **first** match.
- `elif` and `else` are optional.

**One-line conditional (ternary):**

```python
status = "active" if seats > 0 else "inactive"
```

**Membership tests read nicely:**

```python
country = "PK"
if country in ("US", "CA", "PK"):
    print("supported")   # -> supported
```

**match / case** (Python 3.10+): useful when branching on the shape of a value:

```python
def handle(event):
    match event:
        case {"type": "created", "id": id_}:
            return f"new record {id_}"
        case {"type": "deleted"}:
            return "removed"
        case _:
            return "unknown"

print(handle({"type": "created", "id": 7}))  # -> new record 7
```

⚠️ **Common mistakes**
- Forgetting the `:` at the end of `if ...:`.
- Mixing tabs and spaces → `IndentationError`. Configure your editor to insert 4 spaces.
- Order matters: put the most specific condition first.

---

## 2. for loops and range()

A `for` loop goes over **each item** of anything iterable (list, string, dict, file, ...).

```python
customers = ["Acme", "Globex", "Initech"]
for name in customers:
    print(name)
# -> Acme
# -> Globex
# -> Initech
```

**range()** gives numbers: `range(stop)`, `range(start, stop)`, `range(start, stop, step)`. `stop` is excluded.

```python
print(list(range(5)))         # -> [0, 1, 2, 3, 4]
print(list(range(2, 10, 3)))  # -> [2, 5, 8]
```

**enumerate()**: get the index *and* the item (don't use `range(len(...))`):

```python
for i, name in enumerate(customers, start=1):
    print(i, name)
# -> 1 Acme
# -> 2 Globex
# -> 3 Initech
```

**zip()**: walk two lists side by side:

```python
names = ["Acme", "Globex"]
mrr = [1200, 800]
for name, amount in zip(names, mrr):
    print(f"{name}: ${amount}")
# -> Acme: $1200
# -> Globex: $800
```

**Looping over a dict:**

```python
plan_prices = {"starter": 10, "pro": 30}
for plan, price in plan_prices.items():
    print(plan, price)
```

⚠️ **Common mistake:** changing a list while looping over it (adding/removing items) skips elements. Loop over a copy (`for x in items[:]`) or build a new list.

---

## 3. while loops

Repeat **while** a condition is true. Use when you don't know in advance how many times.

```python
attempt = 1
max_attempts = 3
while attempt <= max_attempts:
    print(f"try #{attempt}")
    attempt += 1
# -> try #1
# -> try #2
# -> try #3
```

Classic FDE pattern: **retry until success or give up**:

```python
import random

random.seed(1)
attempt = 0
success = False
while not success and attempt < 5:
    attempt += 1
    success = random.random() > 0.7   # pretend this is an API call
print(f"done after {attempt} attempts, success={success}")
```

⚠️ **Common mistake:** forgetting to update the condition variable → **infinite loop**. Stop it with `Ctrl+C`.

---

## 4. break, continue, pass

- `break`: exit the loop immediately.
- `continue`: skip the rest of this iteration, go to the next.
- `pass`: do nothing (a placeholder where Python needs a statement).

```python
rows = ["ok", "", "ok", "STOP", "ok"]
for row in rows:
    if row == "":
        continue          # skip blank rows
    if row == "STOP":
        break             # stop processing entirely
    print("processing", row)
# -> processing ok
# -> processing ok
```

```python
def not_done_yet():
    pass   # TODO: implement later
```

**for ... else**: the `else` runs only if the loop did **not** `break`. Handy for "search and report not found":

```python
for name in ["Acme", "Globex"]:
    if name == "Initech":
        print("found")
        break
else:
    print("not found")   # -> not found
```

---

## 5. List / dict / set comprehensions

A short way to **build a collection from a loop**. Read them as: "*expression* for *item* in *iterable* if *condition*".

```python
nums = [1, 2, 3, 4, 5, 6]

squares = [n * n for n in nums]
print(squares)            # -> [1, 4, 9, 16, 25, 36]

evens = [n for n in nums if n % 2 == 0]
print(evens)              # -> [2, 4, 6]
```

Same thing with a normal loop (comprehensions are just shorter):

```python
evens = []
for n in nums:
    if n % 2 == 0:
        evens.append(n)
```

**Dict comprehension:**

```python
emails = ["A@x.com", "b@Y.com"]
domains = {e.lower(): e.split("@")[1].lower() for e in emails}
print(domains)   # -> {'a@x.com': 'x.com', 'b@y.com': 'y.com'}
```

**Set comprehension** (unique values):

```python
unique_domains = {e.split("@")[1].lower() for e in ["a@x.com", "b@X.com"]}
print(unique_domains)   # -> {'x.com'}
```

**if/else inside the expression** (goes *before* `for`):

```python
labels = ["even" if n % 2 == 0 else "odd" for n in [1, 2, 3]]
print(labels)   # -> ['odd', 'even', 'odd']
```

⚠️ **Common mistake:** writing a huge nested comprehension nobody can read. If it doesn't fit on one or two lines, use a normal loop.

🔧 **FDE use:** `clean = [r.strip().lower() for r in rows if r.strip()]` cleans a whole column in one readable line.

---

## Recap

- `if/elif/else`: first true branch wins; indentation is the block.
- `for` loops over items; use `enumerate()` for indexes and `zip()` to pair lists.
- `while` repeats until a condition changes; always make sure it can end.
- `break` exits, `continue` skips, `pass` does nothing.
- Comprehensions build lists/dicts/sets from a loop in one line.

## Practice ideas

1. FizzBuzz from 1 to 30.
2. Given a list of order amounts, print only the ones over 100 and the total of those.
3. Write a "retry" loop that tries up to 5 times and prints which attempt succeeded.
4. Turn `["  Alice ", "", "BOB", "  "]` into `["alice", "bob"]` with one comprehension.
