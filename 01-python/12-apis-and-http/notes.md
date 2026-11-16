# 12 — APIs & HTTP: Notes

> **Why this matters for an FDE:** integration *is* the job. Connecting the customer's CRM to your platform, pulling tickets from Zendesk, receiving Stripe webhooks, exposing a small service. All of it is HTTP APIs. This is probably the single most important topic in this module.

---

## 1. HTTP basics: methods, status codes, headers

HTTP is a **request → response** conversation between a client (your script) and a server.

**A request has:** a method, a URL, headers, and (sometimes) a body.

```text
POST https://api.example.com/v1/customers?notify=true
Authorization: Bearer sk_live_abc
Content-Type: application/json

{"name": "Acme", "plan": "pro"}
```

**URL anatomy:** `https://` scheme · `api.example.com` host · `/v1/customers` path · `?notify=true` query params.

**Methods (verbs)**

| Method | Meaning | Body? | Safe to retry? |
|--------|---------|-------|----------------|
| `GET` | read | no | yes |
| `POST` | create / action | yes | **no** (may create duplicates) |
| `PUT` | replace | yes | yes (idempotent) |
| `PATCH` | partial update | yes | usually |
| `DELETE` | delete | no | yes |

**Status codes**: the first digit tells you the story:

| Code | Meaning | What to do |
|------|---------|------------|
| `200 OK`, `201 Created`, `204 No Content` | success | carry on |
| `301/302` | redirect | client usually follows |
| `400 Bad Request` | your payload is wrong | fix the request; don't retry |
| `401 Unauthorized` | missing/invalid credentials | check the token |
| `403 Forbidden` | authenticated but not allowed | check permissions/scopes |
| `404 Not Found` | wrong URL or id | check path |
| `409 Conflict` / `422 Unprocessable` | validation / state conflict | read the error body |
| `429 Too Many Requests` | rate limited | wait (`Retry-After`) and retry |
| `500/502/503/504` | server problem | retry with backoff |

**Headers** you'll use constantly: `Authorization`, `Content-Type: application/json`, `Accept`, `User-Agent`, `Retry-After`, `X-Request-Id`.

**REST** conventions: resources are nouns (`/customers`, `/customers/42/orders`), and the method says the action.

---

## 2. requests library

`python3 -m pip install requests`

```python
import requests

resp = requests.get(
    "https://jsonplaceholder.typicode.com/users",
    params={"username": "Bret"},       # → ?username=Bret (properly encoded)
    timeout=10,                        # ALWAYS set a timeout
)
print(resp.status_code)                # -> 200
print(resp.headers["Content-Type"])    # -> application/json; charset=utf-8
users = resp.json()                    # parse JSON body → Python
print(users[0]["email"])

# POST JSON
new = requests.post(
    "https://jsonplaceholder.typicode.com/posts",
    json={"title": "hi", "body": "test", "userId": 1},   # json= sets Content-Type for you
    timeout=10,
)
print(new.status_code, new.json()["id"])   # -> 201 101
```

**Error handling**: `requests` does NOT raise on 4xx/5xx by itself:

```python
import requests

try:
    r = requests.get("https://jsonplaceholder.typicode.com/nope", timeout=10)
    r.raise_for_status()               # raises HTTPError for 4xx/5xx
except requests.exceptions.HTTPError as e:
    print("HTTP error:", e.response.status_code, e.response.text[:200])
except requests.exceptions.Timeout:
    print("timed out")
except requests.exceptions.ConnectionError:
    print("network problem")
```

**Sessions**: reuse connections and default headers (faster, cleaner):

```python
import requests

session = requests.Session()
session.headers.update({"Accept": "application/json", "User-Agent": "fde-sync/1.0"})
r = session.get("https://jsonplaceholder.typicode.com/todos/1", timeout=10)
print(r.json()["title"])
```

`httpx` is a modern alternative with the same feel plus **async** support (topic 13).

⚠️ **Common mistakes**
- No `timeout` → the script can hang forever on a dead server.
- Not checking the status code and then calling `.json()` on an HTML error page.
- Logging full responses or headers that contain tokens or personal data.

---

## 3. Auth: API keys, tokens, OAuth

| Type | How it's sent |
|------|---------------|
| API key | header like `X-API-Key: abc` (sometimes a query param) |
| Bearer token | `Authorization: Bearer <token>` |
| Basic auth | `auth=("user", "pass")` in requests |
| OAuth 2.0 | exchange client id/secret (or a user login) for a short-lived **access token**, then send it as Bearer |

```python
import os
import requests

API_KEY = os.environ["EXAMPLE_API_KEY"]          # from env, NEVER hard-coded or committed

resp = requests.get(
    "https://api.example.com/v1/me",
    headers={"Authorization": f"Bearer {API_KEY}"},
    timeout=10,
)
```

**OAuth client-credentials flow** (machine-to-machine, very common in enterprise integrations):

```python
import os
import requests

token_resp = requests.post(
    "https://auth.example.com/oauth/token",
    data={
        "grant_type": "client_credentials",
        "client_id": os.environ["CLIENT_ID"],
        "client_secret": os.environ["CLIENT_SECRET"],
        "scope": "read:orders",
    },
    timeout=10,
)
token_resp.raise_for_status()
access_token = token_resp.json()["access_token"]   # expires: refresh when you get a 401
```

**Secrets hygiene:** keep secrets in env vars / a `.env` file that's in `.gitignore` / a secrets manager. Rotate any key that was ever pasted into chat, code, or logs.

---

## 4. Pagination and rate limits

APIs return big lists in **pages**. Three common styles:

| Style | Request | How you know there's more |
|-------|---------|---------------------------|
| page/offset | `?page=2&per_page=100` or `?offset=200&limit=100` | fewer items than `limit`, or a `total` |
| cursor | `?cursor=abc123` | response contains `next_cursor` (null at the end) |
| link header | follow `resp.links["next"]["url"]` | no `next` link |

**Cursor pagination as a generator** (topic 08):

```python
import requests

def fetch_all(session: requests.Session, url: str, params: dict | None = None):
    params = dict(params or {})
    while True:
        resp = session.get(url, params=params, timeout=30)
        resp.raise_for_status()
        body = resp.json()
        yield from body["data"]
        cursor = body.get("next_cursor")
        if not cursor:
            break
        params["cursor"] = cursor
```

**Rate limits**: APIs cap requests per second/minute. Handle `429` and transient `5xx` with **retries + exponential backoff**:

```python
import time
import requests

RETRYABLE = {429, 500, 502, 503, 504}

def get_with_retry(session, url, max_attempts=5, **kwargs):
    for attempt in range(1, max_attempts + 1):
        resp = session.get(url, timeout=30, **kwargs)
        if resp.status_code not in RETRYABLE:
            resp.raise_for_status()
            return resp
        wait = float(resp.headers.get("Retry-After", 2 ** attempt))
        print(f"{resp.status_code}; retry {attempt}/{max_attempts} in {wait:.0f}s")
        time.sleep(wait)
    resp.raise_for_status()
```

Production shortcut: `requests` + `urllib3.util.Retry` mounted on an `HTTPAdapter`, or the `tenacity` library.

**Idempotency:** when retrying `POST`s, send an `Idempotency-Key` header if the API supports it (Stripe does), so a retry can't create a duplicate charge or order.

---

## 5. Webhooks

A **webhook** is the reverse of an API call: the *other* system calls *your* URL when something happens (payment succeeded, ticket created).

Key rules:
1. **Verify the signature**: providers sign the body with a shared secret so you know it's really them.
2. **Respond fast** (2xx within a few seconds); do heavy work in the background.
3. **Expect duplicates and out-of-order delivery**: de-duplicate by event id.

```python
import hashlib
import hmac

def verify_signature(body: bytes, received_sig: str, secret: str) -> bool:
    expected = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, received_sig)   # constant-time compare

body = b'{"event":"order.paid","id":"evt_1"}'
sig = hmac.new(b"whsec_test", body, hashlib.sha256).hexdigest()
print(verify_signature(body, sig, "whsec_test"))   # -> True
print(verify_signature(body, "forged", "whsec_test"))  # -> False
```

For local testing, tunnel a public URL to your laptop with `ngrok` or `cloudflared`.

---

## 6. Build a small API with FastAPI

**FastAPI** builds APIs from type hints + pydantic (topic 09). `python3 -m pip install "fastapi[standard]"`

```python
# main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI(title="Customer API")

class CustomerIn(BaseModel):
    name: str
    plan: str = "starter"

class Customer(CustomerIn):
    id: int

DB: dict[int, Customer] = {}

@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/customers/{customer_id}")
def get_customer(customer_id: int) -> Customer:
    if customer_id not in DB:
        raise HTTPException(status_code=404, detail="customer not found")
    return DB[customer_id]

@app.post("/customers", status_code=201)
def create_customer(payload: CustomerIn) -> Customer:
    customer = Customer(id=len(DB) + 1, **payload.model_dump())
    DB[customer.id] = customer
    return customer
```

```bash
fastapi dev main.py                      # or: uvicorn main:app --reload
# open http://127.0.0.1:8000/docs  → interactive Swagger UI for free
curl -X POST localhost:8000/customers -H "Content-Type: application/json" -d '{"name":"Acme"}'
```

FastAPI gives you request validation (bad input → automatic `422` with details), JSON serialisation, and OpenAPI docs. Flask is the older, simpler alternative you'll also see in customer codebases.

**Testing APIs by hand:** `curl`, the `/docs` page, Postman/Bruno/HTTPie.

---

## Recap

- Request = method + URL + headers + body; response = status + headers + body.
- 2xx ok · 4xx your fault (don't blindly retry, except 429) · 5xx their fault (retry with backoff).
- `requests`: always `timeout=`, `raise_for_status()`, `json=` for bodies, `Session` for reuse.
- Secrets from env vars; Bearer tokens in `Authorization`; OAuth gives short-lived tokens.
- Paginate with a generator; handle 429 with `Retry-After` + exponential backoff; idempotency keys for POST retries.
- Webhooks: verify HMAC signature, respond fast, de-duplicate.
- FastAPI + pydantic = a validated, documented API in a few lines.

## Practice ideas

1. Fetch all posts from JSONPlaceholder, count posts per user, save to CSV.
2. Use the GitHub API (`https://api.github.com/repos/python/cpython/issues?per_page=50`) with link-header pagination for 3 pages.
3. Wrap an API in a small `Client` class (base URL, token, session, retry, `list_x()` generator).
4. Build a FastAPI service with `/health`, `GET /items/{id}`, `POST /items`, and a `/webhook` endpoint that verifies an HMAC signature.
