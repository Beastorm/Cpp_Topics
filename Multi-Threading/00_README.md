# 🧵 C++ Concurrency & Multithreading — Notes Index

These notes convert the full threading interview + course syllabus into digestible,
example-driven markdown files. Each file is self-contained and focused on **one theme**,
so nothing is a giant wall of text.

## 📂 File Map

| # | File | Covers |
|---|------|--------|
| 1 | `01_thread_basics.md` | What a thread is, 5 ways to create threads, lifecycle, join/detach, joinable(), passing params, ownership transfer, thread-local storage |
| 2 | `02_mutex_and_locking.md` | Critical sections, mutex, try_lock, timed/recursive/shared mutex, lock_guard, unique_lock, scoped_lock, std::lock, deadlock, livelock |
| 3 | `03_condition_variables_and_futures.md` | Condition variables, producer-consumer, std::promise/future, std::async, package_task, shared_future, exceptions across threads |
| 4 | `04_atomics_and_memory_model.md` | Atomic types, compare_exchange, memory ordering (relaxed/acquire-release/seq_cst/consume), spinlock, static variable thread-safety |
| 5 | `05_lock_free_data_structures.md` | Lock-free stack, memory reclamation (reference counting, hazard pointers) |
| 6 | `06_thread_pools.md` | Simple thread pool → waitable tasks → reduced contention → work stealing, object pools, oversubscription |
| 7 | `07_parallel_algorithms.md` | Parallel accumulate, quicksort, for_each, find, partial sum, matrix multiply/transpose, Parallel STL |
| 8 | `08_cpp20_concurrency.md` | jthread, coroutines (intro, resume, generators), barriers |
| 9 | `09_interview_quickfire.md` | Rapid-fire Q&A: semaphore vs mutex, sleep vs wait, race condition, binary semaphore, producer-consumer with semaphore |

## 🗺️ How to read this

- Start at file 1 and go in order — each file builds on the previous one's vocabulary.
- Every file has its own Table of Contents with jump links.
- Code examples are minimal and compilable (C++17 unless marked C++20).
- ✅ / ❌ tables are used for "when to use vs avoid."
- ASCII diagrams show timing/ownership visually wherever a picture beats a paragraph.

## Status

- [x] 01_thread_basics.md
- [x] 02_mutex_and_locking.md
- [x] 03_condition_variables_and_futures.md
- [x] 04_atomics_and_memory_model.md
- [x] 05_lock_free_data_structures.md
- [x] 06_thread_pools.md
- [x] 07_parallel_algorithms.md
- [x] 08_cpp20_concurrency.md
- [x] 09_interview_quickfire.md
