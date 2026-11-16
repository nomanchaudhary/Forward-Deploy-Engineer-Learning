# 16 — Mini Project: Notes

> **Why this matters for an FDE:** this is a small version of a real customer engagement: understand the problem, build an integration, make it reliable, and hand it over so someone else can run it. It ties every previous topic together and becomes a portfolio piece.

---

## 1. Pick a problem

Choose something **realistic and small enough to finish in 1–2 weeks**. Good FDE-style ideas:

| Idea | Uses |
|------|------|
| **API → clean report:** pull data from a public API (GitHub issues, weather, exchange rates, JSONPlaceholder), clean it, store it in SQLite, export a CSV/Excel summary | requests, pagination, pandas, sqlite3 |
| **CSV reconciliation tool:** compare a "CRM export" and a "billing export"; report missing, duplicate and mismatched customers | csv/pandas, sets, pydantic, CLI |
| **Webhook receiver:** FastAPI service that verifies signed webhooks, stores events, and exposes a `/stats` endpoint | FastAPI, HMAC, sqlite, tests |
| **Concurrent sync job:** sync records from API A to "system B" (a local DB) with async, retries, rate limiting and checkpointing | httpx, asyncio, idempotency, logging |

Write a one-paragraph **problem statement** like a customer would: *"Our ops team spends 3 hours every Monday copying GitHub issues into a spreadsheet. We want a command that produces the weekly report automatically."*

---

## 2. Plan it

Before coding, write in `README.md`:

1. **Goal & success criteria**: what does "done" look like? ("Report generated in < 1 min, matches manual numbers.")
2. **Inputs / outputs**: which API/files in, which files/tables out.
3. **Data model**: the main entities and fields (draft pydantic models).
4. **Steps**: fetch → validate → transform → store → report.
5. **Risks**: rate limits, bad data, auth, big volumes. How you'll handle each.
6. **Out of scope**: what you deliberately won't do (just as important for real customers).

Suggested structure:

```text
16-mini-project/
├── README.md
├── pyproject.toml          (or requirements.txt)
├── .env.example
├── src/report_tool/
│   ├── __init__.py
│   ├── cli.py              # argparse entry point, main() -> exit code
│   ├── config.py           # settings from env vars
│   ├── client.py           # API client: session, auth, retries, pagination
│   ├── models.py           # pydantic models
│   ├── transform.py        # pure functions: cleaning & aggregation (easy to test)
│   └── storage.py          # sqlite / CSV output
└── tests/
    ├── test_transform.py
    └── test_client.py      # with mocked HTTP
```

---

## 3. Build it

Build **end-to-end thin first, then thicken**:

1. **Walking skeleton:** fetch one page → print it. Proves auth and connectivity.
2. Parse into models; write a CSV. Now you have a working (ugly) product.
3. Add pagination, retries/backoff, timeouts.
4. Add storage (SQLite) and idempotent upserts so re-runs don't duplicate.
5. Add the CLI (`--since`, `--output`, `--dry-run`, `--verbose`) and logging.
6. Optional: concurrency, FastAPI endpoint, scheduling.

Commit after each step with a clear message.

---

## 4. Test it

- Unit tests for every function in `transform.py` (happy path, edge cases, bad data).
- Client tests with mocked HTTP: success, pagination, 429 retry, 500 failure.
- One end-to-end run against the real API (manual or a marked slow test).
- Run `pytest --cov` and `mypy` (or Pylance) and fix what they find.
- Try to break it: no network, wrong API key, empty response, a malformed row.

---

## 5. Write a README & hand it over

Write the README for **someone else who has to run it** (the customer's engineer):

- What it does (2–3 sentences) and a sample output.
- Requirements (Python version) and **install** steps (venv + install).
- **Configuration**: every env var, what it means, an example (`.env.example`).
- **Usage**: commands with examples (`report-tool --since 2026-11-01 --output report.csv`).
- How to run tests.
- **Troubleshooting**: common errors and fixes (401 → check the token, 429 → lower concurrency).
- Known limitations and next steps.

Then do a **demo** (record a 3–5 minute video or present to a friend): problem → solution → live run → what you'd do next. Explaining clearly is half of the FDE job.

---

## Definition of done

- Runs from a fresh clone following only the README
- No secrets in the repo; config via env vars
- Handles bad data and API errors without crashing; logs what happened
- Re-running doesn't create duplicates
- Tests pass; core logic covered
- Demo recorded / presented
