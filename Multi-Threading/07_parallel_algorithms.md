# ⚡ 07 — Parallel Algorithms

## 📑 Table of Contents
- [1. The General Pattern: Divide, Compute, Combine](#1-the-general-pattern-divide-compute-combine)
- [2. Parallel Accumulate](#2-parallel-accumulate)
- [3. Parallel Accumulate with async](#3-parallel-accumulate-with-async)
- [4. Parallel quicksort](#4-parallel-quicksort)
- [5. Parallel for_each](#5-parallel-for_each)
- [6. Parallel find](#6-parallel-find)
- [7. Partial Sum (Parallel Prefix Sum)](#7-partial-sum-parallel-prefix-sum)
- [8. Parallel Matrix Multiplication](#8-parallel-matrix-multiplication)
- [9. Parallel Matrix Transpose](#9-parallel-matrix-transpose)
- [10. C++17 Parallel STL](#10-c17-parallel-stl)
- [11. Factors Affecting Concurrent Performance](#11-factors-affecting-concurrent-performance)

---

## 1. The General Pattern: Divide, Compute, Combine

💡 **In plain English:** if you need to peel 100 potatoes, and you have 4 chefs,
the obvious move is: split the potatoes into 4 piles of 25, give one pile to each
chef, let them all peel at the same time, then combine the peeled potatoes back
into one big bowl at the end. Nearly every "parallel algorithm" in this file is
just that idea, applied to numbers or data instead of potatoes.

Almost every parallel algorithm in this file follows the same shape:

```
       [ Full data range ]
              │
   split into N chunks (N ≈ hardware_concurrency())
              │
   ┌──────┬──────┬──────┬──────┐
   ▼      ▼      ▼      ▼
 Chunk1 Chunk2 Chunk3 Chunk4   ← each processed by its OWN thread, independently
   │      │      │      │
   └──────┴──────┴──────┘
              │
      combine partial results  (sum them, merge them, etc.)
```

**Key requirement:** each chunk's work must be **independent** (no shared mutable
state between chunks) — otherwise you're back to needing mutexes, which usually
erases the benefit of parallelizing in the first place.

---

## 2. Parallel Accumulate

Goal: sum a large range, splitting the work across threads.

```cpp
template<typename Iterator, typename T>
void accumulate_block(Iterator first, Iterator last, T& result) {
    result = std::accumulate(first, last, result);
}

template<typename Iterator, typename T>
T parallel_accumulate(Iterator first, Iterator last, T init) {
    unsigned long length = std::distance(first, last);
    if (length == 0) return init;

    unsigned num_threads = std::thread::hardware_concurrency();
    unsigned long block_size = length / num_threads;

    std::vector<T> results(num_threads);
    std::vector<std::thread> threads(num_threads - 1);

    Iterator block_start = first;
    for (unsigned i = 0; i < num_threads - 1; ++i) {
        Iterator block_end = block_start;
        std::advance(block_end, block_size);
        threads[i] = std::thread(accumulate_block<Iterator, T>,
                                  block_start, block_end, std::ref(results[i]));
        block_start = block_end;
    }
    accumulate_block(block_start, last, results.back());  // last chunk on THIS thread

    for (auto& t : threads) t.join();

    return std::accumulate(results.begin(), results.end(), init);
}
```

```
[1..100] split across 4 threads →
   T1: sum(1..25)=325   T2: sum(26..50)=950   T3: sum(51..75)=1575   T4: sum(76..100)=2200
                              combine: 325+950+1575+2200 = 5050
```

---

## 3. Parallel Accumulate with async

Same idea, but let `std::async` (file 03) handle thread creation, and rely on
recursive splitting instead of manual chunk math — often cleaner:

```cpp
template<typename Iterator, typename T>
T parallel_accumulate_async(Iterator first, Iterator last, T init) {
    unsigned long length = std::distance(first, last);
    unsigned long threshold = 1000;   // below this, just do it directly — no benefit splitting further
    if (length <= threshold)
        return std::accumulate(first, last, init);

    Iterator mid = first;
    std::advance(mid, length / 2);

    std::future<T> first_half = std::async(parallel_accumulate_async<Iterator, T>,
                                             first, mid, init);
    T second_half_result = parallel_accumulate_async(mid, last, T{});

    return first_half.get() + second_half_result;
}
```
**Note the threshold check:** splitting all the way down to single elements would
create *way* more `async` calls than cores can usefully run — always stop
recursing once a chunk is "small enough" that sequential work is cheaper than the
overhead of spawning more parallelism.

---

## 4. Parallel quicksort

Classic divide-and-conquer sort — naturally parallel because the two partitions
(elements less than pivot, elements greater) are **completely independent** once
split.

```
        [ pivot | less-than-pivot | greater-than-pivot ]
                        │                    │
              sort on ANOTHER thread    sort on THIS thread (recursion)
                        │                    │
                        └────── combine ──────┘
```

```cpp
template<typename T>
std::list<T> parallel_quicksort(std::list<T> input) {
    if (input.size() < 2) return input;

    std::list<T> result;
    result.splice(result.begin(), input, input.begin());   // pivot = first element
    T pivot = *result.begin();

    auto divide_point = std::partition(input.begin(), input.end(),
                                        [&](const T& t) { return t < pivot; });

    std::list<T> lower_part;
    lower_part.splice(lower_part.end(), input, input.begin(), divide_point);

    // sort the lower half on a NEW thread (async), upper half recursively HERE
    std::future<std::list<T>> new_lower =
        std::async(&parallel_quicksort<T>, std::move(lower_part));

    result.splice(result.end(), parallel_quicksort(std::move(input)));  // upper half
    result.splice(result.begin(), new_lower.get());                    // lower half

    return result;
}
```

**Caution:** naive recursive `async` for every partition, like naive accumulate,
can spawn far too many threads for deeply recursive input (`std::async`'s default
launch policy may fall back to running things on the calling thread when
oversubscribed — which helps, but a thread-pool-based version is usually better in
production).

---

## 5. Parallel for_each

Apply a function to every element, independently, across chunks — nearly
identical shape to parallel accumulate, minus the "combine" step (no result to
merge, unless the function itself accumulates something).

```cpp
template<typename Iterator, typename Func>
void parallel_for_each(Iterator first, Iterator last, Func f) {
    unsigned long length = std::distance(first, last);
    if (length == 0) return;

    unsigned num_threads = std::thread::hardware_concurrency();
    unsigned long block_size = length / num_threads;

    std::vector<std::future<void>> futures(num_threads - 1);
    Iterator block_start = first;
    for (unsigned i = 0; i < num_threads - 1; ++i) {
        Iterator block_end = block_start;
        std::advance(block_end, block_size);
        futures[i] = std::async(std::launch::async, [=, &f] {
            std::for_each(block_start, block_end, f);
        });
        block_start = block_end;
    }
    std::for_each(block_start, last, f);   // last chunk on this thread

    for (auto& fut : futures) fut.get();   // wait for all chunks, propagate exceptions
}
```

---

## 6. Parallel find

Search for a value across chunks in parallel — the first chunk to find a match
should ideally **signal the others to stop early** rather than scanning the whole
range regardless.

### With packaged_task (manual thread management)
```cpp
template<typename Iterator, typename T>
Iterator parallel_find(Iterator first, Iterator last, T match, std::atomic<bool>& done) {
    // each chunk's worker checks `done` periodically and bails out early if another
    // chunk already found the match — avoids wasted work
    for (auto it = first; it != last && !done; ++it) {
        if (*it == match) {
            done = true;
            return it;
        }
    }
    return last;
}
```

### With std::async (simpler, but harder to cancel early)
```cpp
template<typename Iterator, typename T>
Iterator parallel_find_async(Iterator first, Iterator last, T match) {
    unsigned long length = std::distance(first, last);
    unsigned long threshold = 1000;
    if (length < threshold)
        return std::find(first, last, match);

    Iterator mid = first;
    std::advance(mid, length / 2);

    std::future<Iterator> first_half =
        std::async(&parallel_find_async<Iterator, T>, first, mid, match);
    Iterator second_result = parallel_find_async(mid, last, match);

    if (first_half.get() != mid) return first_half.get();
    return second_result;
}
```

| Approach | Can stop early once found? | Complexity |
|---|---|---|
| `packaged_task` + shared `atomic<bool> done` | ✅ Yes, cooperative check | Higher — manual thread/flag management |
| `std::async` recursive split | ❌ Not really (still `.get()`s both halves) | Lower — simpler code |

---

## 7. Partial Sum (Parallel Prefix Sum)

Goal: `result[i] = input[0] + input[1] + ... + input[i]` for every `i` — trickier
to parallelize than a plain sum because **each output depends on all previous
inputs**, not just its own chunk.

```
input:        [ 1   2   3   4   5   6   7   8 ]
partial sum:  [ 1   3   6  10  15  21  28  36 ]
```

**Two-pass approach:**
1. Each thread computes the partial sum **within its own chunk** independently (parallel).
2. Compute each chunk's total, then each chunk adds "the sum of all chunks before it" to every one of its own elements (a small serial pass over chunk totals, then a second parallel pass to apply the offset).

```
Chunk1: [1,2,3,4] → local partial sums [1,3,6,10]         (chunk total = 10)
Chunk2: [5,6,7,8] → local partial sums [5,11,18,26]       (chunk total = 26)

Serial step: chunk2's offset = chunk1's total = 10

Chunk2 final (add offset 10 to each): [15,21,28,36]
Final combined result: [1,3,6,10, 15,21,28,36]
```

```cpp
template<typename Iterator>
void parallel_partial_sum(Iterator first, Iterator last) {
    using ValueType = typename Iterator::value_type;
    unsigned long length = std::distance(first, last);
    if (length <= 1) return;

    unsigned num_chunks = std::thread::hardware_concurrency();
    unsigned long block_size = length / num_chunks;

    std::vector<ValueType> chunk_totals(num_chunks);
    std::vector<std::future<void>> futures;

    // Pass 1: local partial sums per chunk, remember each chunk's final total
    Iterator block_start = first;
    for (unsigned i = 0; i < num_chunks; ++i) {
        Iterator block_end = (i == num_chunks - 1) ? last : std::next(block_start, block_size);
        futures.push_back(std::async(std::launch::async, [block_start, block_end, i, &chunk_totals] {
            std::partial_sum(block_start, block_end, block_start);
            chunk_totals[i] = *std::prev(block_end);
        }));
        block_start = block_end;
    }
    for (auto& f : futures) f.get();

    // Pass 2 (serial, cheap): running offset from previous chunk totals
    ValueType offset = 0;
    std::vector<ValueType> offsets(num_chunks);
    for (unsigned i = 0; i < num_chunks; ++i) {
        offsets[i] = offset;
        offset += chunk_totals[i];
    }

    // Pass 3 (parallel): apply each chunk's offset to all its elements
    block_start = first;
    futures.clear();
    for (unsigned i = 0; i < num_chunks; ++i) {
        Iterator block_end = (i == num_chunks - 1) ? last : std::next(block_start, block_size);
        if (offsets[i] != 0) {
            futures.push_back(std::async(std::launch::async, [block_start, block_end, offset = offsets[i]] {
                std::for_each(block_start, block_end, [offset](ValueType& v) { v += offset; });
            }));
        }
        block_start = block_end;
    }
    for (auto& f : futures) f.get();
}
```

---

## 8. Parallel Matrix Multiplication

For `C = A × B`, each output element `C[i][j]` is an **independent dot product**
of row `i` of A and column `j` of B — perfect for parallelizing across rows (or
individual cells) since there's zero shared mutable state between output cells.

```
   A (rows)   ×   B (cols)   =   C
  [row 0]  ─┐                  [row 0 of C]   ← independent
  [row 1]  ─┼── each row's     [row 1 of C]   ← independent
  [row 2]  ─┘   output row      [row 2 of C]   ← independent
             computed on its own thread
```

```cpp
void parallel_matrix_multiply(const Matrix& A, const Matrix& B, Matrix& C) {
    int rows = A.rows();
    unsigned num_threads = std::min<unsigned>(rows, std::thread::hardware_concurrency());
    std::vector<std::thread> threads;

    auto compute_rows = [&](int row_start, int row_end) {
        for (int i = row_start; i < row_end; ++i)
            for (int j = 0; j < B.cols(); ++j) {
                double sum = 0;
                for (int k = 0; k < A.cols(); ++k)
                    sum += A(i, k) * B(k, j);
                C(i, j) = sum;   // each thread writes to DIFFERENT rows — no race
            }
    };

    int chunk = rows / num_threads;
    for (unsigned t = 0; t < num_threads; ++t) {
        int start = t * chunk;
        int end = (t == num_threads - 1) ? rows : start + chunk;
        threads.emplace_back(compute_rows, start, end);
    }
    for (auto& th : threads) th.join();
}
```

---

## 9. Parallel Matrix Transpose

`Transposed[j][i] = Original[i][j]` — again, every output cell is independent, so
splitting by rows (or blocks, for cache-friendliness) works the same way as
multiplication.

```
Original:            Transposed:
[1 2 3]               [1 4]
[4 5 6]      ──►       [2 5]
                       [3 6]
```

```cpp
void parallel_transpose(const Matrix& src, Matrix& dst) {
    unsigned num_threads = std::thread::hardware_concurrency();
    int rows = src.rows();
    int chunk = rows / num_threads;
    std::vector<std::thread> threads;

    auto transpose_rows = [&](int start, int end) {
        for (int i = start; i < end; ++i)
            for (int j = 0; j < src.cols(); ++j)
                dst(j, i) = src(i, j);   // different (i,j) per thread → no overlap
    };

    for (unsigned t = 0; t < num_threads; ++t) {
        int start = t * chunk;
        int end = (t == num_threads - 1) ? rows : start + chunk;
        threads.emplace_back(transpose_rows, start, end);
    }
    for (auto& th : threads) th.join();
}
```

---

## 10. C++17 Parallel STL

Instead of hand-writing chunking logic, C++17 added **execution policies** to
many standard algorithms — the standard library does the chunking/threading for
you internally.

```cpp
#include <execution>

std::vector<int> data = /* ... */;

std::sort(std::execution::par, data.begin(), data.end());               // parallel sort
std::for_each(std::execution::par, data.begin(), data.end(), [](int& x) { x *= 2; });
int total = std::reduce(std::execution::par, data.begin(), data.end()); // parallel sum
```

| Policy | Meaning |
|---|---|
| `std::execution::seq` | Sequential (normal, no parallelism) |
| `std::execution::par` | Parallel, order of individual steps unspecified |
| `std::execution::par_unseq` | Parallel **and** allows vectorization (steps may interleave/run on SIMD) |

**Trade-off:** far less code than hand-rolled chunking (sections 2–9), but less
control — you can't customize chunk size, thread count, or add early-exit logic
like the custom `parallel_find` above.

---

## 11. Factors Affecting Concurrent Performance

| Factor | Effect |
|---|---|
| **Number of cores vs number of threads** | Oversubscription (file 06, section 2) hurts — more threads than cores adds context-switch overhead, not speed |
| **Data contention** | Threads fighting over the same cache line/lock serializes them — defeats parallelism |
| **False sharing** | Two threads writing to *different* variables that happen to sit in the *same CPU cache line* cause expensive cache invalidation traffic between cores, even though there's no logical data race |
| **Chunk size / granularity** | Too fine-grained (tiny chunks) → thread overhead dominates; too coarse (huge chunks) → poor load balancing across cores |
| **Task dependencies** | If chunks aren't truly independent (partial sum's cross-chunk dependency, section 7), you need extra synchronization passes, cutting into the speedup |
| **Memory bandwidth** | Many algorithms (matrix ops) are memory-bound, not CPU-bound — adding threads doesn't help once you saturate memory bandwidth |
| **Load imbalance** | If chunks take unequal time (e.g. quicksort partitions of very different sizes), some threads finish early and idle — work stealing (file 06) helps here |

**False sharing example:**
```cpp
struct Counters {
    std::atomic<int> a;   // written by Thread 1
    std::atomic<int> b;   // written by Thread 2 — but `a` and `b` are on the
};                        // SAME cache line → every write bounces the line
                           // between cores' caches, even though a/b are logically unrelated
```
**Fix:** pad/align hot per-thread data to separate cache lines (e.g.
`alignas(64)`) so unrelated variables used by different threads don't collide.

---

## ✅ Quick Recap

- Parallel algorithms follow "split into independent chunks → process each on its own thread → combine results."
- Accumulate/for_each/find/matrix ops are naturally parallel because each chunk/cell is independent; partial sum needs an extra serial pass because outputs depend on earlier inputs.
- Always add a size threshold before recursively splitting further — infinite splitting creates more threads/tasks than cores can use.
- C++17's Parallel STL (`std::execution::par`) gives you parallelism for free on standard algorithms, at the cost of less fine control.
- Real-world performance depends on more than "did I use threads" — oversubscription, false sharing, chunk granularity, and memory bandwidth all matter.

**Next:** `08_cpp20_concurrency.md` — jthread, coroutines, and barriers.
