# 14 — Automation & CLI Tools: Notes

> **Why this matters for an FDE:** your deliverables are often small tools a customer's team runs: `python sync.py --since 2026-11-01 --dry-run`. Proper arguments, config from env vars, real logs, exit codes and the ability to run on a schedule make a script production-ready.

---

## 1. Command-line arguments (sys.argv, argparse)

**sys.argv**: the raw list of arguments (fine for tiny scripts):

```python
import sys

# python3 greet.py Jane  → sys.argv == ['greet.py', 'Jane']
name = sys.argv[1] if len(sys.argv) > 1 else "world"
print(f"Hello, {name}")
```

**argparse**: the standard way. You get `--help`, types, defaults and error messages for free.

```python
import argparse
from datetime import date

def parse_args(argv=None):
    parser = argparse.ArgumentParser(description="Sync orders from the CRM to the warehouse.")
    parser.add_argument("source", help="path to the input CSV")                 # positional, required
    parser.add_argument("--since", type=date.fromisoformat, help="YYYY-MM-DD")
    parser.add_argument("--batch-size", type=int, default=100)
    parser.add_argument("--env", choices=["dev", "staging", "prod"], default="dev")
    parser.add_argument("--dry-run", action="store_true", help="don't write anything")
    parser.add_argument("-v", "--verbose", action="store_true")
    return parser.parse_args(argv)

args = parse_args(["orders.csv", "--since", "2026-11-01", "--dry-run"])
print(args.source, args.since, args.batch_size, args.dry_run)
# -> orders.csv 2026-11-01 100 True
```

```text
$ python3 sync.py --help
usage: sync.py [-h] [--since SINCE] [--batch-size BATCH_SIZE] [--env {dev,staging,prod}] [--dry-run] [-v] source
```

- `--batch-size` becomes `args.batch_size` (dashes → underscores).
- Subcommands (`tool.py export ...`, `tool.py import ...`): `parser.add_subparsers()`.
- Nicer third-party options: **typer** (built on type hints) and **click**.

🔧 **FDE habit:** every script that writes data gets a `--dry-run` flag. Customers love being able to see what *would* happen first.

---

## 2. Environment variables, .env files and secrets

Configuration that changes between environments (URLs, credentials, feature flags) goes in **environment variables**, not in code.

```bash
export API_KEY="sk_test_123"     # set for this shell session
API_KEY=abc python3 app.py       # set for one command only
```

```python
import os

api_key = os.environ["API_KEY"]                 # required: KeyError if missing (fail fast)
base_url = os.getenv("API_BASE_URL", "https://api.example.com")   # optional with default
timeout = float(os.getenv("TIMEOUT", "10"))     # env vars are always strings
debug = os.getenv("DEBUG", "").lower() in ("1", "true", "yes")
```

**.env files** for local development (`python3 -m pip install python-dotenv`):

```text
# .env   ← add to .gitignore!  Commit a .env.example with dummy values instead.
API_KEY=sk_test_123
API_BASE_URL=https://sandbox.example.com
```

```python
from dotenv import load_dotenv
import os

load_dotenv()                     # reads .env into os.environ (doesn't override real env vars)
print(os.getenv("API_BASE_URL"))
```

**Secrets rules**
1. Never hard-code or commit secrets. Check `git diff` before committing.
2. Never print or log them. Mask them if needed: `key[:4] + "…"`.
3. In production use the platform's secret store (AWS Secrets Manager, GCP Secret Manager, Vault, Kubernetes secrets, GitHub Actions secrets).
4. Leaked? **Rotate immediately**; deleting the commit is not enough.

(For typed config with validation, see `pydantic-settings` in topic 09.)

---

## 3. Running shell commands (subprocess)

`subprocess.run` lets Python call other programs (`git`, `aws`, `psql`, `kubectl`, ...).

```python
import subprocess

result = subprocess.run(
    ["git", "--version"],          # a LIST of args: no shell parsing, no injection
    capture_output=True,           # collect stdout/stderr
    text=True,                     # str instead of bytes
    check=True,                    # raise CalledProcessError on non-zero exit
    timeout=30,
)
print(result.stdout.strip())       # e.g. git version 2.43.0
print(result.returncode)           # -> 0
```

```python
import subprocess

try:
    subprocess.run(["ls", "/does-not-exist"], capture_output=True, text=True, check=True)
except subprocess.CalledProcessError as e:
    print("failed with code", e.returncode)
    print(e.stderr.strip())
```

⚠️ Avoid `shell=True` with any outside input: `subprocess.run(f"rm {name}", shell=True)` + a filename like `x; rm -rf ~` is a disaster. Pass a list instead.

Related stdlib: `shutil.which("aws")` (is the tool installed?), `shutil.copy/move/rmtree`, `os.walk`, `glob`.

---

## 4. Logging configuration for scripts

`print()` is for users; **logging** is for operators. Logs have levels, timestamps, and can go to files or log platforms.

```python
import logging

logging.basicConfig(
    level=logging.INFO,                                       # DEBUG < INFO < WARNING < ERROR < CRITICAL
    format="%(asctime)s %(levelname)-8s %(name)s: %(message)s",
)
log = logging.getLogger("sync")                               # one logger per module: getLogger(__name__)

log.debug("hidden at INFO level")
log.info("starting sync for %s customers", 120)               # lazy %-formatting is preferred in logs
log.warning("customer %s has no email; skipping", 42)
try:
    1 / 0
except ZeroDivisionError:
    log.exception("unexpected error")                         # ERROR + full traceback
```

```text
2026-11-17 10:00:00,123 INFO     sync: starting sync for 120 customers
2026-11-17 10:00:00,124 WARNING  sync: customer 42 has no email; skipping
```

Tips:
- Tie `--verbose` to `logging.DEBUG`: `level=logging.DEBUG if args.verbose else logging.INFO`.
- Log to a file too: `logging.basicConfig(..., handlers=[logging.StreamHandler(), logging.FileHandler("sync.log")])`.
- In cloud/containers, log to **stdout** (often as JSON lines) and let the platform collect it.
- Log **counts and ids**, not whole records with personal data.

---

## 5. Exit codes, scheduling and long-running jobs

**Exit codes** tell the caller (cron, CI, Airflow, another script) whether you succeeded: `0` = success, non-zero = failure.

```python
import sys
import logging

def main() -> int:
    failures = 0
    # ... do work, count failures ...
    if failures:
        logging.error("%d records failed", failures)
        return 1
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

```bash
python3 sync.py && echo "ok" || echo "failed"
```

**Scheduling**

| Option | When |
|--------|------|
| `cron` (Linux) | simple "every night at 2am" on a server: `0 2 * * * /app/.venv/bin/python /app/sync.py >> /var/log/sync.log 2>&1` |
| systemd timers | like cron, with better logging/restarts |
| GitHub Actions / cloud schedulers | no server to maintain |
| Airflow / Prefect / Dagster | many dependent jobs, retries, monitoring |
| `schedule` / APScheduler | scheduling inside a long-running Python process |

In cron always use **absolute paths** and the venv's Python, because cron has a minimal environment.

**Making jobs safe to re-run**
- **Idempotent:** running twice gives the same result (upsert instead of insert; track "last synced at").
- **Checkpoint** progress so a crash at record 40,000 resumes there instead of restarting.
- Handle `Ctrl+C` / SIGTERM gracefully (`except KeyboardInterrupt:` → save checkpoint, exit cleanly).
- Show progress for long runs: `tqdm` (`for row in tqdm(rows):`).

---

## 6. Packaging & sharing your tool

Make your tool installable so a customer runs `sync-tool --help` instead of `python3 path/to/script.py`.

```toml
# pyproject.toml
[project]
name = "sync-tool"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["requests>=2.32", "python-dotenv>=1.0"]

[project.scripts]
sync-tool = "sync_tool.cli:main"     # command name = module:function

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

```text
sync-tool/
├── pyproject.toml
├── README.md            ← how to install, configure (env vars), run, and troubleshoot
├── .env.example
├── src/sync_tool/
│   ├── __init__.py
│   └── cli.py           ← def main(): ...
└── tests/
```

```bash
python3 -m pip install -e .     # editable install for development
sync-tool --help
```

Other ways to ship: a **Docker image** (most common for servers, covered in the Git/Linux/Cloud module), a git repo + `requirements.txt` + README, or `uv tool install` / `pipx install` for CLI tools.

---

## Recap

- `argparse` for real CLIs: positional args, `--flags`, `type=`, `choices=`, `store_true`; add `--dry-run`.
- Config and secrets from environment variables; `.env` for local dev only, never committed.
- `subprocess.run([...], check=True, capture_output=True, text=True, timeout=...)`; avoid `shell=True`.
- `logging` with levels and timestamps instead of `print` for operational output.
- Return proper exit codes; schedule with cron/CI/orchestrators; make jobs idempotent and resumable.
- Package with `pyproject.toml` + `[project.scripts]` and a good README.

## Practice ideas

1. Turn a previous script (e.g. the CSV cleaner) into a CLI with `argparse`, `--dry-run` and `--verbose`.
2. Read an API key from `.env`, fail with a clear message and exit code 2 if missing.
3. Write a script that runs `git log --oneline -5` via subprocess and logs each commit.
4. Package it with `pyproject.toml` so it installs as a command; schedule it with cron every 5 minutes on your machine.
