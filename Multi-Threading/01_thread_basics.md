# 🧵 01 — Thread Basics in C++

## 📑 Table of Contents
- [1. What is a Thread?](#1-what-is-a-thread)
- [2. User Threads vs Kernel Threads vs CPU Cores](#2-user-threads-vs-kernel-threads-vs-cpu-cores)
- [3. 5 Ways to Create Threads in C++11](#3-5-ways-to-create-threads-in-c11)
- [4. Life Cycle of a Thread](#4-life-cycle-of-a-thread)
- [5. join() vs detach() vs joinable()](#5-join-vs-detach-vs-joinable)
- [6. Handling join() in Exception Scenarios](#6-handling-join-in-exception-scenarios)
- [7. Passing Parameters to a Thread](#7-passing-parameters-to-a-thread)
- [8. Problems When Passing Parameters](#8-problematic-situations-when-passing-parameters)
- [9. Transferring Ownership of a Thread](#9-transferring-ownership-of-a-thread)
- [10. Useful Thread Operations](#10-useful-thread-operations)
- [11. Thread Local Storage](#11-thread-local-storage-tls)

---

## 1. What is a Thread?

💡 **In plain English:** think of your program (the **process**) as a small
restaurant kitchen. A **thread** is one chef working in that kitchen. All the
chefs share the same fridge and pantry (**memory**), but each chef has their own
personal cutting board and workspace (their **own stack**) that nobody else
touches. One chef alone can only do one thing at a time — hire more chefs
(threads) and multiple dishes get made at once.

More precisely: a **process** is a running program with its own memory space. A
**thread** is a lightweight "worker" *inside* that process. All threads in the
same process **share the same memory** (heap, global variables) but each thread
gets its **own stack**.

```
Process (your .exe)
 ├── Heap / Globals   ← shared by ALL threads (the shared fridge)
 ├── Thread 1  → own stack, own registers, own Program Counter (chef 1's own bench)
 ├── Thread 2  → own stack, own registers, own Program Counter (chef 2's own bench)
 └── Thread 3  → own stack, own registers, own Program Counter (chef 3's own bench)
```

**Why use threads?**
- Do multiple things at once (download a file *while* updating the UI).
- Use multiple CPU cores for real parallel speed-up (image processing, matrix math).

**The catch:** because threads share memory (the same fridge), two threads
grabbing and changing the same item at the same time can step on each other's
work — this is called a **race condition** (covered in file 09). This is the #1
reason multithreading is considered "hard": one chef alone is simple, but two
chefs both grabbing the same bowl of eggs at once can get messy.

---

## 2. User Threads vs Kernel Threads vs CPU Cores

💡 **In plain English:** you might imagine that the moment you write
`std::thread t(...)`, your code is instantly running on its own CPU core. It's
not quite that direct — there's a small chain of "who is actually in charge of
running this." This section explains that chain, so later ideas like
"oversubscription" (too many threads) make sense.

Before creating threads, it helps to know **who actually schedules them** — i.e.
who decides which piece of code runs on the physical CPU at any given moment.
There are two "levels" of thread, and a mapping model connects them:

- **User thread** — an abstraction managed by a **user-space library/runtime**
  (e.g. a green-thread scheduler, a coroutine runtime) — basically, a "thread"
  that your *own program* keeps track of, not the operating system. The OS
  ("kernel," the core part of the operating system that manages hardware and
  scheduling) doesn't necessarily know it exists.
- **Kernel thread** — a real entity the **OS scheduler** (the part of the
  operating system that decides what runs when) knows about, has its own kernel
  stack + scheduling info (on Linux/Android this is called a `task_struct`), and
  is the thing that actually gets placed onto a **CPU core** to run.

```
 User Threads  →  Kernel Threads  →  CPU Cores
 (your code)      (OS scheduler)     (hardware)
```

There are 3 classic models for how user threads map to kernel threads:

### 🔹 N:1 — Many-to-One ("green threads")
All user threads are multiplexed onto a **single** kernel thread. Switching between
user threads happens entirely in user space (fast, no syscall).

```
User threads:   U1   U2   U3   U4
                 \    \   /    /
                  \    \ /    /
                [ user-space scheduler ]
                          │
                  [ Kernel Thread K1 ]
                          │
                    [ CPU Core 0 ]
```

| ✅ Pros | ❌ Cons |
|---|---|
| Very cheap thread switch (no syscall) | Only ONE core is ever used → no real parallelism |
| Can have thousands of "threads" cheaply | If one user thread makes a blocking syscall, **all** of them freeze |

### 🔹 1:1 — One-to-One (what `std::thread` / pthreads use on Linux & Android)
Every user thread gets its **own dedicated** kernel thread. This is the model used
by `pthread_create` (and therefore `std::thread`, since libstdc++/bionic implement
it on top of pthreads).

```
User thread U1  ──── Kernel Thread K1 ──── CPU Core 0
User thread U2  ──── Kernel Thread K2 ──── CPU Core 1
User thread U3  ──── Kernel Thread K3 ──── CPU Core 2
```

| ✅ Pros | ❌ Cons |
|---|---|
| True parallelism — each thread can run on a different core at the same time | Creating a thread = a syscall (`clone()` on Linux) → heavier than a user-space switch |
| One thread blocking (e.g. on I/O) never blocks the others | Kernel scheduling overhead grows with thread count |
| Simple mental model — what you create is what gets scheduled | Too many threads vs too few cores → **oversubscription** (see file 06) |

### 🔹 M:N — Many-to-Many (hybrid, used by Go's goroutines, old Solaris threads)
A pool of **M** user threads is scheduled across a smaller pool of **N** kernel
threads, by a smart user-space scheduler that can move a user thread to a different
kernel thread if one blocks.

```
User threads: U1 U2 U3 U4 U5 U6
                \  \  |  |  /  /
             [ M:N user-space scheduler ]
                /        |        \
        Kernel K1     Kernel K2     Kernel K3
            │             │             │
        CPU Core 0    CPU Core 1    CPU Core 2
```

| ✅ Pros | ❌ Cons |
|---|---|
| Cheap user-space switching **and** real multi-core parallelism | Scheduler itself is complex to build correctly |
| A blocking user thread can be migrated off its kernel thread so others aren't stuck | Harder to debug (extra scheduling layer) |

### Quick comparison

| Model | Kernel threads per user thread | Uses multiple cores? | Blocking call affects others? | Real-world example |
|---|---|---|---|---|
| N:1 | many : 1 | ❌ No | ✅ Yes, everyone blocks | Old green-thread runtimes |
| 1:1 | 1 : 1 | ✅ Yes | ❌ No | `std::thread` / pthreads (Linux, Android) |
| M:N | many : fewer | ✅ Yes | ❌ No (runtime migrates) | Go goroutines |

### And then: Kernel Thread → Core

Once a kernel thread exists, the **OS scheduler** decides which core it runs on —
and it can **migrate** a kernel thread between cores over its lifetime.

```
             OS Scheduler
         ┌─────┬─────┬─────┐
         ▼     ▼     ▼     ▼
       Core0 Core1 Core2 Core3
         │     │     │     │
        K1    K2    K3   (idle)
```

If there are **more kernel threads than cores**, the scheduler time-slices them on
the same core (context switching) — this is exactly why
`std::thread::hardware_concurrency()` (section 10 below) matters: spawning far more
`std::thread`s than you have cores just adds context-switch overhead without adding
real parallelism.

> 🔧 **AOSP/Linux note:** since `std::thread` follows the 1:1 model on Linux/Android,
> every `std::thread` you spawn shows up as its own schedulable entity in `ps -T` /
> `/proc/<pid>/task/` — same as any pthread. There's no hidden user-space multiplexing
> happening underneath.

---

## 3. 5 Ways to Create Threads in C++11

💡 **In plain English:** creating a thread is like hiring a chef and handing them
a recipe card — the "recipe card" is just *something you can call/run*, in
whatever form is convenient. C++ calls anything runnable like this a **callable**.
A "callable" is simply: a function, or anything that *behaves* like a function
when you put `()` after it. `std::thread` accepts any of these 5 flavors:

### ① Function Pointer
```cpp
void task() { std::cout << "Hello from thread\n"; }

std::thread t1(task);
t1.join();
```

### ② Lambda Function
```cpp
std::thread t2([] {
    std::cout << "Hello from lambda\n";
});
t2.join();
```

### ③ Functor (a class with `operator()`)
```cpp
class Task {
public:
    void operator()() { std::cout << "Hello from functor\n"; }
};

std::thread t3(Task());   // note: Task() not Task — see "most vexing parse" below
t3.join();
```

⚠️ **Most Vexing Parse trap:** `std::thread t3(Task());` can be misread by the
compiler as a *function declaration* named `t3` returning `std::thread`. Fix with
extra parens or braces:
```cpp
std::thread t3((Task()));   // extra parens
std::thread t4{Task()};     // brace-init, safest
```

### ④ Member Function of a Class
```cpp
class Printer {
public:
    void print(int id) { std::cout << "Printer " << id << "\n"; }
};

Printer p;
std::thread t5(&Printer::print, &p, 42);   // &Class::method, object, args...
t5.join();
```

### ⑤ Static Member Function
```cpp
class Logger {
public:
    static void log(std::string msg) { std::cout << msg << "\n"; }
};

std::thread t6(&Logger::log, "system started");
t6.join();
```

| Method | Syntax pattern | When to use |
|---|---|---|
| Function pointer | `thread(func)` | Simple, standalone tasks |
| Lambda | `thread([]{...})` | Short, one-off logic (most common today) |
| Functor | `thread(Obj())` | Reusable callable object with state |
| Member function | `thread(&Class::fn, &obj, args)` | Task needs an object's data |
| Static member | `thread(&Class::staticFn, args)` | Utility function, no object needed |

---

## 4. Life Cycle of a Thread

```
   [Created]
       │  std::thread t(func);
       ▼
   [Running] ──────► executes func()
       │
       ├── join()   → main thread WAITS here until t finishes → [Joined/Terminated]
       │
       └── detach() → t runs independently in background → [Detached]
                        (main thread does NOT wait)
```

- **Created** — the moment `std::thread t(...)` runs, the OS thread starts immediately
  (there's no "start()" call in C++ — construction *is* starting).
- **Running** — executing the callable you gave it.
- **Joined** — someone called `join()`, and it blocked until the thread's work finished.
- **Detached** — someone called `detach()`; the thread lives on its own. Once detached,
  you lose the ability to `join()` it later.
- **Destructed** — if a `std::thread` object is destroyed while still joinable
  (neither joined nor detached), the program calls `std::terminate()` and crashes.
  This is the single most common beginner bug.

---

## 5. join() vs detach() vs joinable()

| | `join()` | `detach()` |
|---|---|---|
| Blocks caller? | ✅ Yes, waits till thread finishes | ❌ No, returns immediately |
| Thread after call | Cleaned up automatically | Runs independently in background |
| Can you join later? | N/A (already joined) | ❌ Never |
| Typical use case | Need the result before continuing | Fire-and-forget (e.g. logging) |

`joinable()` answers: **"Is this thread object still attached to a live OS thread that
hasn't been joined/detached yet?"**

```cpp
std::thread t(task);

if (t.joinable()) {
    t.join();   // safe: only join if it makes sense to
}
```

A default-constructed `std::thread` (no function given) is **not** joinable.
After `join()` or `detach()`, the thread object becomes **not** joinable again.

---

## 6. Handling join() in Exception Scenarios

**Problem:** if an exception is thrown *between* creating a thread and calling
`join()`, the `join()` line is skipped → the thread object is destroyed while still
joinable → **crash** (`std::terminate`).

```cpp
void risky() {
    std::thread t(task);

    doSomethingThatMightThrow();  // 💥 if this throws...

    t.join();   // ...this never runs → crash
}
```

### ✅ Fix 1: try/catch
```cpp
void risky() {
    std::thread t(task);
    try {
        doSomethingThatMightThrow();
    } catch (...) {
        t.join();   // clean up before rethrowing
        throw;
    }
    t.join();
}
```

### ✅ Fix 2 (preferred): RAII wrapper — "thread guard"
Wrap the thread in a small class whose destructor calls `join()`. Since destructors
run automatically during stack unwinding (even on exceptions), you get safety for free.

```cpp
class ThreadGuard {
    std::thread& t;
public:
    explicit ThreadGuard(std::thread& t_) : t(t_) {}
    ~ThreadGuard() {
        if (t.joinable()) t.join();
    }
    ThreadGuard(const ThreadGuard&) = delete;
    ThreadGuard& operator=(const ThreadGuard&) = delete;
};

void risky() {
    std::thread t(task);
    ThreadGuard g(t);        // guaranteed join, even on exception
    doSomethingThatMightThrow();
}   // g destructs here → t.join() called automatically
```

> This is exactly the same pattern as `std::lock_guard` for mutexes — "acquire in
> constructor, release in destructor." In C++20, `std::jthread` builds this in natively
> (see file 08).

---

## 7. Passing Parameters to a Thread

Parameters are passed as **extra arguments** to the `std::thread` constructor —
they get copied/moved into internal storage, then forwarded to your function.

```cpp
void greet(std::string name, int times) {
    for (int i = 0; i < times; ++i)
        std::cout << "Hi " << name << "\n";
}

std::thread t(greet, "Sky", 3);   // args copied into the thread
t.join();
```

To pass by **reference**, wrap with `std::ref()` — otherwise C++ copies by default
(even if the target function signature takes a reference!):

```cpp
void increment(int& x) { x++; }

int value = 10;
std::thread t(increment, std::ref(value));   // must use std::ref
t.join();
std::cout << value;   // 11
```

---

## 8. Problematic Situations When Passing Parameters

### ❌ Problem 1 — Dangling reference to a local variable
```cpp
void risky_launcher() {
    int local = 5;
    std::thread t(increment, std::ref(local));
    t.detach();   // 💥 function returns, `local` is destroyed,
}                 //    but the detached thread might still be using it!
```
**Fix:** never `detach()` a thread that references a local variable. Either `join()`
before the local goes out of scope, or pass by value/heap-allocate.

### ❌ Problem 2 — Implicit conversions happen on the CALLER's side too late
```cpp
void print(std::string s) { std::cout << s; }

char buffer[50];
sprintf(buffer, "data");
std::thread t(print, buffer);   // buffer (char*) is copied as char*,
t.detach();                     // then converted to std::string LATER,
                                 // inside the new thread — after `buffer`
                                 // may already be destroyed/reused!
```
**Fix:** convert explicitly *before* passing:
```cpp
std::thread t(print, std::string(buffer));
```

### ❌ Problem 3 — Forgetting a member function needs `this`
```cpp
class Worker {
public:
    void run() { /* ... */ }
};
Worker w;
std::thread t(&Worker::run, w);   // note: passes COPY of w
// Use &w if you want the SAME object modified inside the thread
```

| Trap | Symptom | Fix |
|---|---|---|
| Passing local by reference + detach | Crash / undefined behavior | join before scope ends, or pass by value |
| Raw pointer/char* to temp buffer | Garbage data read in thread | Convert to `std::string` before passing |
| Forgetting `std::ref` | Function modifies a copy, not original | Wrap with `std::ref(x)` |

---

## 9. Transferring Ownership of a Thread

`std::thread` is **move-only** — it cannot be copied (only one owner may control a
given OS thread at a time). You move it instead.

```cpp
std::thread t1(task);
std::thread t2 = std::move(t1);   // ownership moves to t2
// t1 is now empty (not joinable), t2 owns the running thread

t2.join();
```

**Why this matters:** you can return a thread from a function, store threads in a
`std::vector<std::thread>`, or swap ownership between wrapper classes — all via `move`.

```cpp
std::vector<std::thread> pool;
for (int i = 0; i < 4; ++i)
    pool.emplace_back(task);      // move-constructed directly into vector

for (auto& t : pool) t.join();
```

---

## 10. Useful Thread Operations

| Call | What it does |
|---|---|
| `t.get_id()` | Unique ID of thread `t` |
| `std::this_thread::get_id()` | ID of the *currently running* thread |
| `std::this_thread::sleep_for(ms)` | Pause current thread for a duration |
| `std::thread::hardware_concurrency()` | Hint: number of cores available (0 if unknown) |
| `std::this_thread::yield()` | "I'm done for now, let another thread run" (scheduling hint) |
| `t.swap(other)` | Swap which OS thread two `std::thread` objects manage |

```cpp
unsigned cores = std::thread::hardware_concurrency();
std::cout << "This machine reports " << cores << " cores\n";
```

This is commonly used to decide **how many worker threads to spawn** in a thread pool
(file 06) — spawning way more threads than cores causes *oversubscription* (context-switch
overhead outweighs the benefit, as shown in section 2's diagram above).

---

## 11. Thread Local Storage (TLS)

💡 **In plain English:** back to the kitchen analogy — normally the "shared
fridge" (a global variable) is visible to every chef, and if one chef changes
what's in it, every other chef sees that change too. `thread_local` is like
giving each chef their **own personal mini-fridge** with the same label on it —
they can each keep their own separate "counter" or "cache" without ever seeing
or interfering with anyone else's.

Normally, a variable declared at global/namespace scope is **shared** across all
threads. Marking it `thread_local` gives **each thread its own separate copy**.

```cpp
thread_local int counter = 0;

void task() {
    counter++;
    std::cout << "Thread " << std::this_thread::get_id()
              << " counter = " << counter << "\n";
}

std::thread t1(task);
std::thread t2(task);
t1.join();
t2.join();
// Each thread prints counter = 1 — they never share the variable!
```

```
              counter (thread_local)
Thread 1 ───► [ copy A = 1 ]
Thread 2 ───► [ copy B = 1 ]
```

**When to use:** per-thread caches, per-thread random number generators, per-thread
error codes (this is literally how `errno` is implemented under the hood) — anywhere
you want "global-like convenience" without the race conditions of true sharing.

---

## ✅ Quick Recap

- A thread = one execution path inside a process; all threads share heap/globals, each has its own stack.
- User threads map to kernel threads via **N:1** (green threads, no parallelism), **1:1** (`std::thread`/pthreads — true parallelism), or **M:N** (hybrid, e.g. Go).
- Kernel threads are then scheduled onto CPU cores by the OS — more kernel threads than cores means time-slicing/context switching.
- 5 ways to make a callable: function pointer, lambda, functor, member fn, static member fn.
- Every `std::thread` must be `join()`ed or `detach()`ed before it's destroyed — or the program crashes.
- Use a RAII "thread guard" (or C++20 `jthread`) so exceptions can't skip your `join()`.
- Arguments are **copied** into the thread by default — use `std::ref()` for real references, and beware raw pointers/char* to short-lived buffers.
- `std::thread` is move-only — this is how you store threads in containers or hand off ownership.
- `thread_local` gives each thread a private copy of a variable — no synchronization needed.

**Next:** `02_mutex_and_locking.md` — how to safely let threads touch *shared* data.
