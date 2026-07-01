# ⚡ 09 — Interview Quick-Fire Q&A

Rapid answers to the concurrency questions that don't need a whole file to
themselves — each links back to the fuller explanation elsewhere in these notes.

## 📑 Table of Contents
- [1. Race Condition](#1-race-condition)
- [2. Sleep vs Wait](#2-sleep-vs-wait)
- [3. Mutex vs Semaphore](#3-mutex-vs-semaphore)
- [4. Binary Semaphore](#4-binary-semaphore)
- [5. Producer-Consumer Using a Semaphore](#5-producer-consumer-using-a-semaphore)
- [6. Is a Static Variable Thread-Safe?](#6-is-a-static-variable-thread-safe)
- [7. Deadlock vs Livelock](#7-deadlock-vs-livelock)
- [8. Oversubscription](#8-oversubscription)
- [9. Thread or Process Synchronization](#9-thread-or-process-synchronization)
- [10. Thread Pool — One-Liner](#10-thread-pool--one-liner)
- [11. Object Pool in Multithreading](#11-object-pool-in-multithreading)
- [12. Atomic Variables — One-Liner](#12-atomic-variables--one-liner)

---

## 1. Race Condition

**Two or more threads access shared data at the same time, and the final result
depends on the unpredictable order/timing of execution.** Fixed with mutexes
(file 02), atomics (file 04), or by avoiding shared mutable state altogether.

```
Thread A: read x, +1, write x
Thread B: read x, +1, write x     ← if these interleave, one increment is LOST
```

---

## 2. Sleep vs Wait

| | `sleep_for(duration)` | `condition_variable::wait()` |
|---|---|---|
| Wakes up when... | The **time** elapses, period | **Notified**, or spuriously (must re-check predicate) |
| Holds a lock while paused? | N/A (doesn't use a mutex) | ❌ No — releases the mutex while asleep, re-acquires on wake |
| Use case | "Pause for a fixed amount of time" | "Wait until some *condition* becomes true" |

**Common mistake:** using `sleep_for()` in a loop to "poll" for a condition
instead of a proper `condition_variable` — wastes CPU and adds artificial latency
(you're always waiting *at least* the sleep duration, even if the condition became
true immediately). See file 03 for the proper pattern.

---

## 3. Mutex vs Semaphore

| | Mutex | Semaphore |
|---|---|---|
| Represents | Ownership of a single resource | A **count** of available resources/permits |
| Who can release it | Only the thread that locked it (ownership) | Any thread (no ownership concept) |
| Values | Locked / unlocked (binary, essentially) | Any non-negative integer count |
| Typical use | Protecting a critical section | Limiting concurrent access to N instances of a resource (e.g. a connection pool of size 5) |

```
Mutex:      🔒 (locked by thread A) — only thread A may unlock it
Semaphore:  count = 3  → 3 threads may proceed concurrently, decrementing count each time,
                          incrementing when done — ANY thread can increment/decrement
```

`std::mutex` is C++11; `std::counting_semaphore<N>` / `std::binary_semaphore` are
C++20 additions (`<semaphore>`).

---

## 4. Binary Semaphore

A semaphore whose count is restricted to **0 or 1** — conceptually similar to a
mutex, but **without ownership**: any thread can "release" (increment) it, not
just the thread that "acquired" (decremented) it. That makes it suited to
**signaling** between threads (like a condition variable's notify), not just
mutual exclusion.

```cpp
#include <semaphore>
std::binary_semaphore sig(0);   // starts "empty" (0)

void waiter() {
    sig.acquire();               // blocks until count > 0
    std::cout << "Signaled!\n";
}
void signaler() {
    sig.release();                // any thread can do this — no ownership check
}
```

---

## 5. Producer-Consumer Using a Semaphore

Same problem as file 03's condition-variable version, solved with two counting
semaphores instead: one tracks "how many empty slots" in the buffer, the other
tracks "how many filled slots."

```
empty_slots (semaphore, starts = BUFFER_SIZE)
filled_slots (semaphore, starts = 0)

Producer: acquire(empty_slots) → write item → release(filled_slots)
Consumer: acquire(filled_slots) → read item  → release(empty_slots)
```

```cpp
std::counting_semaphore<10> empty_slots(10);   // buffer capacity 10
std::counting_semaphore<10> filled_slots(0);
std::mutex buffer_mutex;                       // still needed to protect the buffer itself
std::queue<int> buffer;

void producer(int item) {
    empty_slots.acquire();                      // wait for a free slot
    {
        std::lock_guard<std::mutex> lock(buffer_mutex);
        buffer.push(item);
    }
    filled_slots.release();                     // signal: one more item available
}

void consumer() {
    filled_slots.acquire();                      // wait for an item
    int item;
    {
        std::lock_guard<std::mutex> lock(buffer_mutex);
        item = buffer.front();
        buffer.pop();
    }
    empty_slots.release();                        // signal: one more free slot
}
```

**Why still need a mutex?** The semaphores control *how many* producers/consumers
may proceed, but the actual `push`/`pop` on the shared `queue` is still a critical
section that needs mutual exclusion — semaphores and mutexes solve *different*
problems and are often combined.

---

## 6. Is a Static Variable Thread-Safe?

(Full detail in file 04, section 6.) Short answer:
- **First initialization** of a static local variable (C++11+): ✅ thread-safe by
  the standard.
- **Reads/writes to its value afterward**: ❌ not automatically thread-safe —
  needs a mutex/atomic like any other shared mutable data.

---

## 7. Deadlock vs Livelock

(Full detail in file 02, sections 13–14.)
- **Deadlock:** threads are **blocked**, waiting on each other in a cycle — nobody
  proceeds, ever.
- **Livelock:** threads are **actively running**, repeatedly retrying/backing off
  in a way that still results in **no real progress**.

---

## 8. Oversubscription

(Full detail in file 06, section 2.) Running **more active threads than CPU
cores** — the OS must context-switch between them constantly, and beyond a point,
adding threads makes total throughput *worse*, not better, due to context-switch
and cache-locality overhead.

---

## 9. Thread or Process Synchronization

**Thread synchronization** = coordinating threads *within the same process*
(they already share memory) — tools: mutex, condition_variable, atomics, barriers
(all covered in files 02–04, 08).

**Process synchronization** = coordinating *separate processes* (they do **not**
share memory by default) — needs OS-level, cross-process primitives: named
semaphores, named mutexes, shared memory + a synchronization primitive placed
inside that shared memory, or file locks / pipes for signaling.

| | Thread sync | Process sync |
|---|---|---|
| Shared memory by default? | ✅ Yes | ❌ No |
| Typical tools | `std::mutex`, `condition_variable`, atomics | named semaphores, shared memory + sync primitive, pipes/sockets |
| Overhead | Lower (same address space) | Higher (kernel-mediated, often involves IPC — see your AOSP IPC notes for file-based IPC, sockets, etc.) |

---

## 10. Thread Pool — One-Liner

**A fixed set of worker threads created once, reused across many tasks pulled
from a shared queue** — avoids the repeated cost of creating/destroying threads
per task, and lets you size concurrency to match `hardware_concurrency()` instead
of task count. Full detail: file 06.

---

## 11. Object Pool in Multithreading

**A fixed set of pre-constructed, reusable objects** (buffers, connections)
borrowed and returned by worker threads, instead of constructing/destructing a
new object per task. The borrow/return operations themselves need
synchronization (mutex or lock-free free-list) since multiple threads share the
pool. Full detail: file 06, section 7.

---

## 12. Atomic Variables — One-Liner

**Operations that are indivisible — no other thread can ever observe them "half
done."** `std::atomic<T>` gives lock-free (on most platforms) reads/writes/CAS on
a single variable, cheaper than a mutex for simple counters/flags, and is the
foundation for all lock-free data structures. Full detail: file 04.

---

## ✅ One-Page Cheat Sheet

| Concept | One-line definition |
|---|---|
| Race condition | Shared data + unsynchronized concurrent access → unpredictable result |
| Deadlock | Threads blocked forever, waiting on each other in a cycle |
| Livelock | Threads keep running but make no real progress |
| Mutex | Exclusive lock, only the locker can unlock |
| Semaphore | A count of permits; any thread can release |
| Binary semaphore | Semaphore capped at 0/1, no ownership (unlike mutex) |
| Oversubscription | More active threads than cores → net slowdown |
| Thread pool | Reuse a fixed set of workers instead of spawning per task |
| Object pool | Reuse a fixed set of expensive objects instead of constructing per task |
| Atomic variable | Indivisible operation, no mutex needed for simple cases |
