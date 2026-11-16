# 11 — Databases from Python: Notes

> **Why this matters for an FDE:** customer data usually lives in a database (Postgres, MySQL, SQL Server, Snowflake). You'll query it, sync it, migrate it and build reports from it, all from Python. (SQL itself gets a full module later: `02 SQL & Data`. This topic is the **Python side**.)

---

## 1. SQL in 5 minutes (the minimum you need)

A relational database stores **tables** (rows + columns). You talk to it in **SQL**:

```sql
CREATE TABLE customers (id INTEGER PRIMARY KEY, name TEXT, plan TEXT, mrr REAL);
INSERT INTO customers (name, plan, mrr) VALUES ('Acme', 'pro', 1200);
SELECT name, mrr FROM customers WHERE plan = 'pro' ORDER BY mrr DESC LIMIT 10;
UPDATE customers SET plan = 'enterprise' WHERE id = 1;
DELETE FROM customers WHERE id = 1;
SELECT plan, COUNT(*), SUM(mrr) FROM customers GROUP BY plan;
SELECT o.id, c.name FROM orders o JOIN customers c ON c.id = o.customer_id;
```

That's enough to start; the SQL module goes much deeper.

---

## 2. sqlite3 (built-in database)

**SQLite** is a full SQL database stored in a single file, built into Python. Perfect for learning, local tools, caches and tests.

```python
import sqlite3

conn = sqlite3.connect("app.db")         # creates the file if missing; ":memory:" for RAM-only
conn.row_factory = sqlite3.Row           # rows behave like dicts: row["name"]

conn.execute("""
    CREATE TABLE IF NOT EXISTS customers (
        id INTEGER PRIMARY KEY,
        name TEXT NOT NULL,
        plan TEXT,
        mrr REAL
    )
""")

conn.execute("INSERT INTO customers (name, plan, mrr) VALUES (?, ?, ?)", ("Acme", "pro", 1200))
conn.executemany(
    "INSERT INTO customers (name, plan, mrr) VALUES (?, ?, ?)",
    [("Globex", "starter", 50), ("Initech", "pro", 900)],
)
conn.commit()                            # save changes!

for row in conn.execute("SELECT name, mrr FROM customers WHERE plan = ? ORDER BY mrr DESC", ("pro",)):
    print(row["name"], row["mrr"])
# -> Acme 1200.0
# -> Initech 900.0

one = conn.execute("SELECT COUNT(*) AS n FROM customers").fetchone()
print(one["n"])

conn.close()
```

- `execute()` runs one statement; `executemany()` runs it for each item.
- `fetchone()`, `fetchall()`, or iterate the cursor directly.
- **Nothing is saved until `commit()`.**

---

## 3. Parameterized queries & SQL injection

**Never** build SQL with f-strings or `+` using outside input.

```python
import sqlite3

conn = sqlite3.connect(":memory:")
conn.execute("CREATE TABLE users (name TEXT)")
conn.execute("INSERT INTO users VALUES ('jane')")

user_input = "x' OR '1'='1"

# ❌ DANGEROUS: the input becomes part of the SQL
bad_sql = f"SELECT * FROM users WHERE name = '{user_input}'"
print(len(conn.execute(bad_sql).fetchall()))       # -> 1  (returned everything!)

# ✅ SAFE: the driver sends the value separately from the SQL
safe = conn.execute("SELECT * FROM users WHERE name = ?", (user_input,)).fetchall()
print(len(safe))                                    # -> 0
```

Placeholder style depends on the driver: sqlite uses `?`, psycopg (Postgres) uses `%s`, SQLAlchemy uses `:name`. Table/column **names** can't be parameters; if they come from input, check them against an allow-list.

🔧 **FDE use:** you're often given read access to production data. One injection bug in a script you wrote is a security incident at the customer. Always parameterize.

---

## 4. Transactions & connection handling

A **transaction** groups statements so they **all succeed or all fail** (e.g. move money: debit A *and* credit B).

```python
import sqlite3

conn = sqlite3.connect(":memory:")
conn.execute("CREATE TABLE accounts (name TEXT PRIMARY KEY, balance REAL)")
conn.executemany("INSERT INTO accounts VALUES (?, ?)", [("A", 100), ("B", 0)])
conn.commit()

try:
    with conn:      # commits on success, ROLLS BACK on exception
        conn.execute("UPDATE accounts SET balance = balance - 50 WHERE name = 'A'")
        raise RuntimeError("crash halfway!")
        conn.execute("UPDATE accounts SET balance = balance + 50 WHERE name = 'B'")
except RuntimeError:
    pass

print(conn.execute("SELECT balance FROM accounts WHERE name = 'A'").fetchone()[0])
# -> 100.0  (the debit was rolled back)
conn.close()
```

Notes:
- `with conn:` in sqlite3 manages the **transaction**, not closing. Close separately (or use `contextlib.closing`).
- For big imports, insert in **batches** inside one transaction (thousands of times faster than committing every row).
- Real servers: open a connection once and reuse it (or use a **connection pool**), and always set timeouts.

---

## 5. Connecting to Postgres/MySQL & SQLAlchemy

Each database has a **driver** following the same DB-API pattern (`connect → cursor/execute → commit → close`):

| Database | Driver (`pip install ...`) |
|----------|----------------------------|
| PostgreSQL | `psycopg[binary]` |
| MySQL | `pymysql` or `mysqlclient` |
| SQL Server | `pyodbc` |
| Snowflake | `snowflake-connector-python` |

```python
import os
import psycopg

with psycopg.connect(os.environ["DATABASE_URL"]) as conn:     # never hard-code credentials
    with conn.cursor() as cur:
        cur.execute("SELECT id, name FROM customers WHERE plan = %s", ("pro",))
        for id_, name in cur.fetchall():
            print(id_, name)
```

**SQLAlchemy** gives one API for all databases. Use its **Core** for plain SQL and its **ORM** to map tables to classes:

```python
from sqlalchemy import create_engine, text

engine = create_engine("sqlite:///app.db")   # or postgresql+psycopg://user:pass@host/db
with engine.begin() as conn:                  # begin() = transaction, auto-commit
    conn.execute(text("CREATE TABLE IF NOT EXISTS t (x INTEGER)"))
    conn.execute(text("INSERT INTO t (x) VALUES (:x)"), [{"x": 1}, {"x": 2}])
    rows = conn.execute(text("SELECT x FROM t")).all()
    print(rows)
```

```python
from sqlalchemy import String, select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, Session

class Base(DeclarativeBase):
    pass

class Customer(Base):
    __tablename__ = "customer"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

Base.metadata.create_all(engine)
with Session(engine) as session:
    session.add(Customer(name="Acme"))
    session.commit()
    for c in session.scalars(select(Customer).where(Customer.name.like("A%"))):
        print(c.id, c.name)
```

**pandas ↔ SQL** (very common for reports and migrations):

```python
import pandas as pd

df = pd.read_sql("SELECT * FROM t", engine)
df.to_sql("t_copy", engine, if_exists="replace", index=False)
```

---

## Recap

- Know basic SQL: `SELECT ... WHERE ... GROUP BY ... JOIN`, `INSERT`, `UPDATE`, `DELETE`.
- `sqlite3` is built-in: `connect → execute → commit → close`; `row_factory = sqlite3.Row` for dict-like rows.
- **Always** use placeholders (`?`, `%s`, `:name`), never f-strings, for values.
- Transactions: all-or-nothing; `with conn:` commits/rolls back; batch big inserts.
- Real DBs: a driver (`psycopg`, `pymysql`) or SQLAlchemy; credentials from env vars; `pd.read_sql` / `to_sql` for data work.

## Practice ideas

1. Load a CSV into a SQLite table, then answer 3 questions with SQL (top customers, revenue by plan, orders per month).
2. Demonstrate SQL injection on a toy table, then fix it with placeholders.
3. Write `upsert_customer(conn, customer)` using `INSERT ... ON CONFLICT(id) DO UPDATE`.
4. Same report two ways: SQL `GROUP BY` vs `pd.read_sql` + `groupby`. Compare.
