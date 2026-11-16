# 03 — Data Structures: Notes

> **Why this matters for an FDE:** customer data arrives as JSON (dicts + lists). Picking the right structure makes lookups instant and code short.

---

## 1. Lists (indexing, slicing, methods)

A **list** is an ordered, changeable collection. Square brackets.

```python
tickets = ["T-1", "T-2", "T-3"]

print(tickets[0])      # -> T-1
print(tickets[-1])     # -> T-3
print(tickets[1:])     # -> ['T-2', 'T-3']
print(len(tickets))    # -> 3
print("T-2" in tickets)  # -> True
```

**Common methods** (these change the list *in place*):

```python
tickets.append("T-4")        # add to end
tickets.insert(0, "T-0")     # add at index
tickets.remove("T-2")        # remove first matching value
last = tickets.pop()         # remove & return last item
tickets.extend(["T-8", "T-9"])  # add many
print(tickets)   # -> ['T-0', 'T-1', 'T-3', 'T-8', 'T-9']
print(last)      # -> T-4
```

**Sorting**

```python
nums = [5, 2, 9, 1]
print(sorted(nums))                # -> [1, 2, 5, 9]   (new list)
print(sorted(nums, reverse=True))  # -> [9, 5, 2, 1]
nums.sort()                        # sorts in place, returns None
print(nums)                        # -> [1, 2, 5, 9]
```

Useful built-ins: `sum()`, `min()`, `max()`, `len()`, `any()`, `all()`.

```python
print(sum([10, 20, 30]))              # -> 60
print(any(n > 25 for n in [10, 30]))  # -> True
```

⚠️ **Common mistakes**
- `x = x.sort()` sets `x` to `None`. Use `x.sort()` or `x = sorted(x)`.
- **Aliasing:** `b = a` does NOT copy. Both names point to the same list.

```python
a = [1, 2]
b = a
b.append(3)
print(a)           # -> [1, 2, 3]   surprise!
c = a.copy()       # or list(a) or a[:]
```

---

## 2. Tuples and unpacking

A **tuple** is like a list but **immutable** (can't change). Parentheses.

```python
point = (10, 20)
rgb = (255, 128, 0)
single = (5,)      # note the comma: (5) is just the number 5
```

Use tuples for fixed groups of values: coordinates, a DB row, `(status, body)`.

**Unpacking:** assign items to names in one go:

```python
x, y = point
print(x, y)            # -> 10 20

a, b = 1, 2
a, b = b, a            # swap without a temp variable
print(a, b)            # -> 2 1

first, *rest = [1, 2, 3, 4]
print(first, rest)     # -> 1 [2, 3, 4]
```

Functions often return tuples:

```python
def min_max(values):
    return min(values), max(values)

low, high = min_max([4, 8, 1])
print(low, high)   # -> 1 8
```

**namedtuple:** a tuple with field names (lightweight record):

```python
from collections import namedtuple

Customer = namedtuple("Customer", ["id", "name"])
c = Customer(1, "Acme")
print(c.name)   # -> Acme
```

---

## 3. Dictionaries

A **dict** maps **keys → values**. Curly braces. Lookups by key are very fast.

```python
customer = {
    "id": 42,
    "name": "Acme",
    "plan": "pro",
    "tags": ["vip", "emea"],
}

print(customer["name"])            # -> Acme
customer["plan"] = "enterprise"    # update
customer["seats"] = 300            # add new key
del customer["tags"]               # remove
print("plan" in customer)          # -> True  (checks keys)
```

**Safe access with .get()**: no crash if the key is missing:

```python
print(customer.get("phone"))             # -> None
print(customer.get("phone", "unknown"))  # -> unknown
```

**Looping**

```python
for key, value in customer.items():
    print(key, "=", value)
# keys():   just the keys
# values(): just the values
```

**Merging**

```python
defaults = {"region": "us", "plan": "starter"}
overrides = {"plan": "pro"}
config = {**defaults, **overrides}    # or: defaults | overrides  (3.9+)
print(config)   # -> {'region': 'us', 'plan': 'pro'}
```

**Nested data** (exactly what JSON from an API looks like):

```python
resp = {"data": {"users": [{"name": "Jane"}, {"name": "Raj"}]}}
print(resp["data"]["users"][1]["name"])   # -> Raj
names = [u["name"] for u in resp["data"]["users"]]
print(names)   # -> ['Jane', 'Raj']
```

**Counting / grouping helpers** from `collections`:

```python
from collections import Counter, defaultdict

print(Counter(["pro", "pro", "starter"]))   # -> Counter({'pro': 2, 'starter': 1})

by_plan = defaultdict(list)
for name, plan in [("Acme", "pro"), ("Globex", "pro"), ("Initech", "starter")]:
    by_plan[plan].append(name)
print(dict(by_plan))   # -> {'pro': ['Acme', 'Globex'], 'starter': ['Initech']}
```

⚠️ **Common mistakes**
- `d["missing"]` raises `KeyError`. Use `.get()` when the key may be absent.
- Keys must be **immutable** (str, int, tuple), never a list.

---

## 4. Sets

A **set** is an unordered collection of **unique** values. Great for de-duplication and "is X in here?" checks.

```python
emails = ["a@x.com", "b@x.com", "a@x.com"]
unique = set(emails)
print(len(unique))   # -> 2

empty = set()        # NOT {}: that's an empty dict
```

**Set math**: comparing two lists of IDs:

```python
in_crm = {1, 2, 3, 4}
in_billing = {3, 4, 5}

print(in_crm & in_billing)   # -> {3, 4}        both (intersection)
print(in_crm | in_billing)   # -> {1, 2, 3, 4, 5} either (union)
print(in_crm - in_billing)   # -> {1, 2}        in CRM but not billed!
print(in_crm ^ in_billing)   # -> {1, 2, 5}     in exactly one
```

🔧 **FDE use:** "Which customers exist in system A but not system B?" is a one-line set difference. This comes up in almost every data migration.

---

## 5. Choosing the right structure

| Need | Use | Why |
|------|-----|-----|
| Ordered items you'll add/remove | `list` | flexible, keeps order |
| Fixed record that shouldn't change | `tuple` / `namedtuple` | immutable, can be a dict key |
| Look things up by a key (id → record) | `dict` | O(1) lookup |
| Unique values / membership tests / compare groups | `set` | O(1) `in`, set math |
| Count things | `Counter` | built for it |
| Group things | `defaultdict(list)` | no "if key not in" boilerplate |

**Performance rule of thumb:** `x in some_list` checks every item (slow for big lists). `x in some_set` / `x in some_dict` is instant. If you check membership in a loop, convert to a set first.

```python
blocked_ids = set(range(100_000))   # set, not list
print(99_999 in blocked_ids)        # -> True, instantly
```

**Index a list of records by id** (very common):

```python
users = [{"id": 1, "name": "Jane"}, {"id": 2, "name": "Raj"}]
by_id = {u["id"]: u for u in users}
print(by_id[2]["name"])   # -> Raj
```

---

## Recap

- `list` = ordered + changeable; `tuple` = ordered + fixed.
- `dict` = key → value; use `.get()` for optional keys.
- `set` = unique values + fast membership + set math.
- Copy explicitly (`.copy()`); `b = a` is the same object.
- `Counter` and `defaultdict` save lots of boilerplate.

## Practice ideas

1. Given a list of order dicts, compute total revenue per customer (dict).
2. Find emails that appear in list A but not list B.
3. Count the 3 most common words in a paragraph (`Counter.most_common(3)`).
4. Turn a list of users into a dict keyed by `id`, then look one up.
