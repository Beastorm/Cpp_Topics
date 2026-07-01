# 🆕 08 — C++20 Concurrency Features

## 📑 Table of Contents
- [1. std::jthread — Introduction](#1-stdjthread--introduction)
- [2. Building Our Own jthread-Like Wrapper](#2-building-our-own-jthread-like-wrapper)
- [3. C++ Coroutines — Introduction](#3-c-coroutines--introduction)
- [4. Coroutines: Resume Functions](#4-coroutines-resume-functions)
- [5. Coroutines: Generators](#5-coroutines-generators)
- [6. C++ Barriers](#6-c-barriers)

---

## 1. std::jthread — Introduction

Recall from file 01: the #1 beginner bug is forgetting to `join()` a `std::thread`
before it's destroyed → crash. `std::jthread` ("joining thread") fixes this by
**automatically joining in its destructor** — same idea as the hand-rolled
`ThreadGuard` RAII wrapper from file 01, but built into the standard library.

```cpp
void task() { /* ... */ }

void f() {
    std::jthread t(task);
    // no need to call t.join() —
}   // t's destructor calls join() automatically, even on exception!
```

**Bonus feature — built-in cancellation:** `jthread` also carries a
`std::stop_token`, so you can cooperatively ask it to stop early.

```cpp
void worker(std::stop_token st) {
    while (!st.stop_requested()) {
        // do work...
    }
}

std::jthread t(worker);
// ... later ...
t.request_stop();   // worker's loop will see stop_requested() == true and exit
```           // jthread's destructor also auto-requests stop + joins if it goes
              // out of scope while still running

```
std::thread:                std::jthread:
  create → run → MUST join    create → run → auto-joins on destruction
  no built-in cancellation    built-in stop_token for cooperative cancellation
```

| | `std::thread` | `std::jthread` |
|---|---|---|
| Auto-joins on destruction | ❌ | ✅ |
| Built-in cancellation support | ❌ | ✅ (`stop_token`/`stop_source`) |
| Otherwise | Same API | Same API |

**Rule of thumb:** prefer `jthread` over `thread` by default in new C++20 code —
it's strictly safer with no real downside.

---

## 2. Building Our Own jthread-Like Wrapper

Understanding *why* `jthread` works helps cement the RAII pattern — it's really
just `ThreadGuard` (file 01) plus a cancellation flag:

```cpp
class MyJThread {
    std::thread t;
    std::shared_ptr<std::atomic<bool>> stop_flag;
public:
    template<typename F>
    MyJThread(F f) : stop_flag(std::make_shared<std::atomic<bool>>(false)) {
        t = std::thread([f, flag = stop_flag] { f(*flag); });
    }

    void request_stop() { *stop_flag = true; }

    ~MyJThread() {
        request_stop();          // ask nicely to stop
        if (t.joinable()) t.join();   // then wait, like ThreadGuard
    }
};
```
This mirrors the real `std::jthread`'s two combined guarantees: **auto-join** +
**cooperative cancellation**, just with a simpler `atomic<bool>` instead of the
full `stop_token`/`stop_source` machinery.

---

## 3. C++ Coroutines — Introduction

💡 **In plain English:** think of watching a video and hitting **pause**. The
video remembers exactly where you left off — which frame, which point in the
audio — and when you hit **play** again, it picks up from exactly that spot, not
from the beginning. A **coroutine** is a function that can do the same thing:
pause partway through, remember exactly where it was (including all its local
variables), and pick back up later — all without needing a separate chef/thread
to babysit it while it's paused.

More precisely: a **coroutine** is a function that can **pause its execution and
resume later**, keeping all its local state intact in between — without needing
a separate OS thread. This is fundamentally different from `std::thread`: coroutines are about
**cooperative suspension on one thread**, not parallel execution across cores.

```
Normal function:     call → run to completion → return
Coroutine:            call → run... → SUSPEND (co_await/co_yield) → [caller does other things]
                                            │
                                       later: RESUME → run more... → SUSPEND again → ... → return
```

A function becomes a coroutine simply by using one of three new keywords inside it:
- `co_await` — suspend until something else is ready
- `co_yield` — suspend, handing a value back to the caller (used for generators)
- `co_return` — return a final value and finish

```cpp
#include <coroutine>

// (simplified — real coroutines need a "promise type" plumbed in; shown conceptually)
task<int> example() {
    std::cout << "start\n";
    co_await some_async_operation();   // suspends HERE, resumes later
    std::cout << "resumed!\n";
    co_return 42;
}
```

**Why this matters for concurrency:** coroutines let you write code that *looks*
sequential ("do this, then this, then this") but actually suspends during waits
(I/O, timers) instead of blocking a whole OS thread — one thread can juggle
thousands of suspended coroutines cheaply, similar in spirit to the N:1/M:N
user-thread models from file 01, but implemented at the language level.

---

## 4. Coroutines: Resume Functions

When a coroutine suspends, **someone** has to call `.resume()` on it later — that
"someone" is typically an event loop, a scheduler, or a future-like handle.

```cpp
struct SimpleTask {
    struct promise_type {
        SimpleTask get_return_object() { return {}; }
        std::suspend_never initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() {}
    };
};

SimpleTask example_coroutine() {
    std::cout << "before suspend\n";
    co_await std::suspend_always{};    // suspends here
    std::cout << "after resume\n";
}
```

```
handle = example_coroutine();     // runs until first suspend point, prints "before suspend"
// ... caller does other work ...
handle.resume();                  // resumes from suspend point, prints "after resume"
```

`std::suspend_always` / `std::suspend_never` are the simplest "awaitable" types —
they control whether the coroutine pauses at that point or just sails through.
Real async frameworks build custom awaitable types that resume the coroutine
automatically once an I/O operation completes.

---

## 5. Coroutines: Generators

A **generator** is a coroutine that produces a *sequence* of values over time,
using `co_yield`, instead of computing them all up front. Great for infinite or
lazily-evaluated sequences.

```cpp
#include <generator>   // C++23 has std::generator; C++20 needs a hand-rolled version

std::generator<int> counting_generator() {
    for (int i = 0; ; ++i)
        co_yield i;          // suspend, hand back `i`, resume here on next request
}

// usage:
for (int value : counting_generator()) {
    if (value > 5) break;
    std::cout << value << " ";   // 0 1 2 3 4 5
}
```

```
Caller asks for next value
      │
Generator RUNS until co_yield → hands back a value → SUSPENDS
      │
Caller uses the value, then asks for next
      │
Generator RESUMES right after the co_yield, continues the loop → next co_yield
```

**Why this beats a hand-built iterator class:** without coroutines, writing a
custom lazy sequence means manually saving/restoring "where was I" state in
member variables. A generator coroutine lets the compiler do that bookkeeping
for you — the loop's local variables (like `i` above) are automatically preserved
across suspensions.

---

## 6. C++ Barriers

A **barrier** is a synchronization point where a **fixed group of threads** all
wait until *everyone* in the group has arrived — then they're all released
together, and the barrier can be reused for the next round.

```
Thread A ──arrive_and_wait()──┐
Thread B ──arrive_and_wait()──┼── ALL WAIT until every thread has called this
Thread C ──arrive_and_wait()──┘
                    │
      once all 3 have arrived → ALL released simultaneously → next phase begins
```

```cpp
#include <barrier>

std::barrier sync_point(3);   // expects exactly 3 threads per round

void worker(int id) {
    std::cout << "Phase 1: thread " << id << "\n";
    sync_point.arrive_and_wait();     // waits here until all 3 threads reach this line

    std::cout << "Phase 2: thread " << id << "\n";   // nobody starts Phase 2 early
    sync_point.arrive_and_wait();     // barrier is reusable for another round
}
```

**Difference vs a condition variable / latch:**

| Primitive | Reusable? | Use case |
|---|---|---|
| `std::latch` (C++20) | ❌ One-shot — counts down once, then stays "open" forever | "Wait until N things have happened" (e.g. wait for N startup tasks) |
| `std::barrier` (C++20) | ✅ Yes — resets automatically for the next round | Repeated phases where a fixed group must all sync up before each phase (e.g. simulation steps, parallel loop iterations) |
| `condition_variable` | ✅ Yes, but manual bookkeeping | General-purpose wait/notify for arbitrary conditions |

Barriers are common in **simulation** and **iterative parallel algorithms** —
e.g. a physics simulation where every thread must finish updating "this frame"
before anyone starts reading data for "the next frame."

---

## ✅ Quick Recap

- `std::jthread` = `std::thread` + automatic join on destruction + built-in cooperative cancellation via `stop_token` — prefer it over plain `thread` in new code.
- Coroutines let a function suspend and resume without blocking an OS thread — `co_await` (suspend for something), `co_yield` (suspend, hand back a value), `co_return` (finish with a value).
- Generators use `co_yield` to lazily produce a sequence, with the compiler automatically preserving local state across suspensions.
- `std::barrier` synchronizes a fixed group of threads at repeated checkpoints, releasing everyone together each round; `std::latch` is the one-shot version.

**Next:** `09_interview_quickfire.md` — rapid-fire answers to common concurrency interview questions.
