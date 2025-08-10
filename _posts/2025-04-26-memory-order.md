---
layout: post
title: "Memory order (in C++20)"
---

In a concurrent program, memory ordering describes the guarantees an implementation gives about the visibility and relative ordering of memory accesses performed by multiple threads. Modern compilers and processors freely re-order ordinary loads and stores to improve performance, as long as each individual thread behaves as if it executed the program in source order. Typical optimizations include:

* **Compiler instruction scheduling** (a compile-time re-ordering pass)
* **Out-of-order or speculative execution** inside the CPU
* **Store buffering**, where each core keeps recent writes in a private buffer until they are flushed to the shared cache

Without extra constraints, two threads that share data may observe each other’s writes in apparently impossible orders, making the program nondeterministic and extremely hard to reason about.

Before C++11 the language had no formal memory model; programmers were limited to whatever guarantees a given compiler–hardware pair provided. Since C++11 (and with only incremental tweaks through C++20) the Standard defines a language-level memory model and a set of atomic operations that let us specify exactly how much ordering we need.

In that model, **happens-before** is the fundamental ordering relation. An evaluation `A` **happens-before** evaluation `B` if

1. A is sequenced-before B in the same thread,
2. A synchronises-with B through an atomic operation or fence, or
3. there exists a transitive chain of evaluations that satisfy (1) or (2).

The implementation must guarantee that every thread perceives memory effects in an order consistent with this global happens-before relation, which is required to be acyclic. Any conflicting accesses that are not ordered by happens-before form a data race, and the behaviour of the entire program becomes undefined.

## The six memory-order constants

The header `<atomic>` defines

```c++
enum class std::memory_order { relaxed, consume, acquire,
                               release, acq_rel, seq_cst };
```

Each enumerator also has an inline constexpr alias such as `std::memory_order_relaxed`; both spellings are standard and equivalent.

| Constant  | What it guarantees                                                                                                                | Typical use-case                                                                                                                     |
| --------- | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `relaxed` | Atomicity only; no inter-thread ordering or synchronisation.                                                                      | Statistics counters, reference counts, ID generators where the exact order of increments is irrelevant.                              |
| `consume` | Orders subsequent operations that are data-dependent on the loaded value; all major compilers currently promote it to `acquire`.  | Extremely rare in portable code; avoid until compilers implement true consume semantics.                                             |
| `acquire` | Prevents later reads/writes from being reordered before the operation (one-way barrier).                                          | Thread that loads a flag and then reads the published data.                                                                          |
| `release` | Prevents earlier reads/writes from being reordered after the operation (mirror one-way barrier).                                  | Thread that writes data and then stores a flag, or the unlock side of a mutex.                                                       |
| `acq_rel` | A single RMW acts as both acquire and release; on a plain load it is ill-formed, and on a plain store it degrades to release.     | fetch_add, compare_exchange, ticket spin-locks, any RMW that must publish its own update and forbid later code from moving above it. |
| `seq_cst` | Behaves as acquire on loads and release on stores and all seq_cst operations across all threads appear in one global total order. | Default; easiest to reason about, but may block hardware optimisations on some weak architectures.                                   |

Understanding and choosing the weakest order that still preserves correctness is the key to high-performance lock-free code. In the next sections we’ll walk through concrete code snippets to see how each order affects real executions.

## What is `seq_cst` memory ordering?

`memory_order_seq_cst` (sequentially consistent) is the strongest ordering offered by C++:

**Per-thread ordering:**

A seq-cst load acts as an acquire, a seq-cst store acts as a release, and a seq-cst RMW acts as acq_rel. In other words, you get the same one-way fences as acquire/release, not a magical full barrier.

**Visibility:**

Te acquire/release effects ensure that every write sequenced-before a seq-cst store is visible after a seq-cst load that reads the stored value.

**Single global order:**

All seq-cst operations, across all threads and variables, appear in one total order that every thread agrees on. That extra guarantee makes reasoning easier but can cost performance on weakly-ordered CPUs.

### Ordering and visibility example

```c++
std::atomic<bool> flag(false);
int x = 0; // NOT ATOMIC
int y = 0; // NOT ATOMIC

// Thread 1
void thread_1()
{
    x++; // A
    flag.store(true /* std::memory_order::seq_cst */); // B (seq_cst release)
    y++; // C   (may move *before* B)
}

// Thread 2
void thread_2()
{
    int ry = y; // D
    while (!flag.load(/* std::memory_order::seq_cst */)) {} // E (seq_cst acquire)
    int rx = x; // F (sees the effect of A)
}
```

Re-ordering rules:

* A may not move after B (release fence), and B itself may not move before A.
* F may not move before E (acquire fence).
* C can move before B, and D can move after E—seq-cst does not forbid those moves because the fences are one-way.

**Visibility:**

If the acquire load E observes true, then rx must read the value stored by A (1).

**Intentional data race ⚠**

Because `y` is not atomic, a re-order where `C` slides in front of `B` and overlaps with `D` creates a bona-fide data race → undefined behaviour. The race is deliberate here to illustrate that ordering guarantees alone do not make non-atomic shared data safe.

### Single global order example

This is one additional constrain, ensures that seq_cst atomic operations will be executed "one-by-one". `A -> B -> C -> D`, where each letter is atomic operation and `A -> B` means that `A` happens before `B`.

```c++
std::atomic<int> x(0);
std::atomic<int> y(0);

int ry = 0;
int rx = 0;

// Thread 1
void thread_1()
{
    x.store(1 /* std::memory_order::seq_cst */); // A
    ry = y.load(/* std::memory_order::seq_cst */); // B
}

// Thread 2
void thread_2()
{
    y.store(1 /* std::memory_order::seq_cst */); // C
    rx = x.load(/* std::memory_order::seq_cst */); // D
}
```

Because every operation is `seq_cst`, the Standard requires that all four actions appear in a single total order that every thread observes in the same way. Intuitively, the actions are “serialised” into some sequence such as `A → B → C → D`.

**What does this total order forbid?**
At least one of the two stores (`A` or `C`) must become visible before its matching load executes, so the result `rx == 0 && ry == 0` is impossible.

**Possible outcomes:**

| First store visible              | Order prefix in the total order  | (ry, rx) after boh threads finish |
| -------------------------------- | -------------------------------- | --------------------------------- |
| `x = 1` (A)                      | A -> B -> ...                    | (0, 1)                            |
| `y = 1` (C)                      | C -> D -> ...                    | (1, 0)                            |
| Both stores visible before loads | A -> C -> ... *or* C -> A -> ... | (1, 1)                            |

Any architecture or compiler that claims conformance must rule out the forbidden (0, 0) outcome; weaker orders such as `acq_rel` do allow it.

## What is `acq_rel` memory order?

Acquire-release (std::memory_order::acq_rel) gives the same guarantees as seq_cst, without Single global order. It's used in RMW (Read-Modify-Write) operations where it must be ensured that two-way barrier (full-barrier) is required.

std::memory_order_acq_rel is meant only for read-modify-write (RMW) operations (e.g. fetch_add, compare_exchange, fetch_or).
On such an operation it acts as

* release with respect to every read and write that precedes the RMW, and
* acquire with respect to every read and write that follows it.

Unlike seq_cst, it does not place the operation in the single global total order, so you lose that last bit of cross-thread coordination—often a worthwhile trade-off for speed.

### Common usage: ticket spin-lock

```c++
class spinlock {
    std::atomic<int> next_{0};     // RMW counter
    std::atomic<int> owner_{0};    // plain atomic

public:
    void lock() {
        int my = next_.fetch_add(1, std::memory_order_acq_rel);   // A
        while (owner_.load(std::memory_order_acquire) != my) { }  // B
    }
    void unlock() {
        owner_.store(my + 1, std::memory_order_release);          // C
    }
};
```

* A must be acq_rel: as a store it publishes the new ticket to other threads (release); as a load it prevents later critical-section reads from moving up (acquire).
* B needs only acquire; it does not publish anything.
* C needs only release; no subsequent reads depend on it.

### Lack of sigle global order

Replace the seq_cst operations from the previous litmus test with true RMWs that use acq_rel:

```c++
std::atomic<int> x(0);
std::atomic<int> y(0);

int ry = 0;
int rx = 0;

// Thread 1 (Core 1)
void thread_1()
{
    x.fetch_add(1, std::memory_order::acq_rel); // A
    ry = y.fetch_add(0, std::memory_order::acq_rel); // B
}

// Thread 2 (Core 2)
void thread_2()
{
    y.fetch_add(1, std::memory_order::acq_rel); // D
    rx = x.fetch_add(0, std::memory_order::acq_rel); // E
}
```

See following example execution that one of the possible scenarios. Let's assume that it's executed on multicore CPU, and Thread 1 is executed by Core 1 and Thread 2 is executed by Core 2.

**Time t~0~:**

* Thread: 1 and 2
* Operation ID: `A` and `D`

Each core increments its own counter; the writes sit in the respective store buffers.

**Time t~1~:**

* Thread: 1
* Operation ID: `B`

`ry == 0`, increment to `y` not yet visible.

**Time t~2~:**

* Thread: 2
* Operation ID: `E`

`rx == 0`, increment to `x` not yet visible.

**Time t~3~:**

Store buffers drain; both writes become globally visible.

Outcome `rx == 0 && ry == 0` was impossible under `seq_cst`, but is allowed with `acq_rel` because there is no single total order forcing one store to precede both loads.

## What is acquire memory ordering?

A load performed with std::memory_order_acquire provides two key guarantees:

* Ordering – no later read or write in the same thread may move before the load (one-way barrier).
* Visibility – if the loaded value was stored by a memory_order_release (or stronger) store, then every write sequenced-before that store becomes visible after the load.

### Relative-ordering guarantee (one-way barrier)

```c++
std::atomic<bool> is_incremented(false);
int x = 0; // NOT ATOMIC
int y = 0; // NOT ATOMIC

// Thread 1 (producer)
void thread_1()
{
    x++;  // A
    is_incremented.store(true); // B
    const auto ry = y; // C
}

// Thread 2 (consumer)
void thread_2()
{
    y++; // D
    while(!is_incremented.load(std::memory_order::acquire)) {} // E
    const auto rx = x; // F
}
```

* The release/acquire pair (B ↔ E) forms a synchronises-with edge.
* D may move past E (it is before the barrier), but F cannot move before E.
* Because A is sequenced-before B, and B synchronises-with E, A happens-before F – therefore rx is guaranteed to read 1.
* The write D to y happens-before the read C via the chain D → E (acquire) → B (release) → C, so no data race exists even though y is non-atomic.

**Time t~0~:**

* Thread: 1
* Operation ID: `A`
* Comment: `x = 1`

**Time t~1~:**

* Thread: 1
* Operation ID: `B`

**Time t~2~:**

* Thread: 2
* Operation ID: `E`

Load sees true; loop ends;

**Time t~3~:**

* Thread: 1 and 2
* Operation ID: `C` and `D`

**⚠ Data race** - Thread 1 reads y while Thread 2 writes it -> undefined behaviour

**Time t~4~:**

* Thread: 2
* Operation ID: `F`

Reads of value x can not be reordered before E, acquire fence guarantees this.

⚠ This listing is intentionally racy. The undefined-behaviour paths are highlighted to illustrate what goes wrong when the release/acquire pair is mis- or under-used.

## What is release memory ordering?

std::memory_order::release is the store-side mirror of acquire.

* Ordering – no read or write that appears before the store in program
order may be moved after it (one-way barrier).
* Visibility – together with a matching memory_order_acquire load that
reads the same value, it creates an inter-thread happens-before edge that
makes all earlier writes visible.

### Relative-ordering guarantee (one-way barrier) for release

```c++
std::atomic<bool> is_incremented(false);
int x = 0; // NOT ATOMIC
int y = 0; // NOT ATOMIC

// Thread 1
void thread_1()
{
    x++;  // A
    is_incremented.store(true, std::memory_order::release); // B
    y++;  // C
}

// Thread 2
void thread_2()
{
    const auto ry = y; // D
    while(!is_incremented.load()) {} // E
    const auto rx = x; // F
}
```

* C may move in front of B, but A may not move after B.
* Because A is sequenced-before B, and B synchronises-with the acquire load E, A happens-before F ⇒ rx == 1.
* D can overlap C → true data race on y (shown intentionally).

|  Time  | Current Thread | Operation ID |                                    Comment                                    |
| :----: | :------------: | :----------: | :---------------------------------------------------------------------------: |
|   t0   |    1 and 2     |   C and D    | ⚠ Data race: Thread 1 writes y while Thread 2 reads it -> undefined behaviour |
| t0 + 1 |       1        |      A       |                  x = 1, can not be reordered after release.                   |
| t0 + 2 |       1        |      B       |                                                                               |
| t0 + 3 |       2        |      E       |                          load sees true; loop ends;                           |
| t0 + 4 |       1        |      F       |                                                                               |

⚠ This listing is intentionally racy. The undefined-behaviour paths are highlighted to illustrate what goes wrong when the release/acquire pair is mis- or under-used.

Release is therefore ideal for “publish a result / set a flag” and for the unlock side of a spin-lock or mutex implementation.

### Visibility guarantee

A release store does not instantly publish data; it merely promises that if another thread later performs an acquire load that observes the stored value, all earlier writes are already visible.

|  Time  | Current Thread | Operation ID |                                    Comment                                    |     |
| :----: | :------------: | :----------: | :---------------------------------------------------------------------------: | --- |
|   t0   |       1        |      A       |                   x = 1 (may still sit in the store buffer)                   |     |
| t0 + 1 |       1        |      B       |                             flag = true (release)                             |     |
| t0 + 2 |       —        |      —       |              Store buffer drains; x and flag reach shared cache.              |     |
| t0 + 3 |       2        |      E       | loads true (acquire) ⇒ all earlier writes by T1, including x, are now visible |     |
| t0 + 4 |       2        |      F       |                               reads x must be 1                               |     |

### Using release and acquire pair

Combination release and acquire is always used as pair, this gives it’s functionality in publisher and consumer data exchange algorithms.

release doesn’t mean that data is instantly published for acquire to use. It is guaranteed that if Thread 2 acquire operation sees change published by Thread 1 release than all writes and read before release are visible in Thread 2.

```c++
std::atomic<bool> is_incremented(false);
int x = 0; // NOT ATOMIC

// Thread 1 (producer)
void thread_1()
{
    x++;  // A
    is_incremented.store(true, std::memory_order::release); // B
}

// Thread 2 (consumer)
void thread_2()
{
    while(!is_incremented.load()) {} // C
    const auto rx = x; // D
}
```

|  Time  | Current Thread | Operation ID |                              Comment                               |     |
| :----: | :------------: | :----------: | :----------------------------------------------------------------: | --- |
|   t0   |       1        |      A       |             x = 1, can not be reordered after release.             |     |
| t0 + 1 |       1        |      B       |           store saves true in private core write buffer            |     |
| t0 + 2 |       2        |      C       |                  load sees false; loop continues;                  |     |
| t0 + 3 |       –        |      –       | store buffer drains; is_incremented == true is visible by Thread 2 |     |
| t0 + 4 |       1        |      F       |                  load sees true; loop continues;                   |     |

## What is relaxed memory ordering?

With memory_order_relaxed only atomicity is guaranteed—neither ordering nor synchronisation.

```c++
std::atomic<bool> is_incremented(false);
int x = 0; // NOT ATOMIC

// Thread 1
void thread_1()
{
    x++;  // A
    is_incremented.store(true, std::memory_order::relaxed); // B
}

// Thread 2
void thread_2()
{
    while(!is_incremented.load()) {} // C
    std::cout << x << std::endl; // D
}
```

Because the two operations in Thread 1 are relaxed and independent, the implementation may appear to execute B before A. One possible (perfectly legal) history is:

|  Time  | Current Thread | Operation ID |                                    Comment                                     |     |
| :----: | :------------: | :----------: | :----------------------------------------------------------------------------: | --- |
|   t₀   |       2        |      C       |                        load sees false; loop continues.                        |     |
| t0 + 1 |       1        |      B       |    is_increment set to true before x is incremented (A has been reordered).    |     |
| t0 + 2 |       2        |      C       |                           load sees true; loop ends.                           |     |
| t0 + 3 |    2 and 1     |   D and A    | ⚠ Data race: Thread 2 reads x while Thread 1 writes it -> undefined behaviour. |     |
| t0 + 4 |       1        |      F       |                        load sees true; loop continues;                         |     |

⚠ This listing is intentionally racy.

### No visibility guarantee

Even if A happens before B in program order, the new value of x may linger in Thread 1’s store buffer:

```c++
std::atomic<bool> is_incremented(false);
int x = 0; // NOT ATOMIC

// Thread 1
void thread_1()
{
    x++;  // A
    is_incremented.store(true, std::memory_order::relaxed); // B
}

// Thread 2
void thread_2()
{
    while(!is_incremented.load()) {} // C
    std::cout << x << std::endl; // D
    std::cout << x << std::endl; // E
}
```

something different can also happen.

Scenario: A is executed before B, but incremented value x is stored in private store buffer, and is_incremented atomic value has been published to main memory so all other thread can see it. This implies that operations executed in Thread 1 in order A -> B are visible to other Thread 2 as executed in order B -> A.

|  Time  | Current Thread | Operation ID |                      Comment                      |     |
| :----: | :------------: | :----------: | :-----------------------------------------------: | --- |
|   t₀   |       1        |      A       | x = 1, but value is still in core’s store buffer. |     |
| t₀ + 1 |       1        |      B       |     is_incremented published to shared cache      |     |
| t₀ + 2 |       2        |      C       |             load sees true; loop ends             |     |
| t₀ + 3 |       2        |      D       |              Prints 0, old x value.               |     |
| t₀ + 4 |       –        |      –       | Store buffer drains; x becomes globally visible.  |     |
| t₀ + 5 |       2        |      E       |                     Prints 1.                     |     |

⚠ This listing is intentionally racy.

Again, the behaviour is legal because neither thread imposed ordering between the write to x and the flag.

### Correct usage of relaxed memory ordering

relaxed memory order is the right choice when you need atomicity but no cross-thread ordering at all.
Typical patterns include:

* Statistical counters – e.g. events_total.fetch_add(1, std::memory_order_relaxed); where the exact order of increments is irrelevant.
* Non-blocking identifiers – generating unique IDs or ticket numbers.

Never use `relaxed` to signal that data at other addresses is ready; readers might see the flag and still observe stale data due to re-ordering or store buffering.