# Session Manager — Coding Interview Practice

Practice for engineering-scenario coding interviews, where you write an OOP class and the
interviewer keeps adding constraints, rather than solving classic algorithm puzzles.

Built step by step in 5 levels, from a basic in-memory class to a thread-safe version with
background cleanup, plus a discussion of how to scale it with Redis.

---

## The problem

> Implement a session manager for a web application (or an AI agent backend). When a user logs in
> or starts a conversation, they get a session. The session expires after a fixed period of time.
> Users can also end their session manually.

**API**

| Method | Behavior |
|---|---|
| `create_session(user_id)` | Creates a session, returns a unique `session_id` |
| `get_session(session_id)` | Returns `user_id` if the session exists and hasn't expired, else `None` |
| `delete_session(session_id)` | Removes the session; no error if it doesn't exist |

**Where it's used:** login sessions on websites, conversation threads in chat/agent apps
(e.g. LangGraph `thread_id` + checkpointer), seat holds in ticket booking.
The backend calls `get_session` on every request — it's the "door check" before any business logic runs.

---

## Level 1 — Basic version

**Data model:** a dict of dicts

```python
self.sessions = {
    "3f2a-...": {"user_id": "seren", "expires_at": 1759830000.0},
}
```

**Key ideas**
- `uuid.uuid4()` → random, practically globally unique ID (unguessable, no coordination needed across servers)
- `time.time()` → current time in seconds; expired means `expires_at <= now`
- **TTL** (time to live) = a duration; `expires_at` = a point in time
- **Lazy expiration:** expired sessions are deleted when someone accesses them
- **Early return** keeps `get_session` readable: not found → `None`; expired → delete + `None`; valid → `user_id`

## Level 2 — Sliding expiration

Extend the session every time it's successfully accessed.

**Rule: check first, extend last.** If you extend before the expiry check, expired sessions get revived and never expire.

```python
if session["expires_at"] <= time.time():
    self.sessions.pop(session_id)
    return None
session["expires_at"] = time.time() + self.ttl   # only after all checks pass
return session["user_id"]
```

Clarifying question worth asking: *"Should every access extend the session, or only user-initiated actions?"*
(background polling could otherwise keep a session alive forever)

## Level 3 — Efficient cleanup with a min-heap

**Problem:** sessions that are never accessed again are never deleted → memory grows forever.

**Naive fix:** scan all sessions → O(n) per cleanup, even if only a few expired.

**Better:** keep two structures over the same data

| Structure | Good at | Used by |
|---|---|---|
| `dict` | O(1) lookup by `session_id` | `get_session`, `delete_session` |
| min-heap of `(expires_at, session_id)` | O(1) peek at the soonest-expiring | `cleanup_expired` |

Cleanup pops from the heap top until it reaches an entry that hasn't expired → **O(k log n)**, k = number expired.
Trade-off: `create_session` becomes O(log n) because of the heap push.

**The stale-entry trap:** sliding expiration updates the dict, but the old heap entry stays.
So when popping from the heap, check the dict for the real expiry:

1. Not in dict anymore (already deleted) → skip
2. Really expired → delete from dict
3. Was extended → **push the new `(expires_at, session_id)` back**, otherwise it is never checked again

## Level 4 — Concurrency

**Race condition example:** request thread checks "expired?" → cleanup thread deletes the session →
request thread tries to delete it → `KeyError`. Two heap pushes at the same time can also corrupt heap order silently.

**Fix:** one `threading.Lock` shared by all methods; wrap every read/write of shared state.

```python
with self.lock:          # acquire; other threads wait here
    ...                  # critical section
                         # released automatically, even on return or exception
```

- Shared state (`self.sessions`, `self.heap`) → inside the lock
- Local work (`uuid4()`, computing `expires_at`) → outside the lock, to keep the critical section short

**Background cleanup thread**

```python
self.cleanup_thread = threading.Thread(target=self._cleanup_loop, daemon=True)
self.cleanup_thread.start()

def _cleanup_loop(self):
    while True:
        self.cleanup_expired()
        time.sleep(self.cleanup_interval)
```

- `target=self._cleanup_loop` — pass the function, **no parentheses**
- `daemon=True` — thread ends when the main program ends
- Runs on a timer, independent of whether anyone calls the other methods

**Possible follow-ups**
- *Stop the thread cleanly?* → use `threading.Event` as a stop flag instead of `while True`
- *Cleanup holds the lock too long?* → clean in batches (e.g. max 1000 per round), release the lock in between

## Level 5 — Scaling to millions of users (discussion)

**Problem with 50 servers behind a load balancer:** each server has its own in-memory dict.
- A user's next request may hit another server → session not found
- A server restart wipes all its sessions

**Fix:** move session state to a shared store — **Redis** (a networked, in-memory key-value store).

| What I built by hand | Redis equivalent |
|---|---|
| `self.sessions` dict | Redis keys |
| `expires_at` + heap + cleanup thread | TTL (`set(key, value, ex=seconds)`), auto-deleted |
| Sliding expiration | `expire(key, seconds)` |
| `threading.Lock` | Single Redis commands are atomic |

Redis itself expires keys lazily on access plus periodic sampling — the same two ideas as Levels 1 and 3–4.

**New problems a single Redis creates**
- Bottleneck / latency → **sharding** by hash of `session_id`
- Single point of failure → **replication** with automatic failover
- Capacity → sharding
- In practice: a managed Redis service from the cloud provider

The public interface stays the same; only the storage backend changes.

---

## Python notes

- `dict.pop(key, default)` removes and returns; without a default it raises `KeyError` if missing
- Don't delete from a dict while iterating it — iterate over a copy: `for k, v in list(d.items()):`
- `heapq` compares tuples by their first element → `(expires_at, session_id)` sorts by expiry; `h[0]` peeks, `heappop` removes
- `continue` only works inside a loop
- Class names are capitalized: `threading.Lock()`, `threading.Thread(...)`
- Indentation defines scope — a `def` indented one level too deep becomes a nested function, not a method
- In Colab, re-run the class cell after editing it, otherwise tests use the old definition

## Mistakes I made (and what they taught me)

| Mistake | Lesson |
|---|---|
| Forgot `return None` after deleting an expired session | Code keeps running after `pop`; expired session was returned |
| Extended `expires_at` before the expiry check | Check first, then mutate |
| `self.heap[0] <= now` | Heap items are tuples → need `self.heap[0][0]` |
| `"expries_at"` typo → `KeyError` | Read the traceback: the arrow shows the line, the error names the key |
| `_cleanup_loop` indented too deep → `AttributeError` | Class body and method body indentation must be consistent |

## Optional extension: agent version

Store conversation history in the session value and truncate when it grows too long
(simplest form of context window management):
`session_id → {user_id, expires_at, messages: [...]}`
