# 🚀 05 — Lock-Free Data Structures

## 📑 Table of Contents
- [1. Terminology: Lock-Free vs Wait-Free vs Obstruction-Free](#1-terminology-lock-free-vs-wait-free-vs-obstruction-free)
- [2. Stack Recap (the Mutex Version)](#2-stack-recap-the-mutex-version)
- [3. A Simple Lock-Free Stack](#3-a-simple-lock-free-stack)
- [4. The Memory Reclamation Problem](#4-the-memory-reclamation-problem)
- [5. Reclamation via Thread Counting](#5-reclamation-via-thread-counting)
- [6. Reclamation via Reference Counting](#6-reclamation-via-reference-counting)
- [7. Reclamation via Hazard Pointers](#7-reclamation-via-hazard-pointers)

---

## 1. Terminology: Lock-Free vs Wait-Free vs Obstruction-Free

💡 **In plain English:** so far (file 02), we solved "two people can't use the
bathroom at once" by putting a lock on the door — if it's occupied, you wait
outside. A **lock-free** approach removes the door entirely: everyone tries to
walk in at once, and if there's a collision, whoever "loses" just steps back and
tries again immediately (instead of standing in a queue). Nobody ever waits
around doing nothing — they just keep retrying until it works.

| Term | Guarantee |
|---|---|
| **Obstruction-free** | A thread makes progress if it runs *alone* (no contention) — weakest guarantee |
| **Lock-free** | At any point, **at least one** thread in the system is guaranteed to make progress, even under contention (others might retry, but the system as a whole never stalls) |
| **Wait-free** | **Every** thread makes progress in a bounded number of steps, regardless of what other threads do — strongest, hardest to achieve |

"Lock-free" doesn't mean "no synchronization" — it means **no OS-level mutex/lock**
is used; synchronization instead happens via atomic CAS retry loops (file 04).

```
Lock-based:    Thread waits (blocked) if lock is held → OS puts it to sleep
Lock-free:     Thread never blocks — it retries its CAS loop until it succeeds
```

**Why bother?** No thread can ever be put to sleep holding a lock and block
everyone else (no deadlock risk from this structure), and no OS context-switch
cost for contention — at the price of much trickier code, especially around
memory cleanup (section 4 onward).

---

## 2. Stack Recap (the Mutex Version)

The simple, safe version from file 02: one mutex guards the whole stack.

```cpp
template<typename T>
class LockingStack {
    std::stack<T> data;
    std::mutex m;
public:
    void push(T v) {
        std::lock_guard<std::mutex> lock(m);
        data.push(std::move(v));
    }
    std::shared_ptr<T> pop() {
        std::lock_guard<std::mutex> lock(m);
        if (data.empty()) return nullptr;
        auto res = std::make_shared<T>(std::move(data.top()));
        data.pop();
        return res;
    }
};
```
Simple and correct — but every `push`/`pop` serializes through one lock, even
under heavy contention. A lock-free version tries to let multiple threads succeed
concurrently using CAS instead.

---

## 3. A Simple Lock-Free Stack

Implemented as a singly-linked list; `head` is an atomic pointer. Push and pop both
use a CAS retry loop.

```
head ──► [Node C] ──► [Node B] ──► [Node A] ──► nullptr
```

```cpp
template<typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
        Node(T v) : data(std::move(v)), next(nullptr) {}
    };
    std::atomic<Node*> head{nullptr};

public:
    void push(T value) {
        Node* new_node = new Node(std::move(value));
        new_node->next = head.load();
        while (!head.compare_exchange_weak(new_node->next, new_node)) {
            // head changed underneath us — new_node->next auto-refreshed, retry
        }
    }

    std::shared_ptr<T> pop() {
        Node* old_head = head.load();
        while (old_head &&
               !head.compare_exchange_weak(old_head, old_head->next)) {
            // someone else popped/pushed first — old_head refreshed, retry
        }
        if (!old_head) return nullptr;
        auto result = std::make_shared<T>(std::move(old_head->data));
        // ⚠️ we CANNOT just `delete old_head` here — see section 4!
        return result;
    }
};
```

```
push():  new_node.next = head  ──►  CAS(head, new_node.next, new_node)
                                       │succeeds → new_node is now head
                                       │fails (someone else pushed first)
                                       └──► retry with refreshed head

pop():   old_head = head  ──►  CAS(head, old_head, old_head->next)
                                       │succeeds → old_head removed
                                       │fails → retry with refreshed old_head
```

---

## 4. The Memory Reclamation Problem

The tricky part isn't push/pop — it's **when is it safe to `delete old_head`?**

```
Thread A: pop() reads old_head = NodeX, is ABOUT to read old_head->data...
Thread B: pop() also gets NodeX as old_head, succeeds its CAS, deletes NodeX!
Thread A: now reads old_head->data → 💥 USE-AFTER-FREE (dangling pointer)
```

Because multiple threads can be "in the middle of" reading a node when another
thread logically removes it, you **cannot safely free a node the instant it's
popped** — some other thread might still hold a pointer to it. You need a
reclamation strategy. Three common ones:

---

## 5. Reclamation via Thread Counting

Keep a global "how many threads are currently inside `pop()`" counter. Only
actually `delete` nodes once you're confident no thread could still be reading
them.

```cpp
std::atomic<unsigned> threads_in_pop{0};
std::atomic<Node*> to_be_deleted{nullptr};

std::shared_ptr<T> pop() {
    ++threads_in_pop;                       // "I'm in here, don't free under me"
    Node* old_head = head.load();
    while (old_head &&
           !head.compare_exchange_weak(old_head, old_head->next)) {}

    std::shared_ptr<T> result;
    if (old_head) result = std::make_shared<T>(std::move(old_head->data));

    try_reclaim(old_head);   // only frees if it's now safe (see below)
    return result;
}

void try_reclaim(Node* old_head) {
    if (threads_in_pop == 1) {
        // I'm the ONLY thread in pop() right now — safe to delete
        Node* nodes = to_be_deleted.exchange(nullptr);
        if (--threads_in_pop == 0) delete_nodes(nodes);
        else if (nodes) chain_pending_delete(nodes);  // put back for later
        delete old_head;
    } else {
        chain_pending_delete(old_head);   // add to "to delete later" list instead
        --threads_in_pop;
    }
}
```

**Idea:** if you're the *only* thread currently inside `pop()`, nobody else can be
holding a stale pointer, so it's safe to actually delete. Otherwise, queue the
node up for later ("to be deleted") and check again next time.

| ✅ Pros | ❌ Cons |
|---|---|
| Conceptually simple | Deletion is "bursty" — only happens when contention briefly drops to one thread; can build up a backlog under constant high load |

---

## 6. Reclamation via Reference Counting

Give each node a **reference count** of how many threads currently hold a pointer
to it (using `std::atomic<int>` or a std::shared_ptr with atomic operations).
A node is only actually deleted once its ref-count hits zero.

```
Node.refcount = 2   (Thread A holds a pointer, Thread B holds a pointer)
Thread A finishes → refcount-- → 1
Thread B finishes → refcount-- → 0 → NOW it's safe to delete
```

```cpp
struct Node {
    T data;
    std::atomic<int> internal_count{0};
    Node* next;
};

std::shared_ptr<T> pop() {
    Node* old_head = head.load();
    for (;;) {
        increase_head_count(old_head);      // "I'm about to use this node"
        if (!old_head) return nullptr;
        if (head.compare_exchange_strong(old_head, old_head->next)) {
            T res = std::move(old_head->data);
            int count = old_head->internal_count.fetch_sub(2);  // release our claim
            if (count == 1) delete old_head;   // we were the last reference
            return std::make_shared<T>(std::move(res));
        } else if (old_head->internal_count.fetch_sub(1) == 1) {
            delete old_head;   // CAS failed, and we were still last one out
        }
    }
}
```

| ✅ Pros | ❌ Cons |
|---|---|
| Deletion happens as soon as it's truly safe, no backlog buildup | Every node needs an atomic counter — memory + CAS overhead per node |

---

## 7. Reclamation via Hazard Pointers

Each thread publishes, in a well-known shared array, **which node(s) it is
currently accessing** ("this pointer is hazardous to delete right now"). Before
actually freeing a node, a thread scans that array — if any thread has marked the
node as hazardous, deletion is deferred.

```
Hazard Pointer Array (shared, one slot per thread):
 Thread1_hazard → NodeX     ← "I'm reading NodeX, don't delete it!"
 Thread2_hazard → nullptr   ← not reading anything right now

Thread B wants to delete NodeX:
   scan hazard array → sees Thread1 has it marked → DEFER deletion, try later
```

```cpp
std::atomic<Node*>& get_hazard_pointer_for_this_thread();  // simplified

std::shared_ptr<T> pop() {
    std::atomic<Node*>& hp = get_hazard_pointer_for_this_thread();
    Node* old_head = head.load();
    Node* temp;
    do {
        temp = old_head;
        hp.store(old_head);            // publish: "I'm about to touch this"
        old_head = head.load();
    } while (old_head != temp);        // make sure head didn't change mid-publish

    while (old_head &&
           !head.compare_exchange_strong(old_head, old_head->next)) {
        hp.store(old_head);
    }
    hp.store(nullptr);                 // done using it

    std::shared_ptr<T> result;
    if (old_head) {
        result = std::make_shared<T>(std::move(old_head->data));
        if (!is_hazardous(old_head))   // scan hazard array
            delete old_head;
        else
            add_to_reclaim_later_list(old_head);
    }
    return result;
}
```

| ✅ Pros | ❌ Cons |
|---|---|
| No per-node atomic counter needed | Requires a shared hazard-pointer array + periodic scanning |
| Well-studied, used in real lock-free libraries | More bookkeeping code than reference counting |

### Comparing the three approaches

| Strategy | Overhead location | Deletion timing |
|---|---|---|
| Thread counting | Global "threads in pop" counter | Bursty — only when contention drops to 1 |
| Reference counting | Per-node atomic counter | Immediate, as soon as last reference drops |
| Hazard pointers | Shared per-thread hazard array | Deferred + periodic scan/cleanup |

---

## ✅ Quick Recap

- Lock-free ≠ no synchronization — it means no OS mutex; threads instead retry CAS loops, and the system as a whole always makes progress even if individual threads retry.
- A lock-free stack's push/pop are both "read current head → build new state → CAS it in → retry on failure."
- The hard part of lock-free structures is **memory reclamation**: you can't `delete` a node the instant it's logically removed, because another thread might still be mid-read on it.
- Three classic solutions: thread counting (delete only when you're the sole thread in the danger zone), reference counting (delete when a per-node counter hits zero), and hazard pointers (publish "I'm using this" and check before deleting).

**Next:** `06_thread_pools.md` — reusing a fixed set of worker threads instead of spawning a new thread per task.
