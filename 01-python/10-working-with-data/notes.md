# 10 — Working with Data: Notes

> **Why this matters for an FDE:** the first week at most customers is "here's an export, can you make sense of it?" JSON from APIs, CSVs from Excel/CRMs, messy dates and duplicates. Reading, cleaning, reshaping and writing data is the core FDE skill.

---

## 1. JSON (json module)

**JSON** is the text format of the web. It maps directly to Python types:

| JSON | Python |
|------|--------|
| object `{}` | `dict` |
| array `[]` | `list` |
| string | `str` |
| number | `int` / `float` |
| `true` / `false` | `True` / `False` |
| `null` | `None` |

```python
import json

# str ↔ Python
payload = '{"id": 1, "name": "Acme", "active": true, "tags": null}'
data = json.loads(payload)          # loads = load *string*
print(data["active"])               # -> True
text = json.dumps(data, indent=2)   # pretty-printed string
print(json.dumps({"price": 9.5}))   # -> {"price": 9.5}

# file ↔ Python
with open("customer.json", "w", encoding="utf-8") as f:
    json.dump(data, f, indent=2, ensure_ascii=False)
with open("customer.json", encoding="utf-8") as f:
    loaded = json.load(f)
```

**Things JSON can't store** (datetime, Decimal, sets, your objects) need converting:

```python
from datetime import datetime, timezone

event = {"at": datetime(2026, 11, 17, tzinfo=timezone.utc)}
print(json.dumps(event, default=str))   # -> {"at": "2026-11-17 00:00:00+00:00"}
```

**JSON Lines (`.jsonl`)**: one JSON object per line. Common for logs, exports and LLM datasets; easy to stream:

```python
rows = [{"id": 1}, {"id": 2}]
with open("events.jsonl", "w", encoding="utf-8") as f:
    for r in rows:
        f.write(json.dumps(r) + "\n")
with open("events.jsonl", encoding="utf-8") as f:
    events = [json.loads(line) for line in f if line.strip()]
print(len(events))   # -> 2
```

⚠️ JSON requires **double quotes**; `{'a': 1}` is a Python dict repr, not valid JSON. And `json.loads` vs `json.load`: the `s` means string.

---

## 2. CSV (csv module)

Never split CSV lines with `line.split(",")`: quoted fields like `"Acme, Inc."` break it. Use the `csv` module.

```python
import csv

rows = [
    {"id": 1, "name": "Acme, Inc.", "mrr": 1200},
    {"id": 2, "name": "Globex", "mrr": 800},
]

# write: newline="" is required to avoid blank lines on Windows
with open("customers.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["id", "name", "mrr"])
    writer.writeheader()
    writer.writerows(rows)

# read: each row is a dict keyed by the header
with open("customers.csv", newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):
        print(row["name"], int(row["mrr"]))   # values are ALWAYS strings → convert
# -> Acme, Inc. 1200
# -> Globex 800
```

- `csv.reader` / `csv.writer` work with lists instead of dicts.
- Other delimiters: `csv.DictReader(f, delimiter=";")` (European Excel) or `"\t"` (TSV).
- Excel exports often start with a BOM: open with `encoding="utf-8-sig"`.

---

## 3. pandas basics: read, filter, group, write

**pandas** is the go-to library for tabular data. `python3 -m pip install pandas`.

A **DataFrame** is a table; each column is a **Series**.

```python
import pandas as pd

df = pd.DataFrame({
    "customer": ["Acme", "Globex", "Acme", "Initech"],
    "plan": ["pro", "starter", "pro", "enterprise"],
    "amount": [100.0, 20.0, 150.0, 900.0],
    "date": ["2026-10-01", "2026-10-03", "2026-11-02", "2026-11-05"],
})
# usually: df = pd.read_csv("orders.csv")  / pd.read_json / pd.read_excel / pd.read_sql

# inspect
print(df.head())        # first 5 rows
print(df.shape)         # -> (4, 4)
df.info()               # column types + non-null counts
print(df.describe())    # stats for numeric columns

# select
print(df["amount"].sum())               # -> 1170.0
small = df[["customer", "amount"]]      # several columns

# filter (use & | ~ with parentheses, not and/or)
big = df[(df["amount"] > 50) & (df["plan"] != "starter")]

# new columns
df["date"] = pd.to_datetime(df["date"])
df["month"] = df["date"].dt.to_period("M")
df["amount_with_tax"] = df["amount"] * 1.17

# group & aggregate
per_customer = df.groupby("customer")["amount"].agg(["count", "sum", "mean"])
print(per_customer.sort_values("sum", ascending=False))

monthly = df.groupby("month", as_index=False)["amount"].sum()

# combine tables (like SQL JOIN)
accounts = pd.DataFrame({"customer": ["Acme", "Globex"], "owner": ["Jane", "Raj"]})
joined = df.merge(accounts, on="customer", how="left")   # Initech gets owner NaN

# write
joined.to_csv("report.csv", index=False)
# joined.to_excel("report.xlsx", index=False)   # needs: pip install openpyxl
```

Mental model: **select → filter → transform → group → write**. Almost every report is these five steps.

Other handy calls: `value_counts()`, `pivot_table()`, `sort_values()`, `rename(columns=...)`, `apply()` (slow; prefer vectorised column math), `pd.json_normalize()` (flatten nested JSON into columns).

---

## 4. Cleaning messy data

Real customer data has: extra spaces, inconsistent case, duplicates, missing values, numbers stored as text, mixed date formats.

```python
import pandas as pd

raw = pd.DataFrame({
    "email": [" Jane@Acme.com", "jane@acme.com ", None, "raj@globex.com"],
    "amount": ["1,200", "1200", "n/a", "300.5"],
    "signup": ["2026-11-01", "11/02/2026", "", "2026-11-03"],
})

df = raw.copy()

# 1. normalise text
df["email"] = df["email"].str.strip().str.lower()

# 2. numbers stored as text → numeric (bad values become NaN instead of crashing)
df["amount"] = pd.to_numeric(df["amount"].str.replace(",", ""), errors="coerce")

# 3. dates in mixed formats → datetime (unparseable → NaT)
df["signup"] = pd.to_datetime(df["signup"], format="mixed", errors="coerce")

# 4. missing values
print(df.isna().sum())                 # count missing per column
df = df.dropna(subset=["email"])       # drop rows with no email
df["amount"] = df["amount"].fillna(0)  # or fill with a default

# 5. duplicates
df = df.drop_duplicates(subset=["email"], keep="first")

print(df)
```

**Cleaning checklist** to run on every new dataset:
1. `df.shape`, `df.info()`, `df.head()`: what am I looking at?
2. Missing values: `df.isna().sum()`
3. Duplicates: `df.duplicated(subset=[key]).sum()`
4. Types: are numbers/dates really numeric/datetime?
5. Unexpected values: `df["col"].value_counts()`
6. **Keep the raw file untouched**; write cleaned output to a new file and log what you changed. Customers will ask "where did those 37 rows go?"

**Without pandas** (small data / no dependencies), the same ideas apply with plain Python: `strip()`, `lower()`, a `set()` of seen keys for de-duplication, `try/except` around conversions.

---

## 5. Dates, times and time zones

Dates are the most common source of data bugs.

```python
from datetime import datetime, timezone, timedelta
from zoneinfo import ZoneInfo

utc_now = datetime.now(timezone.utc)
karachi = utc_now.astimezone(ZoneInfo("Asia/Karachi"))
ny = utc_now.astimezone(ZoneInfo("America/New_York"))

d = datetime.strptime("17/11/2026 14:30", "%d/%m/%Y %H:%M")   # parse
print(d.strftime("%Y-%m-%d"))                                  # format -> 2026-11-17
print(d + timedelta(days=30))                                  # date math

iso = "2026-11-17T09:00:00+00:00"
print(datetime.fromisoformat(iso).tzinfo)                      # -> UTC
```

Rules:
- **Store and compute in UTC**, convert to local time only for display.
- Use **ISO 8601** (`2026-11-17T09:00:00Z`) when exchanging dates with other systems.
- `03/04/2026`: is it March 4 or April 3? **Ask the customer**, never guess.
- Unix timestamps: `datetime.fromtimestamp(1763370000, tz=timezone.utc)`.

---

## 6. Regex for data cleaning

(Regex syntax is covered in topic 08 §6.) Typical cleaning uses:

```python
import re
import pandas as pd

phones = pd.Series(["+1 (555) 010-9999", "555.010.1234", "call me"])
print(phones.str.replace(r"\D", "", regex=True).tolist())
# -> ['15550109999', '5550101234', '']

s = pd.Series(["Order INV-0042 paid", "INV-0100 refund", "none"])
print(s.str.extract(r"(INV-\d+)")[0].tolist())
# -> ['INV-0042', 'INV-0100', nan]

valid = re.compile(r"^[\w.+-]+@[\w-]+\.[\w.]+$")
print([bool(valid.match(e)) for e in ["a@b.com", "bad@", "x@y.io"]])
# -> [True, False, True]
```

---

## Recap

- `json.loads/dumps` for strings, `json.load/dump` for files; `default=str` for dates.
- Use the `csv` module (never `split(",")`); open with `newline=""`; values are strings.
- pandas: `read_*` → filter → transform → `groupby().agg()` → `merge` → `to_csv`.
- Clean in order: text → types (`to_numeric`, `to_datetime` with `errors="coerce"`) → missing → duplicates. Keep raw data untouched.
- Dates: UTC + ISO 8601; clarify ambiguous formats with the customer.

## Practice ideas

1. Load a JSON API response (e.g. from `https://jsonplaceholder.typicode.com/users`) saved to a file and write selected fields to CSV.
2. Take a messy CSV (make one: spaces, duplicates, `n/a`, mixed dates), clean it, and print a "what changed" summary.
3. With pandas: monthly revenue per plan from an orders CSV, sorted, saved as `report.csv`.
4. Flatten nested JSON with `pd.json_normalize` and compare with doing it by hand.
