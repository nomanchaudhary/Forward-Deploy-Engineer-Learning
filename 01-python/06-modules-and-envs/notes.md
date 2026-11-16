# 06 — Modules & Environments: Notes

> **Why this matters for an FDE:** "it works on my machine" is the #1 deployment failure. Knowing how imports, packages and virtual environments work lets you ship code that runs the same on the customer's server.

---

## 1. import and your own modules

A **module** is any `.py` file. A **package** is a folder of modules (optionally with `__init__.py`).

```python
import math
print(math.sqrt(16))          # -> 4.0

from math import sqrt, pi     # import specific names
print(round(pi, 2))           # -> 3.14

import datetime as dt         # alias
print(dt.date(2026, 1, 1))    # -> 2026-01-01
```

**Your own module:**

```text
project/
├── main.py
└── utils/
    ├── __init__.py
    └── text.py
```

```python
# utils/text.py
def normalize(s: str) -> str:
    return s.strip().lower()
```

```python
# main.py
from utils.text import normalize
print(normalize("  ACME "))   # -> acme
```

Run from the project root: `python3 main.py`. Python searches for imports in the script's folder, then installed packages (`sys.path` shows the full list).

⚠️ **Common mistakes**
- Naming your file the same as a library (`requests.py`, `json.py`, `random.py`) → you import your file instead of the library.
- `from module import *`: you can't tell where names came from. Avoid it.
- Circular imports (a imports b, b imports a). Move shared code into a third module.

---

## 2. Standard library tour (os, sys, datetime, collections)

Python ships "batteries included". Learn these before reaching for pip.

**os**: operating system & environment variables

```python
import os

print(os.getcwd())                         # current directory
api_key = os.environ.get("API_KEY")         # None if not set
debug = os.getenv("DEBUG", "false") == "true"
```

**sys**: the interpreter itself

```python
import sys

print(sys.version)      # Python version
print(sys.argv)         # command-line args: ['script.py', 'arg1', ...]
# sys.exit(1)           # exit with an error code (non-zero = failure)
```

**datetime**: dates and times

```python
from datetime import datetime, date, timedelta, timezone

now = datetime.now(timezone.utc)                # always prefer timezone-aware
print(now.isoformat())                          # e.g. 2026-11-17T09:30:00.123456+00:00
yesterday = date.today() - timedelta(days=1)
parsed = datetime.strptime("2026-11-17 14:05", "%Y-%m-%d %H:%M")
print(parsed.strftime("%d %b %Y"))              # -> 17 Nov 2026
print(datetime.fromisoformat("2026-11-17T14:05:00+00:00").year)   # -> 2026
```

**collections**: better containers (see topic 03): `Counter`, `defaultdict`, `deque`, `namedtuple`.

**Other stdlib modules you'll use constantly**

| Module | For |
|--------|-----|
| `json`, `csv` | data formats (topic 10) |
| `pathlib`, `shutil` | paths, copy/move files |
| `re` | regular expressions (topic 10) |
| `logging` | real logs instead of print (topic 15) |
| `argparse`, `subprocess` | CLIs and shell commands (topic 14) |
| `itertools`, `functools` | iteration helpers, caching, `partial` |
| `uuid`, `hashlib`, `secrets` | ids, hashes, secure tokens |
| `time` | `time.sleep()`, timing code |

---

## 3. pip and requirements.txt

**pip** installs third-party packages from PyPI.

```bash
python3 -m pip install requests          # install
python3 -m pip install "requests==2.32.3"  # exact version
python3 -m pip list                      # what's installed
python3 -m pip show requests             # details of one package
python3 -m pip uninstall requests
```

Use `python3 -m pip` (not bare `pip`) so you install into the **same** Python you run.

**requirements.txt**: the list of packages your project needs:

```text
requests==2.32.3
pandas>=2.2,<3
python-dotenv==1.0.1
```

```bash
python3 -m pip freeze > requirements.txt   # snapshot current env
python3 -m pip install -r requirements.txt # recreate it elsewhere
```

**Pin versions** for anything you deploy to a customer. Unpinned = it can break next month without you changing a line.

**Modern alternatives** you'll meet: `uv` (very fast pip + venv replacement), `poetry`, and `pyproject.toml` (the standard project config file). The concepts are identical.

```bash
uv venv && uv pip install -r requirements.txt   # if uv is available
```

---

## 4. Virtual environments (venv)

A **virtual environment** is an isolated folder with its own Python + packages, so project A's `pandas 1.x` doesn't fight project B's `pandas 2.x`.

```bash
python3 -m venv .venv            # create (once per project)
source .venv/bin/activate        # activate (Linux/Mac)
# .venv\Scripts\activate         # activate (Windows)
which python                     # now points inside .venv
python -m pip install -r requirements.txt
deactivate                       # leave the venv
```

Rules:
- One venv per project, in the project folder, named `.venv`.
- **Never commit** `.venv/` (it's in `.gitignore`); commit `requirements.txt` instead.
- If things get weird: delete `.venv` and recreate it. It's disposable.

🔧 **FDE use:** on a customer server you'll typically `git clone`, create a venv, `pip install -r requirements.txt`, and run. Practise this until it's muscle memory.

---

## 5. `if __name__ == "__main__"`

Every module has a `__name__`. When you **run** a file directly, it's `"__main__"`. When it's **imported**, it's the module's name.

```python
# report.py
def build_report(rows):
    return f"{len(rows)} rows"

def main():
    print(build_report([1, 2, 3]))

if __name__ == "__main__":
    main()
```

- `python3 report.py` → runs `main()` → prints `3 rows`.
- `from report import build_report` in another file or a test → `main()` does **not** run.

Always structure scripts this way: functions at the top, a `main()` function, and the guard at the bottom. It makes the code importable and testable.

---

## Recap

- Module = `.py` file; package = folder of modules. Don't name files after libraries.
- Learn the stdlib first: `os`, `sys`, `datetime`, `pathlib`, `json`, `csv`, `collections`, `logging`.
- `python3 -m pip install`; pin versions in `requirements.txt`.
- One `.venv` per project; never commit it.
- End every script with `if __name__ == "__main__": main()`.

## Practice ideas

1. Create a venv, install `requests`, freeze to `requirements.txt`, delete the venv, recreate it from the file.
2. Split a script into `main.py` + `helpers/` package and import from it.
3. Print the number of days until a date given as `YYYY-MM-DD`.
4. Read a `NAME` env var and greet it, with a default if it's not set.
