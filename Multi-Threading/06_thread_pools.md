# 🏊 06 — Thread Pools

## 📑 Table of Contents
- [1. Why Thread Pools?](#1-why-thread-pools)
- [2. Oversubscription](#2-oversubscription)
- [3. A Simple Thread Pool](#3-a-simple-thread-pool)
- [4. Thread Pool That Lets You Wait on Tasks](#4-thread-pool-that-lets-you-wait-on-tasks)
- [5. Minimizing Contention on the Work Queue](#5-minimizing-contention-on-the-work-queue)
- [6. Work Stealing](#6-work-stealing)
- [7. Object Pool (Related Pattern)](#7-object-pool-related-pattern)

---

## 1. Why Thread Pools?

💡 **In plain English:** hiring and training a brand-new chef for every single
dish that comes in, then firing them the moment the dish is done, would be
incredibly wasteful — even if the dish itself only takes a minute to cook. It's
much smarter to hire a fixed team of chefs once, and just keep handing them new
orders as they come in. A **thread pool** applies that same idea to threads.

Creating a `std::thread` isn't free — it's a real syscall (a request made
directly to the operating system, specifically `clone()` on Linux, per file 01's
1:1 model), with kernel bookkeeping overhead. If your program creates and
destroys thousands of short-lived threads for thousands of small tasks, that
overhead can dwarf the actual work being done.

**A thread pool** creates a **fixed set of worker threads once**, up front, and
reuses them for many tasks over time — tasks are pushed into a shared queue,
workers pull from it in a loop.

```
                 ┌──────────────┐
Task 1 ──┐       │              │
Task 2 ──┼──►    │  Task Queue  │  ◄── Worker 1 ──┐
Task 3 ──┤       │              │  ◄── Worker 2 ──┼── fixed pool, created ONCE
Task 4 ──┘       │              │  ◄── Worker 3 ──┘
                 └──────────────┘
```

---

## 2. Oversubscription

**Oversubscription** = running more active threads than there are CPU cores to
run them on. The OS has to constantly context-switch between them, which costs
real time and cache locality — beyond a certain point, adding more threads makes
things **slower**, not faster.

```
4 CPU cores, 4 threads:  each thread gets a dedicated core → maximum throughput
4 CPU cores, 40 threads: OS constantly swaps 40 threads across 4 cores →
                          massive context-switch overhead, cache thrashing
```

This is exactly why thread pools are usually sized around
`std::thread::hardware_concurrency()` (file 01, section 10) rather than "one
thread per task."

---

## 3. A Simple Thread Pool

Fixed number of worker threads, all pulling `std::function<void()>` tasks off a
single thread-safe queue (built in file 03).

```cpp
class ThreadPool {
    std::vector<std::thread> workers;
    ThreadSafeQueue<std::function<void()>> tasks;
    std::atomic<bool> stop{false};

    void worker_loop() {
        while (!stop) {
            std::function<void()> task;
            if (tasks.try_pop(task)) {
                task();                       // run it
            } else {
                std::this_thread::yield();    // nothing to do, let others run
            }
        }
    }

public:
    explicit ThreadPool(unsigned n = std::thread::hardware_concurrency()) {
        for (unsigned i = 0; i < n; ++i)
            workers.emplace_back(&ThreadPool::worker_loop, this);
    }

    void submit(std::function<void()> f) {
        tasks.push(std::move(f));
    }

    ~ThreadPool() {
        stop = true;
        for (auto& w : workers) w.join();
    }
};
```

**Limitation:** `submit()` is fire-and-forget — you never get a return value or
know when a specific task finished. Fixed in the next section.

---

## 4. Thread Pool That Lets You Wait on Tasks

Combine the pool with `std::packaged_task` (file 03) — wrapping each submitted
task gives you a `future` back, so callers can wait for / retrieve results.

```cpp
class WaitableThreadPool {
    ThreadSafeQueue<std::function<void()>> tasks;
    std::vector<std::thread> workers;
    std::atomic<bool> stop{false};

public:
    template<typename F, typename... Args>
    auto submit(F&& f, Args&&... args) {
        using ReturnType = std::invoke_result_t<F, Args...>;

        auto task = std::make_shared<std::packaged_task<ReturnType()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...));

        std::future<ReturnType> result = task->get_future();
        tasks.push([task] { (*task)(); });   // erase the type into std::function
        return result;                       // caller can .get() this later
    }
    // ... constructor / worker loop / destructor same shape as section 3 ...
};
```

```cpp
WaitableThreadPool pool(4);
std::future<int> f = pool.submit([] { return 21 * 2; });
std::cout << f.get();   // 42 — waits for the specific task to complete
```

This is the shape used by real-world pools (e.g. `std::async`-like APIs, or
Android's `AsyncTask`/executor patterns conceptually) — submit returns a handle,
not a blocking call.

---

## 5. Minimizing Contention on the Work Queue

With **one shared queue**, every worker thread fights over the **same lock** to
push/pop — under heavy load, that lock itself becomes the bottleneck (defeats the
purpose of having multiple workers!).

```
❌ One shared queue:            ✅ Per-thread queues:

  [ single queue ]               W1's queue   W2's queue   W3's queue
   ▲   ▲   ▲                        ▲             ▲             ▲
   │   │   │  ← everyone            │             │             │
  W1  W2  W3    fights for          W1            W2            W3
      one lock                (each worker mostly touches ONLY its own queue)
```

**Fix:** give each worker its **own local queue**. A worker pushes new tasks
(e.g. spawned by itself) onto its own queue and pops from its own queue first —
no lock contention with other workers most of the time.

```cpp
class LocalQueuePool {
    std::vector<std::unique_ptr<ThreadSafeQueue<std::function<void()>>>> local_queues;
    thread_local static ThreadSafeQueue<std::function<void()>>* my_queue;

    void worker_loop(unsigned index) {
        my_queue = local_queues[index].get();
        while (!stop) {
            std::function<void()> task;
            if (my_queue->try_pop(task)) task();
            else std::this_thread::yield();
        }
    }
    // submit(): pushes to my_queue if called from a worker, else to a fallback global queue
};
```

**New problem this creates:** what if one worker's local queue is empty but
another worker's queue is *overflowing* with tasks? The idle worker just sits
there doing nothing while work piles up elsewhere. That's exactly what work
stealing fixes.

---

## 6. Work Stealing

💡 **In plain English:** picture each chef with their own personal stack of order
tickets. If Chef A runs out of tickets but Chef B still has a huge pile, it would
be silly for Chef A to just stand around — instead, Chef A quietly grabs a ticket
from the *bottom* of Chef B's pile (so they're not both reaching for the same
ticket at the same time) and gets to work on it.

Each worker has its own queue (as above), **but** when a worker's own queue is
empty, it's allowed to "steal" a task from the **back** of another worker's queue
instead of sitting idle.

```
W1's queue: [ T1 T2 T3 T4 T5 ]      W2's queue: [ empty ]
                                │
              W2 is idle → steals from the BACK of W1's queue
                                ▼
W1's queue: [ T1 T2 T3 T4 ]         W2's queue: [ T5 ]  ← now processing it
```

**Why steal from the back, and process own tasks from the front?** Own tasks are
usually processed front-to-back (FIFO, oldest first). Stealing from the *opposite*
end (the back) minimizes lock contention between the owner (working at the front)
and thieves (working at the back) — they rarely fight over the same end of the
queue.

```cpp
void worker_loop(unsigned my_index) {
    while (!stop) {
        std::function<void()> task;
        if (my_queue->try_pop_front(task)) {
            task();
        } else if (try_steal_from_others(my_index, task)) {
            task();     // stole work from a busier neighbor
        } else {
            std::this_thread::yield();
        }
    }
}
```

| Design | Contention | Load balance |
|---|---|---|
| Single shared queue | High (everyone fights for one lock) | Perfect (whoever's free just grabs next task) |
| Per-thread queues, no stealing | Low | Poor (idle workers stay idle if their queue is empty) |
| Per-thread queues + work stealing | Low-moderate | Good (idle workers pull from busy ones) |

This is the model used by production task schedulers (Intel TBB, Java's
`ForkJoinPool`, Rust's Rayon) for exactly this reason: low contention **and** good
load balancing.

---

## 7. Object Pool (Related Pattern)

Not strictly a *threading* primitive, but often paired with thread pools: an
**object pool** pre-allocates a fixed set of reusable, expensive-to-construct
objects (buffers, connections) instead of constructing/destructing them per task.

```
Object Pool: [ Obj1(free) Obj2(in-use) Obj3(free) Obj4(in-use) ]

Worker needs an object → borrows a "free" one → uses it → returns it to the pool (marked free again)
```

In a multithreaded context, the pool's "borrow"/"return" operations themselves
need synchronization (typically a mutex or a lock-free free-list built the same
way as file 05's lock-free stack) — borrowing/returning is essentially "pop/push"
on a concurrent free-list of available objects.

```cpp
template<typename T>
class ObjectPool {
    ThreadSafeQueue<std::unique_ptr<T>> free_objects;
public:
    std::unique_ptr<T> acquire() {
        std::unique_ptr<T> obj;
        if (!free_objects.try_pop(obj)) obj = std::make_unique<T>();  // create if empty
        return obj;
    }
    void release(std::unique_ptr<T> obj) {
        free_objects.push(std::move(obj));
    }
};
```

**Why pair it with a thread pool:** worker threads repeatedly need short-lived
buffers/connections — reusing pre-built objects avoids repeated
allocation/deallocation overhead on every task, same motivation as reusing threads
instead of spawning new ones.

---

## ✅ Quick Recap

- Thread pools amortize the cost of thread creation by reusing a fixed set of workers across many tasks.
- Oversubscription (more active threads than cores) makes things *slower* — size pools around `hardware_concurrency()`.
- Wrapping tasks in `packaged_task` lets `submit()` return a `future`, so callers can retrieve results/exceptions.
- A single shared task queue is simple but becomes a contention bottleneck under load.
- Per-thread local queues reduce contention but can leave some workers idle while others are overloaded.
- Work stealing fixes that: idle workers steal from the back of a busy worker's queue, while the owner works from the front — minimizing contention between the two.
- Object pools apply the same "reuse instead of recreate" idea to expensive objects, and are commonly synchronized the same way as a concurrent stack/queue.

**Next:** `07_parallel_algorithms.md` — using threads to parallelize real algorithms (accumulate, sort, search, matrix ops).
