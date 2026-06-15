# Stage 02 — Systems & Performance

**Status:** not started
**Estimated time:** 4–5 days
**Type:** additive
**Depends on:** Stage 01 (CMake project, Timer, primitives)

HFT is not about writing correct code. Any firm can do that. It's about writing correct code that runs at nanosecond latency and stays there under load. This stage teaches you the hardware model that makes low-latency systems possible — and the C++ techniques that exploit it. Without this, you can implement an order book but you can't make it fast. With this, you understand why every design decision in the codebase is made the way it is.

---

## What You're Building

- A benchmark harness using your Stage 01 Timer
- A cache-friendly struct layout for the order book (pre-built for Stage 03)
- A lock-free single-producer single-consumer (SPSC) ring buffer
- A set of microbenchmarks that produce measurable, explainable numbers

---

## Concepts This Stage Teaches

```yaml
concepts:
  CPU cache hierarchy:
    what: >
      L1 cache: ~4 cycles, ~32KB per core.
      L2 cache: ~12 cycles, ~256KB per core.
      L3 cache: ~40 cycles, ~8MB shared.
      RAM: ~200 cycles.
      Numbers vary by processor but the ratios are stable.
    why: >
      An HFT system lives or dies on cache hit rate. An order book lookup that
      hits L1 takes 4ns. The same lookup hitting RAM takes 200ns. That's 50x.
      At 100,000 messages per second, 50x latency means you miss every opportunity.
    exercise: >
      Write two loops over a large array: one accessing elements in order (sequential),
      one accessing randomly (random). Measure with your Timer. The difference is the cache.

  Cache line and false sharing:
    what: >
      A cache line is 64 bytes — the smallest unit the CPU loads from RAM.
      False sharing: two threads write to different variables that share a cache line.
      Each write invalidates the other thread's cache copy. Both stall.
    why: >
      In multi-threaded trading systems, the order book is read by one thread and
      updated by another. If their counters share a cache line, you've introduced
      invisible, hard-to-diagnose latency. The fix is alignment padding.
    pattern: >
      struct alignas(64) Counter {
        std::atomic<int64_t> value{0};
        char padding[56];  // pad to 64 bytes — one full cache line
      };

  Memory layout and struct packing:
    what: >
      The compiler adds padding between struct members to align them to their natural
      size. A char followed by an int64_t wastes 7 bytes. Reordering members eliminates
      the padding and makes the struct smaller — more fit in a cache line.
    why: >
      The order book stores millions of price levels. A Level struct that's 64 bytes
      holds 1 per cache line. A Level struct that's 32 bytes holds 2. Everything else
      being equal, the 32-byte struct is faster.
    tool: >
      static_assert(sizeof(Level) == 32, "Level must be 32 bytes");
      Use this throughout to catch accidental padding.

  Branch prediction:
    what: >
      Modern CPUs predict the outcome of if-statements and speculatively execute
      the predicted path. A mispredicted branch flushes the pipeline: ~15 cycle penalty.
    why: >
      Hot paths in order matching execute thousands of branches per second.
      [[likely]] and [[unlikely]] hints in C++20 guide the predictor.
    pattern: >
      if ([[likely]] order.type == OrderType::Limit) { ... }
      if ([[unlikely]] position > risk_limit) { trigger_kill_switch(); }

  std::atomic and memory ordering:
    what: >
      std::atomic<T> ensures reads and writes to T are indivisible. Memory ordering
      controls visibility across threads: relaxed, acquire, release, seq_cst.
    why: >
      Lock-free data structures use atomics instead of mutexes. A mutex is ~100ns.
      An atomic load with acquire ordering is ~5ns. On a hot path, that's the difference.
    ordering_guide: >
      seq_cst:  safe default, but slowest — use for correctness first
      acquire:  use on load side of a producer/consumer handoff
      release:  use on store side of a producer/consumer handoff
      relaxed:  use for counters where ordering doesn't matter (stats, telemetry)

  Lock-free SPSC ring buffer:
    what: >
      A circular buffer with one writer thread and one reader thread.
      The writer advances a write cursor; the reader advances a read cursor.
      No mutex required — just atomic loads and stores with acquire/release ordering.
    why: >
      The canonical HFT data structure for passing market data between threads.
      Feed handler writes. Strategy reads. Zero contention, zero blocking, predictable latency.

  Memory allocator behavior:
    what: >
      new/delete call the OS allocator (malloc/free). These are non-deterministic:
      they can take microseconds in the worst case. HFT systems pre-allocate everything
      they need at startup and never allocate on the hot path.
    why: >
      A 10 microsecond allocation delay in the middle of a strategy decision is an eternity.
      Object pools, arena allocators, and pre-allocated arrays replace dynamic allocation
      on hot paths.
    exercise: >
      Measure: how long does new int[1000000] take? How long does accessing
      a pre-allocated array of the same size take? Your Timer gives you the answer.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Build a benchmark harness"
    detail: >
      In utils/, write a Benchmark helper:
        - Takes a name and a callable (std::function<void()>)
        - Runs it N times, records min/max/avg nanoseconds using Timer
        - Prints a formatted result line

      template<typename Fn>
      void benchmark(const std::string& name, int iterations, Fn fn) {
        Timer t;
        int64_t total = 0;
        int64_t min_ns = INT64_MAX, max_ns = 0;
        for (int i = 0; i < iterations; ++i) {
          t.reset();
          fn();
          int64_t elapsed = t.elapsed_ns();
          total += elapsed;
          min_ns = std::min(min_ns, elapsed);
          max_ns = std::max(max_ns, elapsed);
        }
        printf("%-30s  avg=%5ldns  min=%5ldns  max=%5ldns\n",
               name.c_str(), total/iterations, min_ns, max_ns);
      }

  2:
    task: "Cache line experiment"
    detail: >
      Allocate a std::vector<int> of 100 million elements.
      Benchmark A: iterate sequentially, sum all elements.
      Benchmark B: iterate with stride 16 (every 16th element), sum.
      Benchmark C: iterate with stride 64.

      Print all three. Explain the numbers using the cache hierarchy.
      The gap between A and C is your intuition for why data layout matters.

  3:
    task: "False sharing demonstration"
    detail: >
      Create two structs:
        struct Shared   { int64_t a; int64_t b; };  // a and b share a cache line
        struct Padded   { int64_t a; char pad[56]; int64_t b; };  // separate lines

      Spawn two threads: one increments .a in a loop, one increments .b.
      Benchmark both structs. Shared will be slower. Padded will be faster.
      Print the ratio. This is false sharing made tangible.

  4:
    task: "Struct layout optimization"
    detail: >
      Design the Order struct you'll use in the order book (Stage 03):
        struct Order {
          OrderId  id;          // 8 bytes
          Price    price;       // 8 bytes
          Quantity quantity;    // 8 bytes
          Side     side;        // 1 byte — enum class
          OrderType type;       // 1 byte
          // padding here? how many bytes?
        };

      Add static_assert(sizeof(Order) == X) where X is the size you calculated.
      Reorder the fields to minimize padding. Use offsetof() to verify positions.
      Goal: fit 2 orders in a single 64-byte cache line if possible.

  5:
    task: "Implement SPSC ring buffer"
    detail: >
      In core/, write RingBuffer<T, N>:
        - Fixed capacity N (power of 2 for fast modulo via bitmask)
        - push(const T&): returns bool — false if full
        - pop(T&): returns bool — false if empty
        - write_cursor_: atomic<size_t>, read by consumer, written by producer
        - read_cursor_:  atomic<size_t>, read by producer, written by consumer

      Alignment: write_cursor_ and read_cursor_ must each be on their own cache line.
      Memory ordering: producer stores with release, consumer loads with acquire.

      Test it: producer thread pushes 1 million integers, consumer reads them.
      Verify all arrived, no data race (run with -fsanitize=thread).

  6:
    task: "Pre-allocation pattern"
    detail: >
      In core/, write ObjectPool<T>:
        - Constructor: pre-allocates N objects with placement new into a raw buffer
        - acquire(): returns T* from the pool — O(1), no heap allocation
        - release(T*): returns T* to the pool
        - Pool is not thread-safe — single thread only (the order book thread)

      Benchmark: allocate/free 1 million objects via new/delete vs via ObjectPool.
      The pool should be 10–100x faster.

  7:
    task: "Compile and measure everything"
    detail: >
      From main.cpp, run all benchmarks in order and print results.
      Compile with -O2 -march=native. Rerun with -O0. Compare.
      Note what the optimizer does to your naive benchmark loop
      (hint: it might optimize it away entirely unless you use the result).

      Use volatile or std::atomic<> sinks to prevent dead-code elimination:
        volatile int64_t sink = 0;
        sink += result;   // compiler can't prove this is unused
```

---

## Done When

```yaml
done_when:
  - You can explain L1/L2/L3/RAM latency numbers from memory
  - You have measured false sharing and can explain why it happens
  - Your Order struct is cache-line-optimized with static_assert in place
  - SPSC ring buffer passes concurrent correctness test with -fsanitize=thread
  - ObjectPool benchmarks at least 10x faster than new/delete for 1M allocations
  - You can explain acquire/release memory ordering in one paragraph
```

---

## Interview Signal

```yaml
what_this_demonstrates:
  - You know the hardware model, not just the language
  - You can design data structures around cache behavior — an explicit JD requirement
  - You have measured latency, not estimated it — engineers at HFT firms trust numbers
  - You can build a lock-free structure correctly — a senior C++ expectation
```

---

## Open Questions

```yaml
open_questions:
  - What is NUMA (Non-Uniform Memory Access)? How does it affect multi-socket servers?
  - What is CPU affinity (pthread_setaffinity_np)? Why do HFT systems pin threads to cores?
  - What is kernel bypass networking (DPDK, OpenOnload)? Why does it matter for latency?
  - What is busy-waiting vs blocking I/O? What does an HFT feed handler actually do?
  - What compiler flags does HFT typically use beyond -O2? (-O3, -funroll-loops, PGO)
```
