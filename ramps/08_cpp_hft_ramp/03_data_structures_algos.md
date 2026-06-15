# Stage 03 — Data Structures & Algorithms for Trading

**Status:** not started
**Estimated time:** 5–7 days
**Type:** additive
**Depends on:** Stage 01 (primitives, Order struct), Stage 02 (cache layout, ObjectPool)

This stage is the interview gate. Any HFT coding screen will test order book implementation, priority queue mechanics, and algorithmic complexity reasoning. But it goes further than a LeetCode problem — you're building the real data structure, cache-optimized, using the types from Stage 01 and the layout principles from Stage 02. When you finish, you have a production-quality order book that you can explain line by line.

---

## What You're Building

- A price-time priority limit order book for a single instrument
- L2 book reconstruction (aggregated by price level)
- A ring buffer market data feed simulator
- Benchmark proving sub-microsecond per-operation latency on the hot path

---

## Concepts This Stage Teaches

```yaml
concepts:
  Order book structure:
    what: >
      An order book has two sides: bids (buy orders, sorted descending by price)
      and asks (sell orders, sorted ascending by price).
      Each price level holds a queue of orders sorted by arrival time (price-time priority).
      The best bid is the highest bid price. The best ask is the lowest ask price.
      The spread is best_ask - best_bid.
    data_structure: >
      std::map<Price, std::deque<Order>> bids;  // std::greater<Price> — descending
      std::map<Price, std::deque<Order>> asks;  // std::less<Price>    — ascending

  Price-time priority matching:
    what: >
      When a new order arrives:
        - Limit order: placed in the book at its price level, at the back of that level's queue
        - Market order: matched against the best available price immediately
        - Fill: when a buy price >= best ask, or a sell price <= best bid, a match occurs
      Within a price level, the oldest order (front of the queue) matches first.
    implementation: >
      match(Order& incoming):
        while incoming.quantity > 0 and book has opposing orders:
          best_level = bids.begin() or asks.begin()
          resting = best_level->second.front()
          fill_qty = min(incoming.quantity, resting.quantity)
          emit Fill(incoming.id, resting.id, fill_qty, best_level->first)
          reduce quantities; remove fully-filled orders; remove empty price levels

  L2 book (market depth):
    what: >
      L2 data aggregates all orders at each price into a single quantity.
      You see: price=100.25 qty=500, price=100.00 qty=1200 — not the individual orders.
      This is what exchanges publish on their market data feed.
    why: >
      You need to be able to reconstruct L2 from L3 (individual orders) and
      apply L2 snapshots/deltas from a feed. Both directions matter in backtesting.

  std::map vs sorted array for price levels:
    what: >
      std::map gives O(log N) insert/lookup/delete. For small N (most books have
      < 100 active price levels), the constant factor matters more than the asymptotic.
      A flat sorted array with binary search can be faster due to cache locality.
    why: >
      This is the first design tradeoff you'll be asked about in a system design interview.
      Know both options, know when to switch, and have a benchmark ready.

  Priority queue internals:
    what: >
      std::priority_queue is a max-heap by default. For the best bid (max price)
      you use the default. For the best ask (min price), use std::greater<Price>.
      A heap gives O(log N) push and O(1) top(). But it does NOT support fast cancel.
    why: >
      Cancel requires finding an arbitrary order. In a heap, that's O(N).
      The real order book uses a combination: map for price levels (O(log N) cancel
      at level granularity) + deque for time-ordering within a level.

  Order ID index:
    what: >
      A separate hash map from OrderId to the order's location in the book.
      Needed for O(1) cancel: look up the order, remove it from its price level's deque.
    why: >
      Without the index, cancel is O(N * M) — scan every price level, scan every order.
      With std::unordered_map<OrderId, iterator>, cancel is O(1) amortized.

  Ring buffer feed simulation:
    what: >
      Use the SPSC ring buffer from Stage 02 to simulate a market data feed.
      A producer thread generates random order events (new/cancel/modify).
      The consumer thread is the order book, processing events in sequence.
    why: >
      This is the actual architecture. Feed handler → ring buffer → order book engine.
      Building it now means Stage 04 (exchange mechanics) slots in naturally.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Define the OrderBook class skeleton"
    detail: >
      In src/orderbook/, create OrderBook.hpp:

        class OrderBook {
        public:
          void add_order(Order order);
          void cancel_order(OrderId id);
          void modify_order(OrderId id, Quantity new_qty);

          Price best_bid() const;
          Price best_ask() const;
          int64_t spread() const;

          std::vector<std::pair<Price, Quantity>> bid_levels(int depth) const;
          std::vector<std::pair<Price, Quantity>> ask_levels(int depth) const;

        private:
          std::map<Price, std::deque<Order>, std::greater<Price>> bids_;
          std::map<Price, std::deque<Order>, std::less<Price>>    asks_;
          std::unordered_map<uint64_t, /* iterator */> order_index_;
        };

      Implement all methods. No matching yet — that's step 3.

  2:
    task: "Test add and cancel"
    detail: >
      Write a test in src/tests/ (use a simple main, not a framework yet):
        - Add 5 buy orders at 3 different price levels
        - Add 5 sell orders at 3 different price levels
        - Verify best_bid() and best_ask() are correct
        - Cancel the best bid — verify best_bid() updates
        - Cancel a non-existent order — verify it doesn't crash

      This is TDD for the order book. It's also the form a take-home test might take.

  3:
    task: "Implement matching engine"
    detail: >
      Add a private match() method called after every add_order().
      Matching rules:
        - A buy order at price P matches against asks where ask_price <= P
        - A sell order at price P matches against bids where bid_price >= P
        - Fill quantity = min(incoming.quantity, resting.quantity)
        - Emit a Fill event (just print for now — Stage 04 formalizes events)
        - Reduce quantities; remove fully-filled orders; remove empty levels

      Write test: add a buy at 100.50. Add a sell at 100.00. They should match.
      Verify Fill is emitted. Verify both orders are removed from the book.

  4:
    task: "Add L2 snapshot output"
    detail: >
      Implement:
        struct L2Level { Price price; Quantity total_qty; int order_count; };

        std::vector<L2Level> get_l2_bids(int depth) const;
        std::vector<L2Level> get_l2_asks(int depth) const;

      Print a formatted order book snapshot:
        === BID ===             === ASK ===
        100.50   500 (3)        100.75   200 (1)
        100.25  1200 (5)        101.00   800 (4)
        100.00   300 (2)        101.50   450 (3)

      This is the output format used in debugging and backtesting reports.

  5:
    task: "Feed simulator with ring buffer"
    detail: >
      In src/feed/, write FeedSimulator:
        - Generates random Order events: 60% adds, 30% cancels, 10% modifies
        - Prices distributed around a midpoint with gaussian-ish spread
        - Pushes events into the Stage 02 RingBuffer
        - Runs in its own thread

      OrderBook now reads from the ring buffer in a loop.
      Run for 1 million events. Verify book state is consistent at the end.

  6:
    task: "Benchmark the hot path"
    detail: >
      Measure:
        - add_order() for a non-matching limit order: target < 500ns
        - add_order() that triggers a match: target < 1000ns
        - cancel_order() via order index: target < 200ns
        - best_bid() / best_ask(): target < 50ns

      Use your Benchmark harness from Stage 02. Run 1 million iterations each.
      If you're above these targets, profile (perf, valgrind --tool=callgrind)
      and find the bottleneck. The most common culprit: map node allocations.

  7:
    task: "Flat array variant (optional but interview-gold)"
    detail: >
      Implement a second order book variant: FlatOrderBook
        - Price levels stored in a sorted std::vector instead of std::map
        - Binary search for price level lookup
        - Compare its benchmark numbers to the std::map version

      The fact that you benchmarked both and know the tradeoffs is an immediate
      signal in a system design interview.
```

---

## Done When

```yaml
done_when:
  - Order book handles add, cancel, modify, and matching correctly
  - Tests cover: normal adds, partial fills, full fills, cancel, modify, multiple levels
  - L2 snapshot prints correctly
  - Feed simulator runs 1M events without crash or inconsistency
  - add_order() (non-matching) benchmarks < 500ns per operation
  - You can explain price-time priority matching without looking at your code
  - You can explain the cancel O(1) trick (order index) in an interview
```

---

## Interview Signal

```yaml
what_this_demonstrates:
  - You can implement an order book from scratch — the canonical HFT interview problem
  - You know the real data structures (map + deque + hash index), not just "use a heap"
  - You have benchmarks with numbers, not estimates
  - You understand L2 vs L3 data — required domain knowledge per the JD
```

---

## Open Questions

```yaml
open_questions:
  - How do real exchanges handle order amendment — new OrderId or same?
  - What is iceberg/hidden quantity? How does it affect matching?
  - What is a stop order? A stop-limit order? (Expanding your order type vocabulary)
  - What is maker/taker fee structure and how does it affect strategy?
  - How do you reconstruct a full order book from a snapshot + delta feed?
    (This is Stage 04 — note it here)
```
