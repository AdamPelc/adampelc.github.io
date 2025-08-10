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
