# ⚛️ 04 — Atomics & the C++ Memory Model

## 📑 Table of Contents
- [1. Why Atomics Exist](#1-why-atomics-exist)
- [2. std::atomic_flag](#2-stdatomic_flag)
- [3. std::atomic\<bool\> and Other Atomic Types](#3-stdatomicbool-and-other-atomic-types)
- [4. compare_exchange (CAS)](#4-compare_exchange-cas)
- [5. Atomic Pointers](#5-atomic-pointers)
- [6. Is a Static Variable Thread-Safe?](#6-is-a-static-variable-thread-safe)
- [7. Memory Ordering — Why It Exists](#7-memory-ordering--why-it-exists)
- [8. memory_order_seq_cst](#8-memory_order_seq_cst)
- [9. memory_order_relaxed](#9-memory_order_relaxed)
- [10. memory_order_acquire / memory_order_release](#10-memory_order_acquire--memory_order_release)
- [11. Transitive Synchronization & Release Sequence](#11-transitive-synchronization--release-sequence)
- [12. memory_order_consume](#12-memory_order_consume)
- [13. Implementing a Spinlock](#13-implementing-a-spinlock)

---

## 1. Why Atomics Exist

💡 **In plain English:** "atomic" here just means **"cannot be split into smaller
steps that another thread could sneak in between."** Imagine two people counting
the same jar of candy at the same time by both reading the current number, adding
1 in their heads, then writing the new number down — if they read the number at
the same moment, both might read "5," both write back "6," and one candy's worth
of counting is silently lost. An **atomic** operation makes sure that "read, add,
write" happens as one uninterruptible block, so this can never happen.

`counter++` on a plain `int` is actually **3 separate steps**: read, add 1, write
back. If two threads do this at once, one increment can be lost:

```
Thread A: read counter (=5) ─────────────► write 6
Thread B:      read counter (=5) ──► write 6
Result: counter = 6, but it should be 7!  (one increment was LOST)
```

An **atomic** operation is indivisible — no other thread can ever observe it
"half done." `std::atomic<int>` makes `++` a single, safe, hardware-backed
operation with no lock needed.

```cpp
std::atomic<int> counter{0};
counter++;   // guaranteed atomic — no lost updates, no mutex needed
```

**Atomics vs mutex:** a mutex protects an arbitrary *block* of code; an atomic
protects a *single variable* for a *single operation*, usually implemented with
special CPU instructions instead of OS-level locking — much cheaper for simple
counters/flags.

---

## 2. std::atomic_flag

The **only** atomic type guaranteed to be lock-free on every platform. It's a
minimal boolean flag with just two operations: `test_and_set()` and `clear()`.

```cpp
std::atomic_flag flag = ATOMIC_FLAG_INIT;

flag.test_and_set();   // sets to true, returns the PREVIOUS value
flag.clear();           // resets to false
```

This is the minimal building block used to hand-roll a spinlock (section 13).

---

## 3. std::atomic\<bool\> and Other Atomic Types

`std::atomic<T>` is a template — works for `bool`, `int`, `long`, pointers, and
(with restrictions) user-defined trivially-copyable types.

```cpp
std::atomic<bool> ready{false};
std::atomic<int>  count{0};

ready.store(true);          // atomic write
bool r = ready.load();      // atomic read
count.fetch_add(1);         // atomic add, returns OLD value
count++;                    // same thing, operator sugar
```

| Operation | Meaning |
|---|---|
| `load()` | Atomic read |
| `store(v)` | Atomic write |
| `fetch_add(n)` / `fetch_sub(n)` | Atomic add/subtract, returns old value |
| `exchange(v)` | Atomically set to `v`, return the old value |
| `compare_exchange_weak/strong` | "CAS" — see next section |

---

## 4. compare_exchange (CAS)

💡 **In plain English:** imagine you're editing a shared Google Doc offline, and
when you reconnect you say "if the document still says exactly what it said when
I started editing, replace it with my new version — otherwise, just show me
what it actually says now, and I'll redo my edit based on that." That's exactly
what compare-and-swap does for a single variable: safely update it *only if*
nobody else changed it since you last looked, and if they did, hand you the
up-to-date value so you can try again.

**Compare-And-Swap (CAS)**: "if the current value equals what I *expect*, replace
it with a *new* value — atomically. Otherwise, tell me what the current value
actually is."

```cpp
std::atomic<int> value{10};

int expected = 10;
int desired  = 20;

bool success = value.compare_exchange_strong(expected, desired);
// if value WAS 10: it becomes 20, success = true
// if value was NOT 10: expected is updated to the ACTUAL current value, success = false
```

```
      compare_exchange_strong(expected, desired)
             │
    value == expected? ──yes──► value = desired, return true
             │
             no
             │
             ▼
     expected = <actual current value>, return false
```

This is the core primitive behind **all lock-free data structures** (file 05) — you
read a value, compute a new version, then CAS it in; if someone else changed it
first, you just retry with the fresh value.

**`_weak` vs `_strong`:** `compare_exchange_weak` may fail spuriously (return false
even though values matched, due to hardware quirks on some platforms) — cheaper,
meant to be used in a retry loop anyway. `compare_exchange_strong` never fails
spuriously but can be marginally more expensive.

```cpp
// typical retry loop
int old_val = value.load();
int new_val;
do {
    new_val = old_val + 1;
} while (!value.compare_exchange_weak(old_val, new_val));
```

---

## 5. Atomic Pointers

`std::atomic<T*>` gives atomic load/store/CAS on a raw pointer — this is the
backbone of lock-free linked structures (a lock-free stack's `head` pointer, for
example — see file 05).

```cpp
std::atomic<Node*> head{nullptr};

Node* new_node = new Node(value);
new_node->next = head.load();
while (!head.compare_exchange_weak(new_node->next, new_node)) {
    // if head changed since we read it, new_node->next is refreshed
    // with the current head automatically — retry the CAS
}
```

---

## 6. Is a Static Variable Thread-Safe?

**It depends on what kind of "static":**

| Case | Thread-safe? | Why |
|---|---|---|
| Static **local** variable initialization (C++11 onward) | ✅ Yes | The standard guarantees the *first* initialization is synchronized — other threads block until it's done |
| Reading/writing a static variable's **value** afterwards | ❌ No | Ordinary shared-data rules apply — need a mutex/atomic if multiple threads mutate it |
| `static const` / `constexpr` (never mutated) | ✅ Yes | No writes ever happen, so there's nothing to race on |

```cpp
void f() {
    static Logger instance;   // ✅ safe: C++11 guarantees only ONE thread
    instance.log("hi");       //    actually constructs it, others wait
}
```
But:
```cpp
static int counter = 0;
void g() {
    counter++;   // ❌ NOT thread-safe — this is a plain shared int, races just like any other
}
```

---

## 7. Memory Ordering — Why It Exists

💡 **In plain English:** imagine you're baking a cake and texting a friend "cake's
in the oven, come over!" — but for efficiency, you actually send the text a
*little before* you've actually put the cake in (since, from your own point of
view, it doesn't matter which order you do these two things in — you'll have
done both by the time anyone checks). Your friend shows up expecting a cake in
the oven, but it's not there yet! This is exactly the kind of surprise that
"memory ordering" rules exist to prevent between threads.

More technically: atomicity alone isn't the whole story. Compilers and CPUs are
allowed to **reorder** instructions for performance, as long as it doesn't
change the result *for a single thread*. But another thread watching from the
outside *can* observe operations appearing "out of order."

```
Thread A:  data = 42;        // (1)
           ready = true;     // (2)   ← compiler/CPU might reorder (1) and (2)!

Thread B:  if (ready) {      // sees ready == true
               use(data);    // but might see the OLD value of data, if (1) got reordered after (2)!
           }
```

**Memory ordering** is how you tell the compiler/CPU "these operations must stay
in this relative order, as seen by other threads." `std::atomic` operations take
an optional memory-order argument to control exactly how strict this guarantee is.

---

## 8. memory_order_seq_cst

**Sequentially Consistent** — the **default**, and the simplest to reason about:
all threads see all atomic operations happen in **one single global order**,
matching what you'd expect from the program's source order. Safest, but has the
most synchronization overhead (may involve full memory fences on some CPUs).

```cpp
data.store(42, std::memory_order_seq_cst);   // default if you omit the argument
```

**Use this unless you have a specific, measured reason not to.**

---

## 9. memory_order_relaxed

**No ordering guarantee at all** — only atomicity (no lost updates/torn reads),
but the compiler/CPU is free to reorder relaxed operations relative to everything
else. Fastest, but easy to misuse.

```cpp
counter.fetch_add(1, std::memory_order_relaxed);
```

**Safe use case:** independent counters/statistics where you don't care about the
order relative to other memory operations — e.g. a hit-counter nobody else's logic
depends on.

**Unsafe use case:** using a relaxed flag to signal "data is ready" — since there's
no ordering guarantee, another thread might see the flag as `true` while still
seeing stale/uninitialized `data`.

---

## 10. memory_order_acquire / memory_order_release

The classic **producer-consumer synchronization pair** without a mutex:

- **Release** (used on a *store*, by the producer): "everything I wrote *before*
  this store must be visible to any thread that later *acquires* the same
  variable."
- **Acquire** (used on a *load*, by the consumer): "everything the releasing
  thread wrote before its release-store becomes visible to me, starting now."

```cpp
std::atomic<bool> ready{false};
int data = 0;

// Producer thread
data = 42;                                        // (1) plain write
ready.store(true, std::memory_order_release);      // (2) release

// Consumer thread
if (ready.load(std::memory_order_acquire)) {       // (3) acquire
    std::cout << data;   // (4) GUARANTEED to see data == 42, because (3) acquire
}                         //     synchronizes-with (2) release
```

```
Producer:  write data=42 ──► RELEASE store(ready=true)
                                      │
                          "everything before this is now visible..."
                                      │
Consumer:                     ACQUIRE load(ready) == true
                                      │
                          "...to everything after this point"
                                      ▼
                              read data → guaranteed 42
```

This is exactly *why* `mutex::unlock()` behaves like a release and
`mutex::lock()` behaves like an acquire internally — acquire/release is the
low-level primitive that higher-level locks are built on.

---

## 11. Transitive Synchronization & Release Sequence

**Transitive synchronization:** if Thread A releases → Thread B acquires that same
release → Thread B then releases something else → Thread C acquires *that* — then
Thread C is guaranteed to see everything Thread A wrote too. The "happens-before"
relationship chains across multiple hops.

```
A: write X; release(flag1)
        │ (B acquires flag1, sees X)
B: read X; write Y; release(flag2)
        │ (C acquires flag2, sees X AND Y)
C: read X, read Y   ← both guaranteed visible, even though C never touched flag1
```

**Release sequence:** if multiple threads perform a series of atomic
read-modify-write operations (like repeated `fetch_add`) on the same atomic
variable after an initial release-store, an acquiring thread that reads *any* value
in that chain still gets the full synchronize-with guarantee back to the original
release — the "chain" of RMW operations doesn't break the synchronization.

---

## 12. memory_order_consume

An even weaker (rarely-used) ordering, intended for **dependency-carrying** loads
— "only the specific memory that the loaded value depends on (e.g. a struct behind
a loaded pointer) needs to be visible, not everything from before the release."

In practice: `memory_order_consume`'s specification has known issues, no
mainstream compiler gives it distinct optimized codegen from `acquire`, and it's
generally **not recommended** for new code — prefer `acquire`/`release` unless
you have a very specific, well-understood use case.

---

## 13. Implementing a Spinlock

A **spinlock** busy-waits (spins) instead of sleeping — cheap to acquire when
contention is very short-lived, since it avoids the cost of an OS context switch.
Built directly on `atomic_flag`:

```cpp
class SpinLock {
    std::atomic_flag flag = ATOMIC_FLAG_INIT;
public:
    void lock() {
        while (flag.test_and_set(std::memory_order_acquire)) {
            // someone else has it — keep spinning ("busy wait")
        }
    }
    void unlock() {
        flag.clear(std::memory_order_release);
    }
};
```

```
Thread A: test_and_set() → false (wasn't set) → lock acquired, proceeds
Thread B: test_and_set() → true (already set) → loops, spinning, checking again
Thread A: ...critical section... clear() → releases
Thread B: test_and_set() → false now → acquires, proceeds
```

| ✅ Good for | ❌ Bad for |
|---|---|
| Very short critical sections, low contention | Long critical sections (wastes CPU spinning) |
| Situations where context-switch cost > spin cost | High core-count contention (many spinners burning cycles) |

---

## ✅ Quick Recap

- Atomics make an operation indivisible without needing an OS-level mutex — cheaper for simple counters/flags/pointers.
- `compare_exchange` (CAS) is the fundamental "try to update, retry if someone beat you to it" primitive behind all lock-free code.
- A `static` local variable's *first construction* is thread-safe by the standard; its *value* afterward is not, unless it's `const`/never mutated.
- Memory ordering controls what reordering is legal across threads: `seq_cst` (safest, default), `relaxed` (fastest, no ordering), `acquire`/`release` (the producer-consumer synchronization pair), `consume` (rarely used, avoid).
- Acquire/release is literally what mutexes are built on under the hood.
- A spinlock trades OS-sleep cost for CPU-spin cost — good only for very short critical sections.

**Next:** `05_lock_free_data_structures.md` — building real lock-free structures (stacks) and the tricky problem of safely freeing memory when other threads might still be reading it.
