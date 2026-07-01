# 🔔 03 — Condition Variables, Promises & Futures

## 📑 Table of Contents
- [1. Why Mutexes Alone Aren't Enough](#1-why-mutexes-alone-arent-enough)
- [2. Condition Variables — Basics](#2-condition-variables--basics)
- [3. Producer-Consumer with a Condition Variable](#3-producer-consumer-with-a-condition-variable)
- [4. Thread-Safe Queue](#4-thread-safe-queue)
- [5. std::promise and std::future](#5-stdpromise-and-stdfuture)
- [6. std::async](#6-stdasync)
- [7. std::packaged_task](#7-stdpackaged_task)
- [8. Retrieving Exceptions Through a Future](#8-retrieving-exceptions-through-a-future)
- [9. std::shared_future](#9-stdshared_future)

---

## 1. Why Mutexes Alone Aren't Enough

A mutex protects data, but it doesn't answer: **"how does a thread wait until some
*condition* becomes true, without wasting CPU?"**

**Bad approach — busy-waiting / polling:**
```cpp
while (!dataReady) { /* spin, wasting 100% CPU */ }
```
This burns a full CPU core doing nothing useful. We need a way to **sleep until
notified**.

---

## 2. Condition Variables — Basics

💡 **In plain English:** instead of a chef repeatedly opening the oven door every
few seconds to check "is the cake done yet?" (wasting time and energy), imagine
the oven has a **bell** that dings when it's ready. The chef can go do something
else and simply wait for the bell. A `std::condition_variable` **is** that bell —
it lets a thread fall asleep and be woken up only when there's actually something
to do.

A `std::condition_variable` lets a thread **sleep** until another thread
**notifies** it. It always works together with a `std::mutex` and a
`std::unique_lock`.

```
Waiting thread:  lock() → check condition → not ready → SLEEP (releases lock while asleep)
                                                              │
Notifying thread:                    lock() → change data → notify() → unlock()
                                                              │
Waiting thread:            WAKES UP → re-acquires lock → re-checks condition → proceeds
```

```cpp
std::mutex m;
std::condition_variable cv;
bool ready = false;

void waiter() {
    std::unique_lock<std::mutex> lock(m);
    cv.wait(lock, [] { return ready; });   // sleeps until ready == true
    std::cout << "Proceeding!\n";
}

void setter() {
    {
        std::lock_guard<std::mutex> lock(m);
        ready = true;
    }
    cv.notify_one();   // wake up one waiting thread
}
```

**Why `wait()` needs a predicate (the lambda):** when a thread wakes up, it must
**re-check** the condition — because of *spurious wakeups* (the OS is allowed to
wake a waiting thread even without a real `notify()`), and because multiple threads
might be woken but only some should actually proceed. The lambda form
`cv.wait(lock, predicate)` automatically loops: "sleep → wake → check predicate →
if false, sleep again."

| Call | Wakes up... |
|---|---|
| `cv.notify_one()` | One arbitrary waiting thread |
| `cv.notify_all()` | Every waiting thread |

---

## 3. Producer-Consumer with a Condition Variable

The classic use case: a **producer** thread adds items to a shared queue, a
**consumer** thread waits for items to appear and processes them.

```
Producer ──push──► [ shared queue ] ──notify──►
                                                 Consumer wakes, wait() returns,
                                                 pops item, processes it
```

```cpp
std::mutex m;
std::condition_variable cv;
std::queue<int> q;

void producer() {
    for (int i = 0; i < 5; ++i) {
        {
            std::lock_guard<std::mutex> lock(m);
            q.push(i);
        }
        cv.notify_one();   // tell consumer something arrived
    }
}

void consumer() {
    for (int i = 0; i < 5; ++i) {
        std::unique_lock<std::mutex> lock(m);
        cv.wait(lock, [] { return !q.empty(); });   // sleep until queue has data
        int value = q.front();
        q.pop();
        lock.unlock();
        std::cout << "Consumed: " << value << "\n";
    }
}
```

---

## 4. Thread-Safe Queue

Wrapping the pattern above into a reusable class (same idea as the thread-safe
stack from file 02 — combine "check" and "act" under one lock):

```cpp
template<typename T>
class ThreadSafeQueue {
    std::queue<T> data;
    mutable std::mutex m;
    std::condition_variable cv;
public:
    void push(T value) {
        std::lock_guard<std::mutex> lock(m);
        data.push(std::move(value));
        cv.notify_one();
    }

    void wait_and_pop(T& out) {
        std::unique_lock<std::mutex> lock(m);
        cv.wait(lock, [this] { return !data.empty(); });
        out = std::move(data.front());
        data.pop();
    }

    bool try_pop(T& out) {
        std::lock_guard<std::mutex> lock(m);
        if (data.empty()) return false;
        out = std::move(data.front());
        data.pop();
        return true;
    }
};
```

This is the backbone of a **thread pool's task queue** (file 06).

---

## 5. std::promise and std::future

💡 **In plain English:** think of ordering food at a restaurant counter. You get
handed a little paper ticket with a number on it (that's your **future** — a
placeholder for a result you don't have yet). Meanwhile, the kitchen is cooking
your order — when it's ready, they attach the actual food to your ticket number
(that's the kitchen fulfilling the **promise**). You can go sit down and wait, and
when you finally check your ticket (`future.get()`), you either get your food, or
if something went wrong, you find out about that instead.

Condition variables are great for signaling "something happened," but what if you
want to **send a value** (or an exception) from one thread to another? That's what
`promise`/`future` are for.

```
Thread A: std::promise<int> p
              │
              ▼ p.get_future() ──► gives you a std::future<int>
              │
Thread B (worker): p.set_value(42)   ← "fills" the promise
              │
Thread A: future.get()  ← BLOCKS until value is set, then returns 42
```

```cpp
void worker(std::promise<int> p) {
    // do some work...
    p.set_value(42);          // deliver the result
}

int main() {
    std::promise<int> p;
    std::future<int> f = p.get_future();

    std::thread t(worker, std::move(p));   // promise moved into the thread
    std::cout << f.get();                  // blocks until set_value() is called
    t.join();
}
```

- `future.get()` can only be called **once** per future — it consumes the value.
- `future.wait()` blocks without retrieving the value (just waits for readiness).
- `future.wait_for(duration)` / `wait_until(time_point)` — waits with a timeout,
  returns a status (`ready`, `timeout`, `deferred`).

---

## 6. std::async

💡 **In plain English:** section 5 was the "long way" of getting a value back
from another thread — manually creating a promise, manually creating a thread,
manually filling in the promise. `std::async` is the "short way": you just say
"go run this function for me," and it automatically hands you back a ticket
(`future`) for the result, without you wiring up the plumbing yourself.

Manually wiring up `promise` + `thread` for every task is repetitive.
`std::async` does it for you — runs a function (maybe on a new thread, maybe later
on this thread) and hands you a `future` for its return value directly.

```cpp
int compute() { return 42; }

std::future<int> f = std::async(compute);
// ... do other work while compute() might be running in parallel ...
std::cout << f.get();   // 42
```

| Launch policy | Behavior |
|---|---|
| `std::launch::async` | Force a **new thread** to run it |
| `std::launch::deferred` | Run **lazily**, on the calling thread, only when `.get()`/`.wait()` is called |
| (default, no policy given) | Implementation picks — may be either, depending on system load |

```cpp
auto f = std::async(std::launch::async, compute);   // force real parallel thread
```

**`async` vs manual `thread` + `promise`:**
| | `std::thread` + `promise` | `std::async` |
|---|---|---|
| Boilerplate | High (manual promise, manual join) | Low (one line) |
| Return value | Manual `set_value()` | Automatic, from function's `return` |
| Exception propagation | Manual (`set_exception`) | Automatic |
| Control over thread creation | Full | Less (implementation may decide) |

---

## 7. std::packaged_task

A middle ground between raw `promise` and `async`: wraps a callable so that calling
it automatically stores its result in an internal promise, giving you a `future` —
but **you** control exactly when/how it actually runs (which thread, which queue).

```cpp
std::packaged_task<int()> task([] { return 42; });
std::future<int> f = task.get_future();

std::thread t(std::move(task));   // you decide: run it on this specific thread
t.join();

std::cout << f.get();   // 42
```

**Why this matters for thread pools:** a pool wants to accept arbitrary callables,
put them in a queue, and hand back a `future` to the caller immediately —
`packaged_task` is exactly the wrapper that makes that possible (see file 06).

---

## 8. Retrieving Exceptions Through a Future

If the task threw an exception instead of returning normally, that exception is
**captured** and **re-thrown** when you call `.get()` — even across threads.

```cpp
std::future<int> f = std::async([] {
    throw std::runtime_error("something broke");
    return 0;
});

try {
    f.get();               // exception re-thrown HERE, on the calling thread
} catch (const std::exception& e) {
    std::cout << "Caught: " << e.what() << "\n";
}
```

This means you never need manual try/catch + error-flag plumbing between threads —
the future carries the exception across for you.

---

## 9. std::shared_future

A plain `std::future` can only be consumed **once**, by **one** thread (`.get()`
moves the value out). `std::shared_future` allows **multiple threads** to all call
`.get()` and read the same result.

```cpp
std::promise<int> p;
std::shared_future<int> sf = p.get_future().share();   // convert to shared_future

std::thread reader1([sf] { std::cout << sf.get(); });
std::thread reader2([sf] { std::cout << sf.get(); });  // both allowed!

p.set_value(100);
reader1.join();
reader2.join();
```

| | `future` | `shared_future` |
|---|---|---|
| Copyable | ❌ (move-only) | ✅ |
| Number of consumers | 1 | Many |
| `.get()` callable more than once | ❌ | ✅ (each call after the first returns the same value) |

---

## ✅ Quick Recap

- Condition variables let a thread sleep until notified, instead of busy-waiting — always used with a mutex + a predicate to guard against spurious wakeups.
- `notify_one()` wakes one waiter, `notify_all()` wakes everyone.
- `promise`/`future` transport a **value** (or exception) from one thread to another; `future::get()` blocks and consumes.
- `std::async` wraps thread creation + promise plumbing into one call; use `launch::async` to force real parallelism.
- `packaged_task` gives you a future while letting *you* control exactly where/when the task runs — the building block for thread pools.
- Exceptions thrown inside a task automatically propagate through `.get()`.
- `shared_future` lets multiple threads read the same result; plain `future` is single-consumer only.

**Next:** `04_atomics_and_memory_model.md` — the low-level rules for how memory operations become visible across threads, and lock-free primitives.
