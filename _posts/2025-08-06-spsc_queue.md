---
layout: post
title: "Single Producer Single Consumer (SPSC) Queue"
---

In this post I walk through a practical SPSC queue for C++. Queues show up everywhere; the standard library even ships a general‑purpose `std::queue` (built on `std::deque` by default). That industrial‑strength container is perfectly fine in single‑threaded code, but once two threads need to pass data, you have to add a synchronization layer around it. Rather than tackling every possible producer/consumer combination, I focus on a single‑producer, single‑consumer design. It’s easier to reason about, performs well, and serves as a good foundation for future variants.

To give a rounded view of the trade‑offs, I present three implementations of the same queue:

* Single‑threaded (no synchronization) — a simple baseline.
* Lock‑based SPSC — a minimal change: guard critical sections with a mutex.
* Mostly lock‑free SPSC — rely on atomics and memory ordering to avoid a mutex.
This trio isn’t meant to be interchangeable in every scenario; the point is to compare behaviour and overhead under the same workload and interface.

## Single‑threaded baseline (no synchronization)

The single‑threaded version exists to nail down semantics and provide a performance baseline. It computes the slot as
`m_write_idx & m_mask`, checks how far the write index is ahead of the read index, and if the buffer is full it destroys
 the oldest element and advances `m_read_idx`. Then it constructs the new element in place and increments `m_write_idx`.

Dequeue is a little simpler: if `m_write_idx == m_read_idx` the queue is empty and we return false. Otherwise we
compute the read slot, move the element out, destroy it in place, bump `m_read_idx`, and return true.

This data structure has some obvious upsides. It’s straightforward to implement because it doesn’t require any
additional synchronization logic on top of the core algorithm. That said, there are still real design choices to make
before you write a single line of code. You have to decide on a memory-allocation strategy, whether the queue should be
bounded or unbounded, and what underlying container fits best—linked list, dynamic array, or some hybrid approach. In a
single-threaded setting it should also be faster than queues designed primarily for multithreaded workloads, since
you’re not paying for synchronization that you don’t need. Rather than guessing, I’ll measure it; the usual wisdom
applies: "don’t assume anything about performance—measure it".

To make those measurements meaningful, we need a set of common scenarios that produce data you can actually compare.
Even then, a direct comparison between single-threaded and multithreaded runs isn’t truly apples to apples. In practice,
I struggle to think of a realistic case where a single-threaded implementation could stand in as a drop-in replacement
for a multithreaded one, or vice versa, so I treat those results as complementary, not interchangeable.

For this article I use one consistent baseline across all implementation variants: a bounded queue with a compile-time
fixed capacity. One detail that makes this interesting is that enqueue is forced—it cannot fail. When the queue is full
, the oldest element is erased and the new element takes its place in that slot. Dequeue, on the other hand, can fail,
but the exact conditions depend on the specific variant of the queue. This deliberate asymmetry between `enqueue` and
`try_dequeue` deserves some thought during implementation, and it maps cleanly to real-world needs. A producer may
continuously generate data while a consumer only cares about the most recent items, so it’s reasonable to accept
slightly more complex synchronization to ensure the consumer reliably sees the freshest element.

```c++
namespace spsc_queue {
    template<typename data_T, std::size_t size_T>
    class queue_t {
    public:
        static_assert(size_T >= 2, "Queue size must be at least 2");
        static_assert(std::has_single_bit(size_T), "Size of the queue must be power of 2");

        queue_t() = default;
        ~queue_t();

        auto enqueue(data_T element) -> void;
        auto try_dequeue(data_T& element_out) -> bool;
        auto try_discard() -> bool;

    private:
        struct cell_t {
            alignas(data_T) std::byte m_buffer[sizeof(data_T)];
        };
        static constexpr std::size_t m_mask = size_T - 1;
        std::size_t m_write_idx = 0;
        std::size_t m_read_idx = 0;
        std::array<cell_t, size_T> m_cells;
    };
}
```

A small but important contract is enforced at compile time with `static_assert`: the queue’s capacity must be a power
of two. This guarantees a fast index wrap-around using a bit mask and avoids ambiguity in full/empty detection.

### Size as a power of two

```c++
static_assert(size_T >= 2, "Queue size must be at least 2");
static_assert(std::popcount(size_T) == 1, "Size of the queue must be power of 2");
```

Why insist on a power-of-two size? It’s a classic optimization that simplifies index arithmetic, cuts branches, and
speeds up wrap-around. In a ring buffer you always need to move from the current slot to the next and, when you fall
off the end, jump back to zero. The straightforward approach is to check the index and, if it’s greater than or equal
to the array size, reset it to 0; that adds a conditional on the hot path. Another common approach is to compute
`(index + 1) % size`, which avoids the branch but still pays for a modulo. If the capacity is a power of two, you can
compute the next position as `(index + 1) & (size - 1)`, which is branch-free and typically cheaper than a modulo, while
keeping the logic easy to reason about.

```c++
++m_write_idx;
if(m_write_idx == std::size(m_buffer)) {
  m_write_idx = 0;
}
```

That wrap-to-zero approach looks fine in isolation, but a queue maintains two counters—one for writes and one for
reads—so it introduces ambiguity. If both `m_write_idx` and `m_read_idx` end up at 0 after wrapping, you can no longer
tell whether the queue is empty or actually full because the write index has lapped the read index and been reset.
Without some extra state, the condition `m_write_idx == m_read_idx` becomes impossible to interpret unambiguously in
this scheme. A cleaner strategy is to let both indices grow monotonically for the entire lifetime of the structure,
which is practical if you know you won’t exceed `std::uint64_t` writes overall. With ever-increasing counters, you
compute the physical slot on each access using a modulo by the buffer size, which yields a stable mapping into the ring.
When the capacity is a power of two you can replace the modulo with a mask, but the key idea remains the same: never
reset the counters, derive the “real” index from them, and you get clear occupancy semantics because
`m_write_idx - m_read_idx` always reflects the number of elements currently enqueued.

```c++
++m_write_idx;
const auto write_idx = m_write_idx % size_T;
```

With this scheme you no longer need any extra state to describe the queue’s condition. The monotonically increasing
counters carry all the information you need: the queue is empty exactly when `m_write_idx == m_read_idx`, and the
physical slot is obtained by mapping the counter into the ring via the modulo. In practice, when the capacity is a
power of two you can replace the modulo with a simple mask, but the key point is that the indices are never reset, so
their difference encodes occupancy unambiguously while the masked value gives you the current array position.

Going a little deeper, we can assert the queue’s capacity is always a power of two. Take a queue of size 16 (2^4):
in base-2 that size is `0b10000`, which implies valid element positions range from `0b0000` to `0b1111`. Any bits above
those lower four don’t matter for addressing the ring. If we define a mask as `size_T - 1`, then for 16 the mask is
`0b1111`. Applying this mask to a monotonically increasing counter yields the physical slot: the “real” index is simply
`counter & (size_T - 1)`. In effect, the mask discards all higher-order bits and keeps only the low `log2(size_T)` bits,
giving you branch-free wrap-around without a modulo.

```c++
constexpr std::size_t m_mask = size_T - 1;
++m_write_idx;
const auto write_idx = m_write_idx & m_mask;
```

This unlocks a straightforward low-level optimization based on masking instead of modulo. To make the benefit concrete,
let’s compare the branchless, power-of-two mask against the more general modulo or wrap-to-zero approach and see how
each affects the hot path.

Module operation ([Compiler-Explorer](https://compiler-explorer.com/z/ss1x98ohK)):

```asm
movabs  rdx, -4392081922311798003
mov     rax, rdi
shr     rax
mul     rdx
mov     rax, rdx
shr     rax, 4
imul    rdx, rax, 42
mov     rax, rdi
sub     rax, rdx
ret
```

Mask operation ([Compiler-Explorer](https://compiler-explorer.com/z/ss1x98ohK)):

```asm
mov     rax, rdi
and     eax, 15
ret
```

### Structure of cell_t struct

Short note on usage of struct `cell_t`.

```c++
struct cell_t {
    alignas(data_T) std::byte m_buffer[sizeof(data_T)];
};
```

Each `cell_t` is meant to hold exactly one `data_T`. To do that safely, the raw byte buffer must share `data_T`’s
alignment, so the array is declared with `alignas(data_T)`. That way we can construct the object in place with placement
new and, later, treat `m_buffer` as a `data_T*` via `reinterpret_cast` without violating alignment rules.

### Enqueue

```c++
template<typename data_T, std::size_t size_T>
auto queue_single_threaded_t<data_T, size_T>::enqueue(data_T element) -> void {
    auto index = m_write_idx & m_mask;
    cell_t& cell = m_cells[index];
    const auto distance = m_write_idx - m_read_idx;
    if (distance == size_T) {
        auto* old_element = reinterpret_cast<data_T*>(&cell.m_buffer);
        old_element->~data_T();
        ++m_read_idx;
    }
    new (cell.m_buffer) data_T(std::move(element));
    ++m_write_idx;
}
```

To determine where the next element should be placed, we first compute the slot that corresponds to the current write
counter. In practice that means mapping the monotonically increasing `m_write_idx` into the ring—using
`m_write_idx % size_T`, or, for power-of-two capacities, `m_write_idx & (size_T - 1)`—and enqueuing at that computed
index.

```c++
auto index = m_write_idx & m_mask;
```

If `m_write_idx == 0b1101101` and `m_mask == 0b1111`, then the slot is `index = m_write_idx & m_mask = 0b1101`. In other
words, masking keeps only the low four bits (because the capacity is 16) and ignores all higher bits on the MSB side, so
the physical position is 13 in decimal while the counter itself keeps climbing. After choosing the slot, the queue
computes the “distance” as `m_write_idx - m_read_idx`. That difference is the live occupancy: it’s 0 when the queue is
empty, and it reaches `size_T` when the buffer is full, which is exactly when the overwrite policy kicks in and the
oldest element is evicted.

```c++
const auto distance = m_write_idx - m_read_idx;
```

If distance equals `size_T`, the queue is full. Because this is a bounded structure and we’ve chosen a forced-enqueue
policy where the newest item replaces the oldest, the enqueue path must take a special branch. In that branch we
identify the oldest slot at `m_read_idx & m_mask`, destroy the object currently stored there, advance `m_read_idx` by
one to free the position, and only then construct the incoming element in that cell. This keeps the “at most `size_T`
live objects” invariant intact and guarantees that enqueue never fails.

```c++
if (distance == size_T) {
    auto* old_element = reinterpret_cast<data_T*>(&cell.m_buffer);
    old_element->~data_T();
    ++m_read_idx;
}
```

The old element must be removed first. Because we know the cell’s byte buffer currently holds a properly constructed
`data_T` and is aligned for it, it’s safe to obtain a pointer with `reinterpret_cast<data_T*>(cell.m_buffer)` and then
call the object’s destructor explicitly. This explicit destruction is required because the value wasn’t created as a
normal automatic variable—it’s living in raw storage via placement new—so “going out of scope” won’t run its destructor.
After destroying the oldest element and constructing the new one in that slot, we advance `m_read_idx` by one so it now
points at the next oldest element currently held in the queue.

```c++
new (cell.m_buffer) data_T(std::move(element));
++m_write_idx;
```

This constructs a valid `data_T` object at the address stored in `cell.m_buffer` using placement new, so the bytes in
that cell become a live instance of `data_T`. For this path to work as written, `data_T` must provide a move
constructor, because the element is emplaced by moving the incoming value into that storage; without a move constructor
you’d need to adjust the enqueue path (for example, use copying) to match the type’s capabilities.

### Dequeue, or at least try dequeue

```c++
template<typename data_T, std::size_t size_T>
auto queue_single_threaded_t<data_T, size_T>::try_dequeue(data_T& element_out) -> bool {
    if (m_write_idx == m_read_idx) {
        return false;
    }
    const auto read_idx = m_read_idx & m_mask;
    auto* element = reinterpret_cast<data_T*>(m_cells[read_idx].m_buffer);
    element_out = std::move(*element);
    element->~data_T();
    ++m_read_idx;
    return true;
}
```

Dequeue is a bit simpler than enqueue because there’s no eviction path or special-case handling to worry about. The
operation begins by comparing the write and read indices; if `m_write_idx` equals `m_read_idx`, the queue is empty and
there’s nothing to remove, so the call returns immediately.

```c++
if (m_write_idx == m_read_idx) {
    return false;
}
```

If the queue is not empty, we can remove an element. As in the enqueue path, the first step is to compute the physical
slot for the current read position by mapping the monotonically increasing `m_read_idx` into the ring—either
`m_read_idx % size_T` or, for power-of-two capacities, `m_read_idx & (size_T - 1)`—and operate on that cell.

```c++
const auto read_idx = m_read_idx & m_mask;
```

From there, the `data_T` instance residing at the address held in `m_buffer` can be safely moved from. Because the cell
contains a properly constructed, correctly aligned `data_T`, taking a pointer to it and performing
`element_out = std::move(*ptr)` is valid; after the move we explicitly destroy the object in place and advance the read
index.

```c++
auto* element = reinterpret_cast<data_T*>(m_cells[read_idx].m_buffer);
element_out = std::move(*element);
element->~data_T();
```

This is one of the uncommon situations where using `reinterpret_cast` does not lead to undefined behavior. The object’s
lifetime began when it was constructed in place during enqueue (via placement new using the move constructor), and the
storage is `alignas(data_T)`, so converting the buffer address back to `data_T*` gives a pointer to a live, properly
aligned `data_T`. Because we’re removing the element, moving from it is legitimate; we extract the value, then
explicitly call the destructor to end that object’s lifetime in the queue’s storage—automatic scope won’t do it for us
here because the object lives in raw memory, not on the stack.

With the value destroyed, the last step is to advance `m_read_idx` to the next position so it points to the new oldest
element remaining in the queue.

```c++
++m_read_idx;
```

## Locking the queue

The current queue is meant for single-threaded use; it provides no synchronization mechanism and should not be shared
across threads as-is. For a class this small, adding a single `std::mutex` is enough to make it safe: we guard the
critical sections in `enqueue` and `try_dequeue`, leaving the public API and policy unchanged (forced enqueue,
`try_dequeue` may fail). Let’s look at the class structure with a mutex added.

```c++
template<typename data_T, std::size_t size_T>
class queue_locking_t {
public:
    static_assert(size_T > 0, "Doesn't make sense");
    static_assert(std::popcount(size_T) == 1, "Size of the queue must be power of 2");
    queue_locking_t() = default;
    ~queue_locking_t();
    auto enqueue(data_T element) -> void;
    auto try_dequeue(data_T& element_out) -> bool;
    auto try_discard() -> bool;
private:
    struct cell_t {
        alignas(data_T) std::byte m_buffer[sizeof(data_T)];
    };
    static constexpr std::size_t m_mask = size_T - 1;
    mutable std::mutex m_mutex;
    std::size_t m_write_idx = 0;
    std::size_t m_read_idx = 0;
    std::array<cell_t, size_T> m_cells;
};
```

The only substantive change from the single-threaded version is the addition of a `std::mutex` member (as declared in
the class). It’s a small tweak, but it’s enough to make the structure safe for concurrent use by a single producer and a
single consumer. The public interface and policy remain exactly the same—enqueue is forced and `try_dequeue` may
fail—while the mutex provides the necessary synchronization so producer and consumer never execute their critical
sections at the same time.

In the enqueue function, we acquire a `std::unique_lock` at the start and then follow the same steps as in the baseline:
compute the target slot from the masked write index, measure the distance between write and read indices, and if the
buffer is full, destroy the oldest element and advance `m_read_idx` to free space. We then placement-new the incoming
value into the cell and advance `m_write_idx`. The lock is released when the scope ends, which establishes the required
`happens-before` relationship for the consumer. This keeps the logic familiar while ensuring that updates to indices and
in-place object lifetimes are observed consistently across threads.

```c++
template<typename data_T, std::size_t size_T>
auto queue_locking_t<data_T, size_T>::enqueue(data_T element) -> void {
    std::unique_lock lock(m_mutex);
    auto index = m_write_idx & m_mask;
    cell_t& cell = m_cells[index];
    const auto distance = m_write_idx - m_read_idx;
    if (distance == size_T) {
        auto* old_element = reinterpret_cast<data_T*>(&cell.m_buffer);
        old_element->~data_T();
        ++m_read_idx;
    }
    new (cell.m_buffer) data_T(std::move(element));
    ++m_write_idx;
}
```

In `try_dequeue` the story mirrors `enqueue`. To avoid races on `m_write_idx` and `m_read_idx`, the lock must be taken
right at the start so the entire critical section is serialized between threads. The trade-off is that this approach
locks everything—both the state that truly needs synchronization and the parts that wouldn’t otherwise contend—so we pay
for mutual exclusion even when it isn’t strictly necessary. That extra serialization can waste cycles and cap how many
`enqueue`/`dequeue` operations fit into a given time slice. There is a more performant alternative that relaxes the
locking by switching to atomics and carefully chosen memory orders, but it’s correspondingly harder to implement
correctly and harder to reason about.

```c++
template<typename data_T, std::size_t size_T>
auto queue_locking_t<data_T, size_T>::try_dequeue(data_T& element_out) -> bool {
    std::unique_lock lock(m_mutex);
    if (m_write_idx == m_read_idx) {
        return false;
    }
    const auto read_idx = m_read_idx & m_mask;
    auto* element = reinterpret_cast<data_T*>(m_cells[read_idx].m_buffer);
    element_out = std::move(*element);
    element->~data_T();
    ++m_read_idx;
    return true;
}
```

## Less locking, almost lock-free

The “relaxed concurrency” version needs a few tweaks to the base class. We replace `m_write_idx` and `m_read_idx` from
`std::size_t` with `std::atomic_size_t` so producer and consumer can observe each other’s progress without a mutex, and
we introduce an `std::atomic_flag m_cell_lock` that protects the tiny overwrite window when enqueue runs on a full
queue. That flag gives us a safe, non-blocking way to remove the oldest element and place the new one in its slot,
preventing a race with the consumer that might be reading the same cell. This preserves the forced-enqueue policy while
keeping synchronization narrow and fast.

```c++
template<typename data_T, std::size_t size_T>
class queue_lock_free_t {
public:
    static_assert(size_T >= 2, "Queue size must be at least 2");
    static_assert(std::has_single_bit(size_T), "Size of the queue must be power of 2");
    queue_lock_free_t() = default;
    ~queue_lock_free_t();
    auto enqueue(data_T element) -> void;
    auto try_dequeue(data_T& element_out) -> bool;
    auto try_discard() -> bool;
private:
    struct cell_t {
        alignas(data_T) std::byte m_buffer[sizeof(data_T)];
    };
    static constexpr std::size_t m_mask = size_T - 1;
    std::atomic_size_t m_write_idx = 0;
    std::atomic_size_t m_read_idx = 0;
    std::atomic_flag m_cell_lock = ATOMIC_FLAG_INIT;
    std::array<cell_t, size_T> m_cells;
};
```

### Enqueue in almost lock-free

```c++
template<typename data_T, std::size_t size_T>
auto queue_lock_free_t<data_T, size_T>::enqueue(data_T element) -> void {
    // Get the current write position
    const auto write_idx = m_write_idx.load(std::memory_order_relaxed);
    auto read_idx = m_read_idx.load(std::memory_order_acquire);
    const auto index = write_idx & m_mask;
    if (write_idx - read_idx == size_T) {
        // Wait for free cell
        while (m_cell_lock.test_and_set(std::memory_order_acquire)) {}
        // Check if element to be overridden has not been already dequeued.
        if (m_read_idx.compare_exchange_weak(read_idx, read_idx + 1, std::memory_order_release, std::memory_order_relaxed)) {
            // It means that element has to be removed before new one will be emplaced
            auto* old_element = reinterpret_cast<data_T*>(m_cells[index].m_buffer);
            old_element->~data_T();
        }
        // Add new element
        new (m_cells[index].m_buffer) data_T(std::move(element));
        m_cell_lock.clear(std::memory_order_release);
    }
    else {
        // There is free space in buffer to add element
        new (m_cells[index].m_buffer) data_T(std::move(element));
    }
    // Advance write index
    m_write_idx.store(write_idx + 1, std::memory_order_release);
  }
```

At first glance the enqueue path in this variant looks noticeably more involved than in the earlier versions. That’s
expected, because it replaces the mutex with atomics and a tiny critical section guarded by a flag. Rather than treating
it as a black box, let’s unpack the logic step by step and see how each piece—index loads, fullness check, overwrite
window, element construction, and publication—fits together.

```c++
const auto write_idx = m_write_idx.load(std::memory_order_relaxed);
```

Because `m_write_idx` is written exclusively by the producer thread, the producer can read it with
`std::memory_order_relaxed`. That load isn’t used to synchronize with the consumer; it’s only for local index
arithmetic. The actual cross-thread visibility is established later when the producer publishes progress with a release
store to `m_write_idx`, which the consumer observes via an acquire load.

```c++
auto read_idx = m_read_idx.load(std::memory_order_acquire);
```

`m_read_idx` is touched by both sides—the consumer advances it after a successful try_dequeue, and the producer may
advance it during a forced overwrite when the buffer is full. Because of that, the producer should load `m_read_idx`
with acquire semantics so it reliably observes the consumer’s prior release update. Symmetrically, when the producer
does advance `m_read_idx` in the overwrite path, that update should be a release (e.g., the success side of a CAS or a
store) so the consumer’s subsequent acquire load sees the destruction of the old element before it tries to read from
that slot.

```c++
if (write_idx - read_idx == size_T)
```

This step performs an initial “is the buffer full?” check by comparing the producer’s view of `m_write_idx` and
`m_read_idx`. I call it potentially full because the state can change between those loads and any subsequent action:
the consumer may successfully call `try_dequeue` and advance `m_read_idx`, creating space, or the producer may race
ahead on another core. In other words, the comparison is a quick hint for the fast path, not a definitive verdict.
Before overwriting we must validate again—either after entering the tiny critical section guarded by `m_cell_lock` or by
using a CAS on `m_read_idx`—so we only destroy the oldest element if it’s still the one we observed.

```c++
while (m_cell_lock.test_and_set(std::memory_order_acquire)) {}
```

`m_cell_lock` is a tiny, purpose-built lock that makes overwriting the oldest element safe. Implemented as an
`std::atomic_flag`, it guards only the narrow critical section where the producer evicts the oldest entry when the
buffer is full, preventing a race with the consumer that might be reading the same cell. The producer acquires it with
`test_and_set` (acquire) and releases it with `clear` (release); the consumer simply checks the flag and, if it’s
already set, fails fast instead of blocking. This keeps synchronization tightly scoped while the rest of the queue
remains effectively lock-free.

```c++
// Check if element to be overridden has not been already dequeued.
if (m_read_idx.compare_exchange_weak(read_idx, read_idx + 1, std::memory_order_release, std::memory_order_relaxed)) {
    // It means that element has to be removed before new one will be emplaced
    auto* old_element = reinterpret_cast<data_T*>(m_cells[index].m_buffer);
    old_element->~data_T();
}
```

A successful `compare_exchange_weak` (or strong) on `m_read_idx` tells us that the value we previously loaded is still
current—no consumer has advanced it via `try_dequeue` in the meantime. By atomically moving `m_read_idx` from the
observed value to `read_idx + 1`, we effectively reserve that oldest slot for overwrite and establish that the consumer
is not reading it. In that case it’s safe to destroy the existing object in the cell and proceed to construct the new
element in place. If the CAS fails, it means the consumer already advanced `m_read_idx`, so the “oldest” we saw is no
longer valid; we must re-read the index and re-evaluate before attempting any removal.

```c++
// Add new element
new (m_cells[index].m_buffer) data_T(std::move(element));
m_cell_lock.clear(std::memory_order_release);
```

Once the oldest element has been removed, the producer constructs the new value directly in that same cell using
placement new. After the object is in place, the producer clears the atomic flag to end the tiny overwrite window and
then publishes progress by performing a release store to `m_write_idx`. That sequence both frees the consumer to proceed
and ensures the consumer’s acquire load of `m_write_idx` observes a fully constructed element before it reads from the
slot.

If the queue wasn’t full in the first place, the path is simpler: the producer places the new element into the computed
slot without taking the flag, and then performs the same release store to advance `m_write_idx`. The consumer’s acquire
load pairs with that release so the fresh value is visible as soon as the indices show it’s available.

```c++
else {
    // There is free space in buffer to add element
    new (m_cells[index].m_buffer) data_T(std::move(element));
}
```

Finally, the producer advances the write index. In the single-threaded and mutex-guarded versions this is just
`++m_write_idx`, while in the lock-free variant we publish progress with a
`store(write_idx + 1, std::memory_order_release)` so the consumer’s acquire load sees the fully constructed element. The
counter itself grows monotonically; wrap-around is handled later by mapping it into the ring with `& m_mask`, so there’s
no need to reset it to zero.

```c++
// Advance write index
m_write_idx.store(write_idx + 1, std::memory_order_release);
```

### Dequeue in almost lock-free

```c++
template<typename data_T, std::size_t size_T>
auto queue_lock_free_t<data_T, size_T>::try_dequeue(data_T& element_out) -> bool {
    auto read_idx = m_read_idx.load(std::memory_order_acquire);
    const auto write_idx = m_write_idx.load(std::memory_order_acquire);
    if (read_idx == write_idx) {
        return false;
    }
    // Following is required for synchronization in forced enqueue
    if (m_cell_lock.test_and_set(std::memory_order_acquire)) {
        return false;
    }
    read_idx = m_read_idx.load(std::memory_order_acquire);
    const auto index = read_idx & m_mask;
    auto* element = reinterpret_cast<data_T*>(m_cells[index].m_buffer);
    element_out = std::move(*element);
    element->~data_T();
    m_read_idx.store(read_idx + 1, std::memory_order_release);
    m_cell_lock.clear(std::memory_order_release);
    return true;
}
```

This path is simpler, but it still introduces a couple of new moving parts. Using acquire semantics when loading
`m_read_idx` and `m_write_idx` guarantees we observe any prior release updates from the other thread, so the indices we
read are up to date and represent a valid, visible state before we proceed.

```c++
auto read_idx = m_read_idx.load(std::memory_order_acquire);
const auto write_idx = m_write_idx.load(std::memory_order_acquire);
```

Next the consumer checks the atomic flag `m_cell_lock`. If the flag is set, it means the producer currently owns the
tiny overwrite window and is in the middle of replacing the oldest element. Because `try_dequeue` always targets that
oldest element, proceeding would race the producer on the very same slot, so the consumer immediately returns false.
This “fail fast” behavior is intentional: it keeps the operation non-blocking and preserves correctness while the
overwrite is in progress.

```c++
if (m_cell_lock.test_and_set(std::memory_order_acquire)) {
  return false;
}
```

Once `try_dequeue` successfully acquires the atomic flag, it owns the small critical window and it’s safe to remove the
oldest element. However, because `m_read_idx` may have been advanced in the meantime (for example, by the producer
during a forced overwrite), the consumer should reload `m_read_idx`—preferably with acquire semantics—before computing
the slot. With the fresh index, it can map to the cell, move the value out, destroy it in place, advance `m_read_idx`
with a release update, and finally clear the flag.

```c++
read_idx = m_read_idx.load(std::memory_order_acquire);
```

The remaining steps mirror the baseline single-threaded flow: compute the slot for the current read position, move the
value out, destroy the object in place, and advance `m_read_idx` to point at the next oldest element. The only addition
in this concurrent variant is the tail of the operation, where we clear `m_cell_lock` to close the tiny critical
section. Clearing the flag (with release semantics) signals that the overwrite/dequeue window is free again, allowing
the producer to proceed and ensuring both threads observe a consistent view of the queue’s state.

```c++
m_cell_lock.clear(std::memory_order_release);
```

This wraps up a brief comparison of several single-producer, single-consumer queue designs, all anchored in the
single-threaded baseline and then extended into the lock-based and mostly lock-free implementations. In the final
section I’ll present benchmark results so the trade-offs aren’t just theoretical; the numbers will make the differences
concrete and allow a clear, side-by-side comparison of how each approach behaves under the same workload.

## Benchmark

*Note: These benchmark results are illustrative and may contain measurement artifacts. Lock-free implementations cannot outperform single-threaded baselines due to synchronization overhead. Actual performance depends heavily on hardware, compiler optimizations, and workload characteristics.*

| Variant       | Enqueue/Dequeue ratio | `enqueue` per second | `try_dequeue` per second |
| ------------- | --------------------- | -------------------- | ------------------------ |
| Single Thread | enqueue only          | 10.27 G/s            | N/A                      |
| Lock-based    | enqueue only          | 298.35 M/s           | N/A                      |
| Lock-free     | enqueue only          | 106.23 M/s           | N/A                      |
| Single Thread | 9 enqueue / 1 dequeue | 2.65 G/s             | 294.84 M/s               |
| Lock-based    | 9 enqueue / 1 dequeue | 90.89 M/s            | 10.10 M/s                |
| Lock-free     | 9 enqueue / 1 dequeue | 106.60 M/s           | 11.84 M/s                |
| Single Thread | 3 enqueue / 1 dequeue | 2.57 G/s             | 857.86 M/s               |
| Lock-based    | 3 enqueue / 1 dequeue | 76.34 M/s            | 25.44 M/s                |
| Lock-free     | 3 enqueue / 1 dequeue | 105.51 M/s           | 35.17 M/s                |
| Single Thread | 1 enqueue / 1 dequeue | 1.63 G/s             | 1.63 G/s                 |
| Lock-based    | 1 enqueue / 1 dequeue | 51.38 M/s            | 51.38 M/s                |
| Lock-free     | 1 enqueue / 1 dequeue | 104.24 M/s           | 104.24 M/s               |
| Single Thread | 1 enqueue / 3 dequeue | 554.33 M/s           | 1.66 G/s                 |
| Lock-based    | 1 enqueue / 3 dequeue | 25.96 M/s            | 77.89 M/s                |
| Lock-free     | 1 enqueue / 3 dequeue | 93.96 M/s            | 281.87 M/s               |
| Single Thread | 1 enqueue / 9 dequeue | 280.38 M/s           | 2.52 G/s                 |
| Lock-based    | 1 enqueue / 9 dequeue | 10.45 M/s            | 94.05 M/s                |
| Lock-free     | 1 enqueue / 9 dequeue | 85.30 M/s            | 767.74 M/s               |
| Single Thread | dequeue only          | N/A                  | 7.97 G/s                 |
| Lock-based    | dequeue only          | N/A                  | 104.82 M/s               |
| Lock-free     | dequeue only          | N/A                  | 5.12 G/s                 |
