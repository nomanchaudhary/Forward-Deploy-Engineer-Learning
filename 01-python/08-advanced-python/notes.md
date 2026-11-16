# 08 — Advanced (Pythonic) Python: Notes

> **Why this matters for an FDE:** these features show up in every real codebase and library you'll integrate with. Retries are decorators, DB sessions are context managers, and streaming a 10 GB export uses generators. Knowing them lets you read customer and SDK code fluently.

---

## 1. Iterables and iterators

- An **iterable** is anything you can loop over (`list`, `str`, `dict`, file, `range`).
- An **iterator** is the object that hands out items **one at a time** with `next()`, and remembers where it is.

```python
nums = [10, 20, 30]
it = iter(nums)        # get an iterator from an iterable
print(next(it))        # -> 10
print(next(it))        # -> 20
print(next(it))        # -> 30
# next(it)             # -> StopIteration (the loop's signal to stop)
```

A `for` loop is just `iter()` + repeated `next()` until `StopIteration`.

**Iterators are single-use:**

```python
squares = map(lambda x: x * x, [1, 2, 3])
print(list(squares))   # -> [1, 4, 9]
print(list(squares))   # -> []   (already consumed!)
```

**itertools**: the toolbox for iterators:

```python
from itertools import islice, chain, groupby, batched  # batched: Python 3.12+

print(list(islice(range(100), 3)))            # -> [0, 1, 2]   (first N)
print(list(chain([1, 2], [3])))               # -> [1, 2, 3]
print(list(batched(range(7), 3)))             # -> [(0, 1, 2), (3, 4, 5), (6,)]

data = sorted(["apple", "avocado", "banana"], key=lambda w: w[0])
for letter, words in groupby(data, key=lambda w: w[0]):
    print(letter, list(words))
# -> a ['apple', 'avocado']
# -> b ['banana']
```

🔧 **FDE use:** `batched(records, 100)` → send records to an API in chunks of 100 (many APIs limit batch size).

---

## 2. Generators and yield

A **generator function** uses `yield` instead of `return`. It produces values **lazily**: one at a time, only when asked. Memory use stays tiny no matter how much data.

```python
def count_up_to(n):
    i = 1
    while i <= n:
        yield i          # pause here, hand out i, resume on next()
        i += 1

for x in count_up_to(3):
    print(x)
# -> 1
# -> 2
# -> 3
```

**Streaming a huge file** (classic use):

```python
def read_errors(path):
    with open(path, encoding="utf-8") as f:
        for line in f:
            if "ERROR" in line:
                yield line.rstrip()

# for err in read_errors("app.log"): print(err)   # never loads the whole file
```

**Generator pipelines**: chain steps, each lazy:

```python
def parse(lines):
    for line in lines:
        yield line.split(",")

def only_paid(rows):
    for row in rows:
        if row[1] == "paid":
            yield row

lines = ["1,paid", "2,refunded", "3,paid"]
print([r[0] for r in only_paid(parse(lines))])   # -> ['1', '3']
```

**Generator expressions**: like list comprehensions but with `()`:

```python
total = sum(x * x for x in range(1_000_000))   # no million-item list created
```

**Paginated API as a generator**: the caller just loops, unaware of pages:

```python
def fetch_all(fetch_page):
    page = 1
    while True:
        items = fetch_page(page)
        if not items:
            return
        yield from items        # yield each item from the list
        page += 1

fake_pages = {1: ["a", "b"], 2: ["c"]}
print(list(fetch_all(lambda p: fake_pages.get(p, []))))   # -> ['a', 'b', 'c']
```

---

## 3. Decorators

A **decorator** is a function that **wraps another function** to add behaviour before/after it runs, without changing its code. The `@decorator` line is shorthand for `func = decorator(func)`.

```python
import functools
import time

def timed(func):
    @functools.wraps(func)            # keep the original name/docstring
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.3f}s")
        return result
    return wrapper

@timed
def slow_add(a, b):
    time.sleep(0.1)
    return a + b

print(slow_add(2, 3))
# -> slow_add took 0.100s
# -> 5
```

**Decorator with arguments**: one more level of nesting. A real-world **retry** decorator:

```python
import functools
import time

def retry(times=3, delay=0.1, exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == times:
                        raise
                    print(f"attempt {attempt} failed: {e}; retrying")
                    time.sleep(delay * 2 ** (attempt - 1))   # exponential backoff
        return wrapper
    return decorator

calls = {"n": 0}

@retry(times=3, exceptions=(ConnectionError,))
def flaky():
    calls["n"] += 1
    if calls["n"] < 3:
        raise ConnectionError("network blip")
    return "ok"

print(flaky())
# -> attempt 1 failed: network blip; retrying
# -> attempt 2 failed: network blip; retrying
# -> ok
```

**Built-in decorators you'll see:** `@property`, `@classmethod`, `@staticmethod`, `@dataclass`, `@functools.cache` / `@functools.lru_cache` (memoise results), and framework ones like FastAPI's `@app.get("/")` or pytest's `@pytest.fixture`.

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

print(fib(80))   # -> 23416728348467685 (instant thanks to caching)
```

---

## 4. Context managers

A **context manager** is what `with` uses: set up → run your block → **always** clean up (even on error).

**Class-based**: implement `__enter__` and `__exit__`:

```python
import time

class Timer:
    def __enter__(self):
        self.start = time.perf_counter()
        return self                  # value bound by "as"

    def __exit__(self, exc_type, exc, tb):
        self.elapsed = time.perf_counter() - self.start
        print(f"block took {self.elapsed:.3f}s")
        return False                 # False = don't swallow exceptions

with Timer():
    sum(range(1_000_000))
```

**Generator-based** with `contextlib`: shorter, and the usual choice:

```python
from contextlib import contextmanager
import os

@contextmanager
def temp_env(key, value):
    old = os.environ.get(key)
    os.environ[key] = value
    try:
        yield                        # the with-block runs here
    finally:                         # cleanup ALWAYS runs
        if old is None:
            del os.environ[key]
        else:
            os.environ[key] = old

with temp_env("MODE", "test"):
    print(os.environ["MODE"])        # -> test
print(os.environ.get("MODE"))        # -> None (restored)
```

**Useful stdlib ones:** `open()`, `contextlib.suppress(FileNotFoundError)`, `tempfile.TemporaryDirectory()`, `threading.Lock()`, DB connections, `requests.Session()`.

---

## 5. Advanced comprehensions & unpacking

**Nested comprehension** (flatten a list of lists). Read the `for`s left to right, like nested loops:

```python
pages = [[1, 2], [3], [4, 5]]
flat = [item for page in pages for item in page]
print(flat)   # -> [1, 2, 3, 4, 5]
```

**Walrus operator `:=`**: assign inside an expression (3.8+):

```python
raw = ["  a ", "", " b"]
clean = [s for r in raw if (s := r.strip())]
print(clean)   # -> ['a', 'b']
```

**Star-unpacking anywhere:**

```python
head, *middle, tail = [1, 2, 3, 4, 5]
print(head, middle, tail)    # -> 1 [2, 3, 4] 5
merged = [*[1, 2], *[3, 4]]  # -> [1, 2, 3, 4]
```

**Useful built-ins:** `any()`, `all()`, `zip(strict=True)` (errors if lengths differ), `sorted(..., key=)`, `enumerate()`, `reversed()`, `min/max(key=)`.

---

## 6. Strings in depth & regular expressions

Text handling is half of data work, so it's worth going deeper than topic 01.

**Encoding:** `str` is text (Unicode); `bytes` is raw data. Convert with `.encode()` / `.decode()`:

```python
b = "café".encode("utf-8")
print(b)                    # -> b'caf\xc3\xa9'
print(b.decode("utf-8"))    # -> café
```

**More string tools**

```python
print("acme corp".title())               # -> Acme Corp
print("42".zfill(5))                     # -> 00042
print("a,b,,c".split(","))               # -> ['a', 'b', '', 'c']
print("INV-0042".removeprefix("INV-"))   # -> 0042
print("  x  ".center(9, "*"))            # -> **  x  **
print("ß".casefold() == "SS".casefold()) # -> True (aggressive case-insensitive compare)
```

**Regular expressions (`re`)**: patterns for finding/extracting/validating text.

| Pattern | Means |
|---------|-------|
| `\d` | digit; `\w` word char; `\s` whitespace |
| `.` | any char (except newline) |
| `*` `+` `?` | 0+, 1+, 0 or 1 |
| `{2,4}` | between 2 and 4 times |
| `[A-Z]` | a char from the set |
| `^` `$` | start / end of string |
| `( )` | capture group; `(?P<name>...)` named group |

Always write patterns as **raw strings** `r"..."` so backslashes aren't mangled.

```python
import re

text = "Order #1234 placed by jane@acme.com on 2026-11-17, ref INV-0042"

print(re.search(r"#(\d+)", text).group(1))           # -> 1234
print(re.findall(r"[\w.+-]+@[\w-]+\.[\w.]+", text))  # -> ['jane@acme.com']

m = re.search(r"(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})", text)
print(m.group("year"), m["month"])                   # -> 2026 11

print(re.sub(r"\s+", " ", "too    many   spaces"))   # -> too many spaces
print(bool(re.fullmatch(r"INV-\d{4}", "INV-0042")))  # -> True

phone = re.compile(r"\D")                            # compile once, reuse many times
print(phone.sub("", "+1 (555) 010-9999"))            # -> 15550109999
```

- `re.search`: first match anywhere. `re.match`: only at the start. `re.fullmatch`: whole string.
- `re.findall`: all matches as a list. `re.sub`: replace.
- Flags: `re.IGNORECASE`, `re.MULTILINE`.

⚠️ Don't over-use regex. If `split()`, `startswith()` or a parser (json, csv, `email.utils`, `urllib.parse`) can do it, prefer those. Test patterns at regex101.com.

---

## Recap

- Iterators hand out items one at a time and are single-use; `itertools` has the helpers.
- Generators (`yield`) stream data lazily, which is perfect for big files and paginated APIs.
- Decorators wrap functions (timing, retry, caching); always use `functools.wraps`.
- Context managers guarantee cleanup; `@contextmanager` + `try/finally` is the easy way.
- Regex for pattern extraction; raw strings; prefer simpler tools when they work.

## Practice ideas

1. Generator `chunks(items, n)` that yields lists of size n (then compare with `itertools.batched`).
2. `@log_calls` decorator printing the function name, args and return value.
3. Context manager `cd(path)` that changes directory and changes back afterwards.
4. Extract all invoice numbers (`INV-\d+`) and amounts (`$1,234.56`) from a block of text.
