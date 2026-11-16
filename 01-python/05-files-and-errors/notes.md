# 05 — Files & Errors: Notes

> **Why this matters for an FDE:** customers hand you exports, logs, and config files, and real-world input is always messy. Reading files safely and handling errors gracefully is what separates a demo script from something a customer can run unattended.

---

## 1. Reading/writing text files

`open(path, mode)` returns a file object. Modes:

| Mode | Meaning |
|------|---------|
| `"r"` | read (default); error if missing |
| `"w"` | write; **creates or erases** the file |
| `"a"` | append to the end |
| `"x"` | create; error if it already exists |
| `"b"` | add for binary, e.g. `"rb"`, `"wb"` |

```python
# write
with open("notes.txt", "w", encoding="utf-8") as f:
    f.write("line one\n")
    f.write("line two\n")

# read everything
with open("notes.txt", encoding="utf-8") as f:
    content = f.read()
print(content)

# read line by line (memory-friendly for huge files)
with open("notes.txt", encoding="utf-8") as f:
    for line in f:
        print(line.rstrip("\n"))
# -> line one
# -> line two

# append
with open("notes.txt", "a", encoding="utf-8") as f:
    f.write("line three\n")
```

Other helpers: `f.readlines()` (list of lines), `f.writelines(list_of_strings)`.

⚠️ **Common mistakes**
- Forgetting `encoding="utf-8"` → weird characters or `UnicodeDecodeError` on Windows / customer files. Always pass it. (For files from Excel, try `encoding="utf-8-sig"` to drop the BOM.)
- Using `"w"` when you meant `"a"` wipes the file.
- Calling `f.read()` on a 5 GB log file. Iterate line by line instead.

---

## 2. The with statement

`with` is a **context manager**: it sets something up and *guarantees* cleanup, even if an error happens.

```python
with open("notes.txt", encoding="utf-8") as f:
    data = f.read()
# here the file is already closed, automatically
print(f.closed)   # -> True
```

Without `with`, you must remember `f.close()`, and an exception would skip it. Always use `with` for files, locks, DB connections, network sessions. (Topic 08 shows how to build your own.)

You can open several at once:

```python
with open("notes.txt", encoding="utf-8") as src, open("copy.txt", "w", encoding="utf-8") as dst:
    dst.write(src.read())
```

---

## 3. pathlib

`pathlib.Path` is the modern way to work with file paths. It works on Windows, Mac and Linux.

```python
from pathlib import Path

base = Path("data")
report = base / "reports" / "2026-11.csv"    # join with /
print(report)          # -> data/reports/2026-11.csv
print(report.name)     # -> 2026-11.csv
print(report.stem)     # -> 2026-11
print(report.suffix)   # -> .csv
print(report.parent)   # -> data/reports

report.parent.mkdir(parents=True, exist_ok=True)   # create folders safely
report.write_text("id,total\n1,100\n", encoding="utf-8")
print(report.exists())                             # -> True
print(report.read_text(encoding="utf-8"))

for csv_file in base.rglob("*.csv"):               # find files recursively
    print(csv_file)
```

**Path relative to the script itself** (so it works no matter where you run it from):

```python
from pathlib import Path

HERE = Path(__file__).resolve().parent
config_path = HERE / "config.json"
```

🔧 **FDE use:** "process every CSV in this folder" → `for p in Path(folder).glob("*.csv"):`.

---

## 4. try / except / else / finally

An **exception** is Python's way of saying "something went wrong". Unhandled, it crashes the program with a **traceback**. You catch exceptions with `try`.

```python
def parse_amount(text):
    try:
        value = float(text)
    except ValueError:
        print(f"skipping bad amount: {text!r}")
        return None
    else:
        # runs only if no exception happened
        return round(value, 2)
    finally:
        # ALWAYS runs: success, error, or return
        pass

print(parse_amount("12.5"))   # -> 12.5
print(parse_amount("abc"))    # -> skipping bad amount: 'abc'  then None
```

**Catch several types, and get the error object:**

```python
from pathlib import Path

try:
    text = Path("missing.txt").read_text(encoding="utf-8")
except (FileNotFoundError, PermissionError) as e:
    print(f"could not read file: {e}")
```

**Common built-in exceptions**

| Exception | Typical cause |
|-----------|---------------|
| `ValueError` | right type, bad value: `int("x")` |
| `TypeError` | wrong type: `"a" + 1` |
| `KeyError` | missing dict key |
| `IndexError` | list index out of range |
| `AttributeError` | `None.strip()` and friends |
| `FileNotFoundError` | path doesn't exist |
| `ZeroDivisionError` | `x / 0` |

**Golden rules**
1. Catch **specific** exceptions. Never `except:` or `except Exception: pass` silently: that hides real bugs.
2. Keep the `try` block **small**: only the line(s) that can fail.
3. If you can't handle it meaningfully, let it propagate (or log and re-raise with `raise`).

```python
import logging

try:
    1 / 0
except ZeroDivisionError:
    logging.exception("calculation failed")   # logs full traceback
    # raise   ← uncomment to re-raise after logging
```

🔧 **FDE pattern: process all rows, collect failures, don't die on row 4,312:**

```python
rows = ["10", "x", "30"]
good, bad = [], []
for i, row in enumerate(rows, start=1):
    try:
        good.append(int(row))
    except ValueError as e:
        bad.append((i, row, str(e)))
print(good)        # -> [10, 30]
print(len(bad))    # -> 1
```

---

## 5. Raising custom exceptions

Use `raise` when *your* code detects a problem. Create your own exception classes to make errors meaningful.

```python
class ConfigError(Exception):
    """Raised when the customer's config is invalid."""


def load_port(config: dict) -> int:
    if "port" not in config:
        raise ConfigError("missing required key 'port'")
    port = config["port"]
    if not 1 <= port <= 65535:
        raise ConfigError(f"port out of range: {port}")
    return port


try:
    load_port({"port": 99999})
except ConfigError as e:
    print("Config problem:", e)   # -> Config problem: port out of range: 99999
```

**Exception chaining**: wrap a low-level error but keep the original cause:

```python
import json

class ParseError(Exception):
    pass

def parse_payload(raw: str) -> dict:
    try:
        return json.loads(raw)
    except json.JSONDecodeError as e:
        raise ParseError("payload is not valid JSON") from e
```

**Reading a traceback**: read it **from the bottom up**. The last line is the error type and message; the lines above show the call chain, with the most recent call at the bottom. Find the first line that points at *your* file.

---

## Recap

- Always `with open(path, mode, encoding="utf-8") as f:`.
- Iterate over a file for big inputs; `"w"` erases, `"a"` appends.
- Use `pathlib.Path` for paths: `/` to join, `.glob()`, `.read_text()`, `.mkdir(parents=True, exist_ok=True)`.
- `try/except` specific errors; `else` for success-only code; `finally` for cleanup.
- Define your own `class XError(Exception)` and `raise ... from e` to keep context.

## Practice ideas

1. Count lines, words and characters in a text file (like `wc`).
2. Read a log file and write only the `ERROR` lines to `errors.log`.
3. Sum a column of numbers from a file, skipping and reporting bad lines with their line numbers.
4. Write `load_config(path)` that raises `ConfigError` for a missing file or missing keys.
