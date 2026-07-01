# 🔒 02 — Mutex & Locking Mechanisms

## 📑 Table of Contents
- [1. Critical Sections & Invariants](#1-critical-sections--invariants)
- [2. Critical Section vs Lock — Don't Confuse the Terms](#2-critical-section-vs-lock--dont-confuse-the-terms)
- [3. Mutex — the basic lock](#3-mutex--the-basic-lock)
- [4. Things to Remember When Using Mutexes](#4-things-to-remember-when-using-mutexes)
- [5. try_lock](#5-try_lock)
- [6. Timed Mutex](#6-timed-mutex)
- [7. Recursive Mutex](#7-recursive-mutex)
- [8. Shared Mutex (Read-Write Lock)](#8-shared-mutex-read-write-lock)
- [9. std::lock — Locking Multiple Mutexes Safely](#9-stdlock--locking-multiple-mutexes-safely)
- [10. lock_guard](#10-lock_guard)
- [11. unique_lock](#11-unique_lock)
- [12. shared_lock](#12-shared_lock)
- [13. scoped_lock (C++17)](#13-scoped_lock-c17)
- [14. Deadlock](#14-deadlock)
- [15. Livelock](#15-livelock)
- [16. Thread-Safe Stack Example](#16-thread-safe-stack-example)
- [17. Lock-Based vs Lock-Free — The Big Picture](#17-lock-based-vs-lock-free--the-big-picture)

---

## 1. Critical Sections & Invariants

💡 **In plain English:** imagine a single shared bathroom (your shared data) in
that kitchen from file 01. If two chefs try to use it at the exact same time,
things go wrong. The "critical section" is just the *time spent inside* the
bathroom — the moment where things could go wrong if someone else barges in.

An **invariant** is a rule that must always hold true about your data — e.g. "the
`size` field always equals the actual number of elements in the list."

A **critical section** is the piece of code that temporarily **breaks** the
invariant while it's mid-update (e.g. adding a node before bumping `size`). If a
second thread reads the data *during* that window, it sees a broken, inconsistent
state → **race condition**.

```
Thread A:  [ start update ] ---- invariant BROKEN ---- [ finish update ]
                                       ▲
Thread B:                    reads data HERE → sees garbage
```

**The fix:** make sure only *one* thread can be inside the critical section at a
time. That's exactly what a mutex gives you.

---

## 2. Critical Section vs Lock — Don't Confuse the Terms

These two words get used almost interchangeably in casual conversation, but they
mean **different things** — and mixing them up makes deadlock/race-condition
reasoning harder. This distinction matters just as much when comparing this whole
file (lock-**based**) against file 05 (lock-**free**).

| Term | What it actually is | Analogy |
|---|---|---|
| **Critical section** | The **piece of code** (the "what") that must never run on two threads at the same time, because it touches shared, mutable data | The bathroom itself |
| **Lock** | The **mechanism** (the "how") used to enforce that rule — a mutex, or an RAII wrapper around one | The bathroom door lock |

```
                 ┌────────────── critical section ──────────────┐
mutex.lock();    │  shared_data.update();                        │   mutex.unlock();
   ▲             └────────────────────────────────────────────────┘        ▲
   │                                                                        │
 the LOCK "goes on"                                              the LOCK "comes off"
 (only one thread admitted)                                    (another thread may now enter)
```

- The **critical section** is a property of your *code* — it exists whether or not
  you remember to protect it. Forgetting to lock it doesn't make it stop being a
  critical section; it just means it's an *unprotected* one (a race condition
  waiting to happen).
- The **lock** is the *tool* you apply around a critical section to actually
  enforce "only one thread at a time."

**One more layer of naming to untangle, specific to C++:** in this file, "lock"
also refers to a group of **wrapper objects** — `lock_guard`, `unique_lock`,
`shared_lock`, `scoped_lock` (sections 10–13). These follow a C++ pattern called
**RAII** ("Resource Acquisition Is Initialization" — a fancy name for a simple
idea: *"grab the resource when the object is created, release it automatically
when the object is destroyed."*). Concretely, that means: the moment one of these
wrapper objects is created, it locks the mutex for you; the moment it goes out of
scope (the function ends, or an exception is thrown), it automatically unlocks —
so you never have to remember to call `unlock()` yourself, and you can't forget it
even by accident.

These wrapper objects are *not* separate synchronization primitives — they're all
just convenience wrappers that call `mutex.lock()`/`mutex.unlock()` for you
automatically. The actual synchronization primitive underneath all of them is
still the `std::mutex` (or `std::shared_mutex`) itself.

```
std::mutex              ← the REAL primitive (owns the lock/unlock state)
     ▲
     │ wrapped by
     │
lock_guard / unique_lock / shared_lock / scoped_lock   ← RAII convenience objects
                                                           around that ONE mutex
```

Keep this straight and the rest of the file reads more precisely: "protect this
critical section with a lock_guard around this mutex" — three distinct concepts,
one sentence.

---

## 3. Mutex — the Basic Lock

**Mutex** = **MUT**ual **EX**clusion. It's a lock: only one thread can hold it at
a time; everyone else waits.

```cpp
#include <mutex>
std::mutex m;
int shared_counter = 0;

void increment() {
    m.lock();
    shared_counter++;      // critical section
    m.unlock();
}
```

```
Thread A ──lock()──► [critical section] ──unlock()──►
Thread B ──lock()── (waits) ──────────────────────────► [critical section] ──unlock()──►
```

If Thread B calls `lock()` while A already holds it, B **blocks** until A calls
`unlock()`.

---

## 4. Things to Remember When Using Mutexes

| ✅ Do | ❌ Don't |
|---|---|
| Lock the *smallest* scope possible | Do heavy/slow work while holding the lock |
| Always pair every `lock()` with an `unlock()` — prefer RAII wrappers | Call `.lock()` manually and risk forgetting `.unlock()` on an exception path |
| Never return a pointer/reference to protected data | Let a raw pointer into your protected data escape the locked function |
| Keep the same locking order everywhere | Lock mutexes in different orders in different functions (→ deadlock) |

> The most common real bug: a function locks `m`, then calls another function that
> *also* tries to lock `m` → deadlocks against itself. Always know exactly which
> mutex protects what, and never call a "locking" function while already holding
> that same mutex.

---

## 5. try_lock

`try_lock()` **attempts** to acquire the lock but returns immediately (no waiting)
— returns `true` if it succeeded, `false` if the mutex was already held by someone
else.

```cpp
if (m.try_lock()) {
    // got the lock
    shared_counter++;
    m.unlock();
} else {
    // someone else has it — do something else instead of waiting
}
```

**Use case:** avoid blocking a thread that has other useful work to do if the lock
isn't immediately free (e.g. a UI thread that shouldn't freeze).

---

## 6. Timed Mutex

`std::timed_mutex` adds a **deadline** to waiting:

```cpp
std::timed_mutex tm;

if (tm.try_lock_for(std::chrono::milliseconds(100))) {
    // got it within 100ms
    tm.unlock();
} else {
    // gave up waiting after 100ms
}
```

Also has `try_lock_until(time_point)` for an absolute deadline instead of a duration.

---

## 7. Recursive Mutex

A normal mutex **cannot** be locked twice by the *same* thread — doing so deadlocks
(the thread waits for itself forever). `std::recursive_mutex` allows the **same
thread** to lock it multiple times (it must unlock the same number of times).

```cpp
std::recursive_mutex rm;

void inner() { rm.lock(); /* ... */ rm.unlock(); }
void outer() {
    rm.lock();
    inner();     // same thread locks rm AGAIN — OK because it's recursive
    rm.unlock();
}
```

⚠️ Usually a sign of design smell (functions calling each other while both lock the
same mutex) — prefer restructuring so only one layer locks, but the tool exists for
legitimate recursive algorithms.

---

## 8. Shared Mutex (Read-Write Lock)

💡 **In plain English:** a normal mutex is like a bathroom with one key — only
one person in, no matter what they're doing. But if everyone is just *looking* at
a public notice board (reading), there's no reason to make them queue up one at a
time. Only when someone wants to *change* what's on the board (writing) do you
need to clear everyone else out.

Sometimes many threads just want to **read** shared data, and only occasionally does
someone **write**. Locking everyone out for reads is wasteful. `std::shared_mutex`
(C++17) supports two lock modes:

```
Many readers at once:      R1 ──┐
                            R2 ──┼── all allowed simultaneously
                            R3 ──┘
One writer, exclusive:      W ── blocks EVERYONE (readers + other writers)
```

```cpp
#include <shared_mutex>
std::shared_mutex sm;
int data = 0;

void reader() {
    std::shared_lock lock(sm);   // shared/read lock — multiple readers OK
    std::cout << data;
}

void writer() {
    std::unique_lock lock(sm);   // exclusive/write lock — nobody else allowed
    data++;
}
```

| Lock mode | Who else can lock? |
|---|---|
| Shared (read) | Other readers ✅, writers ❌ |
| Exclusive (write) | Nobody ❌ |

---

## 9. std::lock — Locking Multiple Mutexes Safely

Locking two mutexes one at a time is dangerous:
```cpp
m1.lock();
m2.lock();   // 💥 if another thread does m2.lock(); m1.lock(); → DEADLOCK
```

`std::lock(m1, m2)` locks **both atomically as a group**, internally using a
deadlock-avoidance algorithm — no matter what order two threads call it in, neither
will deadlock.

```cpp
std::mutex m1, m2;

void transfer() {
    std::lock(m1, m2);              // locks both, safely, in any call order
    std::lock_guard<std::mutex> g1(m1, std::adopt_lock);
    std::lock_guard<std::mutex> g2(m2, std::adopt_lock);
    // ... critical section touching both ...
}   // both unlocked automatically
```
(`std::adopt_lock` tells `lock_guard` "this mutex is already locked, just manage
unlocking it for me.")

---

## 10. lock_guard

The simplest RAII lock wrapper: locks on construction, unlocks on destruction.
No manual unlock, no way to forget — even if an exception is thrown mid-function.

```cpp
void increment() {
    std::lock_guard<std::mutex> lock(m);
    shared_counter++;
}   // lock automatically released here, even on exception
```

**Limitation:** cannot unlock early, cannot be locked/unlocked more than once,
cannot be moved.

---

## 11. unique_lock

Like `lock_guard`, but more flexible: can be unlocked and re-locked, can defer
locking, can be moved, and works with `std::lock`, condition variables, etc.

```cpp
std::unique_lock<std::mutex> lock(m);   // locks immediately
// ... critical section ...
lock.unlock();                          // can release early!
// ... non-critical work ...
lock.lock();                            // can re-lock later
```

Deferred locking:
```cpp
std::unique_lock<std::mutex> lock(m, std::defer_lock);  // NOT locked yet
// do other setup...
lock.lock();   // lock only when actually needed
```

| | `lock_guard` | `unique_lock` |
|---|---|---|
| Can unlock early | ❌ | ✅ |
| Can defer initial lock | ❌ | ✅ |
| Movable | ❌ | ✅ |
| Works with condition_variable::wait | ❌ | ✅ (required) |
| Overhead | Slightly lower | Slightly higher (extra bookkeeping) |

---

## 12. shared_lock

The read-side RAII wrapper for `std::shared_mutex` (shown in section 8) — same
constructor patterns as `unique_lock`, but acquires the **shared/read** lock
instead of exclusive.

```cpp
std::shared_lock<std::shared_mutex> lock(sm);   // read lock, other readers OK too
```

---

## 13. scoped_lock (C++17)

Combines the safety of `std::lock()` with RAII — lock **any number** of mutexes
deadlock-free, in one line:

```cpp
std::scoped_lock lock(m1, m2, m3);   // locks all three, deadlock-avoidant, RAII
```

This replaces the more verbose `std::lock` + two `lock_guard(..., adopt_lock)` dance
from section 9. **Prefer `scoped_lock` whenever locking 2+ mutexes together.**

---

## 14. Deadlock

💡 **In plain English:** imagine two chefs — Chef A is holding the salt and
waiting for the pepper, while Chef B is holding the pepper and waiting for the
salt. Neither will let go of what they're holding. They'll both stand there
forever. That's a deadlock.

Two (or more) threads each hold a lock the other one needs — **nobody can proceed**.

```
Thread A: locks M1 ──► wants M2 (blocked, B has it)
Thread B: locks M2 ──► wants M1 (blocked, A has it)

     A ──waits for──► M2 ──held by──► B
     ▲                                │
     └───────────waits for M1─────────┘
              (circular wait = deadlock)
```

**Classic causes:**
- Locking two mutexes in different order in different functions.
- A function that locks `M` calling another function that also tries to lock `M`.

**Prevention:**
| Technique | How it helps |
|---|---|
| Always lock in the same global order | Removes circular wait possibility |
| Use `std::lock` / `std::scoped_lock` for multi-mutex locking | Handles ordering internally |
| Avoid nested locks entirely if possible | Fewer locks = fewer chances to deadlock |
| Use a lock hierarchy (only lock "lower level" locks while holding "higher level" ones) | Enforces consistent ordering by design |

---

## 15. Livelock

Similar to deadlock, but the threads are **not blocked** — they're actively running,
repeatedly trying and backing off, but **still make no real progress**. Classic
analogy: two people in a hallway, both step aside to let the other pass, at the
exact same time, forever.

```
Thread A: try_lock fails → backs off → retries → try_lock fails → ...
Thread B: try_lock fails → backs off → retries → try_lock fails → ...
(both are "busy," CPU usage is high, but neither ever succeeds)
```

**Fix:** add randomized backoff delays, or fall back to a real blocking `lock()`
after a few failed attempts instead of retrying forever in lockstep.

---

## 16. Thread-Safe Stack Example

A plain `std::stack` is **not** thread-safe — even calling `empty()` then `top()`
as two separate calls is racy (another thread could pop the last element in
between). A thread-safe wrapper locks around the *entire logical operation*:

```cpp
template<typename T>
class ThreadSafeStack {
    std::stack<T> data;
    mutable std::mutex m;
public:
    void push(T value) {
        std::lock_guard<std::mutex> lock(m);
        data.push(std::move(value));
    }

    std::shared_ptr<T> pop() {
        std::lock_guard<std::mutex> lock(m);
        if (data.empty()) return nullptr;      // check + act, atomically together
        auto result = std::make_shared<T>(std::move(data.top()));
        data.pop();
        return result;
    }
};
```

> ⚠️ **Race condition inherited from the interface:** even a "thread-safe" stack can
> still be misused racily — e.g. calling `empty()` and then `pop()` as two separate
> locked calls still has a race window between them, because another thread's `pop()`
> can run in between. That's why `pop()` above checks `empty()` **and** removes the
> element **inside the same lock** — combine "check" and "act" into one atomic
> operation whenever the two are logically related.

---

## 17. Lock-Based vs Lock-Free — The Big Picture

Everything in this file (sections 3–16) is a **lock-based** approach to protecting
a critical section: a thread that can't get in **blocks/sleeps** until the mutex is
free. File 05 covers the opposite philosophy — **lock-free** structures, where a
thread that "loses the race" simply **retries a CAS operation** instead of sleeping.
Same underlying goal (safe concurrent access to shared data), fundamentally
different mechanism:

```
Lock-based (this file):                Lock-free (file 05):

Thread A: mutex.lock() ──► critical    Thread A: read → compute → CAS ──► success, done
           section ──► unlock()                                    └──► fail, RETRY with fresh read
Thread B: mutex.lock() ──► BLOCKS,     Thread B: read → compute → CAS ──► never sleeps,
           OS puts it to sleep,                                          just spins/retries
           until A unlocks
```

| | Lock-based (mutex, this file) | Lock-free (atomics/CAS, file 05) |
|---|---|---|
| What happens if you "lose"? | Thread **blocks/sleeps** (OS context switch) | Thread **retries** the CAS loop (stays runnable) |
| Deadlock possible? | ✅ Yes, if locking order isn't managed | ❌ No lock to deadlock on |
| Progress guarantee | None while blocked — you wait as long as the holder takes | At least one thread always makes progress (that's the definition of lock-free, file 05 §1) |
| Ease of writing correctly | Easier — lock, do work, unlock | Much harder — must handle retries, torn reads, and memory reclamation carefully |
| Overhead source | OS-level scheduling/context switches when contended | CPU-level CAS retries; no OS involvement |
| Best for | General-purpose code, longer critical sections | Very short operations, high contention, avoiding stalls from a blocked/killed thread holding a lock |

**One-line summary:** a **lock** stops other threads from entering by making them
*wait*; **lock-free** code lets every thread keep running and just *try again* if
its update didn't land — trading simpler code (locks) for stronger progress
guarantees and no blocking (lock-free).

---

## ✅ Quick Recap

- Critical section = the *code* that must run exclusively; lock = the *mechanism* (mutex + RAII wrapper) that enforces it — don't conflate the two.
- `try_lock` / `timed_mutex` let you avoid blocking forever; `recursive_mutex` lets the same thread re-lock; `shared_mutex` lets many readers in but only one writer.
- Never lock manually with `.lock()`/`.unlock()` — use RAII: `lock_guard` (simple), `unique_lock` (flexible), `shared_lock` (read-side), `scoped_lock` (multiple mutexes, C++17).
- Locking 2+ mutexes → always use `std::lock`/`std::scoped_lock` to avoid deadlock.
- Deadlock = circular waiting, threads blocked forever. Livelock = threads keep running but make no progress.
- Design thread-safe wrappers so "check" and "act" happen inside a *single* lock, not two separate locked calls.
- Lock-based (this file) trades simplicity for blocking risk; lock-free (file 05) trades harder code for guaranteed progress and no OS-level blocking.

**Next:** `03_condition_variables_and_futures.md` — how threads *wait* for and *signal* each other, and how to get a return value back from a thread.
