# 13 — Concurrency & Async: Notes

> **Why this matters for an FDE:** "sync 50,000 records through an API" takes 14 hours one-at-a-time and 20 minutes with 40 concurrent requests. Concurrency is how your integrations finish in time. LLM apps also stream and call many tools in parallel, which is all async.

---

## 1. Concurrency vs parallelism, I/O-bound vs CPU-bound

- **Concurrency:** juggling many tasks, switching while one waits (one cook, many pots).
- **Parallelism:** literally doing several things at the same instant (many cooks).

What is your program **waiting on**?

| Workload | Example | Bottleneck | Use |
|----------|---------|------------|-----|
| **I/O-bound** | API calls, DB queries, downloads, file reads | waiting on network/disk | `threading` or `asyncio` |
| **CPU-bound** | parsing huge files, image processing, heavy math | the processor | `multiprocessing` / `ProcessPoolExecutor` |

**The GIL:** in standard CPython, only one thread runs Python bytecode at a time. Threads still help a lot for I/O (they release the GIL while waiting) but **not** for CPU work. (Python 3.13+ has an optional "free-threaded" build without the GIL, but the default build still has it.)

Most FDE work is **I/O-bound**, so threads or async are what you'll reach for 90% of the time.

---

## 2. Threads with concurrent.futures

`ThreadPoolExecutor` is the simplest, safest way to run I/O work concurrently.

```python
import time
from concurrent.futures import ThreadPoolExecutor, as_completed

def fetch(customer_id: int) -> dict:
    time.sleep(0.5)                       # pretend: an API call taking 0.5s
    return {"id": customer_id, "status": "ok"}

ids = list(range(1, 21))

start = time.perf_counter()
with ThreadPoolExecutor(max_workers=10) as pool:
    results = list(pool.map(fetch, ids))  # keeps input order
print(len(results), f"{time.perf_counter() - start:.1f}s")   # -> 20 1.0s  (vs 10s sequentially)
```

**as_completed**: handle results as they finish, and handle errors per task:

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def risky(n: int) -> int:
    if n == 3:
        raise ValueError("bad record 3")
    return n * 10

with ThreadPoolExecutor(max_workers=4) as pool:
    futures = {pool.submit(risky, n): n for n in range(5)}
    for fut in as_completed(futures):
        n = futures[fut]
        try:
            print(n, "->", fut.result())
        except ValueError as e:
            print(n, "failed:", e)
```

Rules:
- Pick `max_workers` to respect the API's **rate limit** (5–20 is typical), not "as many as possible".
- `fut.result()` re-raises the task's exception, so always handle it.
- Threads sharing a variable you **modify** need a `threading.Lock` (or avoid sharing: return results instead).

---

## 3. Processes for CPU-bound work

`ProcessPoolExecutor` runs work in separate Python processes, which gives true parallelism on multiple CPU cores.

```python
from concurrent.futures import ProcessPoolExecutor

def heavy(n: int) -> int:
    return sum(i * i for i in range(n))

if __name__ == "__main__":                 # REQUIRED: worker processes re-import this file
    with ProcessPoolExecutor() as pool:    # defaults to number of CPU cores
        results = list(pool.map(heavy, [5_000_000] * 4))
    print(len(results))                    # -> 4
```

- Arguments and results are copied between processes (pickled), so they must be picklable; avoid sending huge objects.
- Startup cost is higher than threads; worth it only for real CPU work.

---

## 4. asyncio basics: async/await, gather

**asyncio** runs many I/O tasks concurrently in **one thread** using an **event loop**. A task gives up control at every `await`, letting others run while it waits.

```python
import asyncio
import time

async def fetch(customer_id: int) -> dict:      # "async def" = coroutine function
    await asyncio.sleep(0.5)                    # non-blocking wait (an API call, in real life)
    return {"id": customer_id}

async def main():
    start = time.perf_counter()
    results = await asyncio.gather(*(fetch(i) for i in range(20)))   # run all concurrently
    print(len(results), f"{time.perf_counter() - start:.1f}s")       # -> 20 0.5s

asyncio.run(main())                             # entry point: starts the event loop
```

Key ideas:
- `async def` defines a **coroutine**. Calling it does **not** run it; you must `await` it (or schedule it with `gather` / `create_task`).
- You can only use `await` inside `async def`.
- `asyncio.run(main())` is called **once**, at the top of your program.

**Limit concurrency with a Semaphore** (respect rate limits):

```python
import asyncio

async def worker(i: int, sem: asyncio.Semaphore) -> int:
    async with sem:                 # at most N inside at once
        await asyncio.sleep(0.1)
        return i

async def main():
    sem = asyncio.Semaphore(5)
    results = await asyncio.gather(*(worker(i, sem) for i in range(50)))
    print(sum(results))             # -> 1225

asyncio.run(main())
```

**Timeouts and errors:**

```python
import asyncio

async def slow():
    await asyncio.sleep(5)

async def main():
    try:
        await asyncio.wait_for(slow(), timeout=0.2)
    except TimeoutError:
        print("gave up after 0.2s")
    results = await asyncio.gather(asyncio.sleep(0.1), slow_fail(), return_exceptions=True)
    print([type(r).__name__ for r in results])   # -> ['NoneType', 'ValueError']

async def slow_fail():
    raise ValueError("boom")

asyncio.run(main())
```

⚠️ **The #1 async mistake:** calling **blocking** code inside async functions (`time.sleep`, `requests.get`, a sync DB driver). It freezes the whole event loop. Use async libraries (`asyncio.sleep`, `httpx.AsyncClient`, `asyncpg`), or push blocking calls to a thread with `await asyncio.to_thread(func, *args)`.

---

## 5. Async HTTP with httpx

`python3 -m pip install httpx`

```python
import asyncio
import httpx

async def fetch_user(client: httpx.AsyncClient, sem: asyncio.Semaphore, user_id: int) -> dict:
    async with sem:
        resp = await client.get(f"/users/{user_id}")
        resp.raise_for_status()
        return resp.json()

async def main():
    sem = asyncio.Semaphore(5)
    async with httpx.AsyncClient(base_url="https://jsonplaceholder.typicode.com", timeout=10) as client:
        users = await asyncio.gather(*(fetch_user(client, sem, i) for i in range(1, 11)))
    print([u["name"] for u in users][:3])

if __name__ == "__main__":
    asyncio.run(main())
```

- Create **one** client and reuse it (connection pooling).
- `httpx.Client` is the sync version with the same API as `requests`.
- FastAPI endpoints can be `async def` too, and the Anthropic/OpenAI SDKs have async clients (`AsyncAnthropic`), the same pattern.

**Threads or asyncio?**
- Already have sync code / sync libraries → `ThreadPoolExecutor` (minimal change).
- Building something new with lots of concurrent I/O (thousands of requests, websockets, streaming) → `asyncio` + `httpx`.

---

## Recap

- I/O-bound → threads or asyncio. CPU-bound → processes.
- `ThreadPoolExecutor.map/submit` is the easy win for existing sync code; cap `max_workers`.
- `ProcessPoolExecutor` needs the `if __name__ == "__main__":` guard and picklable data.
- asyncio: `async def` + `await`, `asyncio.gather`, `asyncio.run(main())` once, `Semaphore` to limit, `wait_for` for timeouts.
- Never block inside async code; use async libraries or `asyncio.to_thread`.

## Practice ideas

1. Fetch 100 JSONPlaceholder posts sequentially, then with `ThreadPoolExecutor`, then with `httpx` + asyncio. Time all three.
2. Add a `Semaphore(5)` and retry-on-429 logic to the async version.
3. Use `ProcessPoolExecutor` to count words in 8 large text files in parallel.
4. Write an async producer/consumer with `asyncio.Queue` (one task produces ids, three workers process them).
