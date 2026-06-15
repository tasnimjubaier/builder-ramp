# Stage 04 — Exchange Mechanics & Market Microstructure

**Status:** not started
**Estimated time:** 3–4 days
**Type:** additive
**Depends on:** Stage 03 (order book, matching engine, feed simulator)

The JD lists "basic understanding of trading concepts, including exchange engine operations, order book mechanics, and transaction processing" under Nice to Have — but in interviews, the quants and PMs in the room will probe this hard. This stage gives you working mental models, not just vocabulary. You'll formalize the event model, build a simulated exchange gateway, and replay real or synthetic market data through your engine.

---

## What You're Building

- A formal event model for the full order lifecycle
- A simulated exchange gateway that accepts orders and returns fills/rejects
- A market data feed parser (synthetic tick data in CSV or binary format)
- A book rebuilder that reconstructs L2 from a snapshot + delta stream

---

## Concepts This Stage Teaches

```yaml
concepts:
  Order lifecycle:
    what: >
      New → Acknowledged → Partially Filled → Filled (terminal)
                         → Cancelled (terminal)
                         → Rejected (terminal)
      Every state transition produces an execution report back to the sender.
    why: >
      Your strategy needs to track this lifecycle. An order you sent may not be
      in the book yet. An order you think is live may have been partially filled.
      Ignoring lifecycle state causes ghost orders and incorrect position tracking.

  Order types in depth:
    what: >
      Limit: execute at price P or better. Goes into the book if not immediately matchable.
      Market: execute immediately at the best available price. No price guarantee.
      IOC (Immediate-or-Cancel): fill what you can immediately; cancel the rest.
      FOK (Fill-or-Kill): fill the entire quantity immediately or cancel entirely.
      Post-only: go into the book as a maker. If it would immediately match, reject it.
    why: >
      These order types are the vocabulary of strategy. A market-maker uses Post-only
      to avoid paying taker fees. An aggressive strategy uses IOC for immediacy.
      Not knowing these types in an HFT interview is a red flag.

  Exchange gateway and FIX protocol:
    what: >
      The exchange gateway is the entry point for orders. It validates, acknowledges,
      and routes to the matching engine. FIX (Financial Information eXchange) is the
      standard protocol: tag=value pairs, e.g. 35=D (new order), 39=2 (filled).
    why: >
      The JD mentions "transaction processing." This is it. You don't need to implement
      FIX fully — but you should know what it is, why it exists, and what tags 35, 39, 54
      mean off the top of your head.
    key_tags: >
      35=D   : New Order Single
      35=8   : Execution Report
      39=0   : Order status New
      39=1   : Partial fill
      39=2   : Full fill
      39=4   : Cancelled
      39=8   : Rejected
      54=1   : Buy side
      54=2   : Sell side

  Market data feed mechanics:
    what: >
      Exchanges publish two data streams:
        - Order feed (L3): every individual add/cancel/modify event
        - Quote feed (L2): aggregated price levels, updated on each change
      Feed delivery: UDP multicast (low latency, no delivery guarantee),
      or TCP (reliable, higher latency).
    why: >
      Your backtester (Stage 06) replays this data. Understanding the feed format
      and reconstruction logic is a prerequisite.

  Book rebuild from snapshot + delta:
    what: >
      On startup, you receive a full L2 snapshot (all price levels and quantities).
      Then delta messages arrive: price_level P changed to quantity Q, or was removed.
      You apply deltas to the snapshot to maintain a current L2 view.
    why: >
      This is how every live system initializes its book view. Implementing it
      forces you to handle the sequencing carefully — a delta that arrives before
      its snapshot must be queued and replayed.

  Adverse selection and queue position:
    what: >
      Adverse selection: you get filled when the market is moving against you.
      A buyer at 100 who fills just before the price drops to 99 was adversely selected.
      Queue position: earlier orders at a price level fill before later ones.
      Being first in queue at a price is a significant competitive edge in HFT.
    why: >
      This is why HFT firms care about latency at the microsecond level. A 10μs
      faster order can be 1000 positions higher in the queue.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Define the formal event model"
    detail: >
      In src/events/, define event types using std::variant:

        struct OrderAck    { OrderId id; Timestamp ts; };
        struct OrderFill   { OrderId maker_id; OrderId taker_id; Price price; Quantity qty; Timestamp ts; };
        struct OrderCancel { OrderId id; Timestamp ts; };
        struct OrderReject { OrderId id; std::string reason; Timestamp ts; };

        using OrderEvent = std::variant<OrderAck, OrderFill, OrderCancel, OrderReject>;

      Add a Timestamp type (int64_t nanoseconds since epoch) to types/primitives.hpp.
      This event model feeds into Stage 05 (strategy callbacks) and Stage 06 (backtest loop).

  2:
    task: "Implement a simulated exchange gateway"
    detail: >
      In src/exchange/, write ExchangeGateway:
        - submit(Order order) → queues into the order book via ring buffer
        - cancel(OrderId id) → queues a cancel into the ring buffer
        - process_pending() → drains the ring buffer, runs matching, emits events

      ExchangeGateway owns the OrderBook and the event output queue.
      The strategy (Stage 05) submits orders to the gateway and receives events back.

  3:
    task: "Implement all order types in the matching engine"
    detail: >
      Extend the Stage 03 matching engine to handle:
        - IOC: after match(), cancel remaining quantity
        - FOK: check total available quantity before matching; reject if insufficient
        - Post-only: if the order would immediately match, reject with reason "would_match"
        - Market: match against the best available levels until filled or book is empty

      Write a test for each order type. Verify IOC leaves no resting quantity.
      Verify FOK either fills completely or leaves the book unchanged.

  4:
    task: "Build a tick data format and reader"
    detail: >
      Define a simple tick data format (CSV is fine for now):
        timestamp_ns,event_type,order_id,side,price,quantity
        1719000000000000000,ADD,1001,B,100.25,500
        1719000000000100000,ADD,1002,S,100.75,300
        1719000000000200000,CANCEL,1001,B,100.25,0

      Write TickReader in src/feed/:
        - Opens a CSV file, parses line by line
        - Emits OrderEvent structs in timestamp order
        - Handles ADD, CANCEL, MODIFY, and TRADE event types

      Generate 10,000 synthetic tick events to a CSV. Replay them through your engine.

  5:
    task: "Implement L2 book rebuild from snapshot + delta"
    detail: >
      Simulate the snapshot/delta problem:
        - Emit a full L2 snapshot at startup (all current price levels)
        - Then stream delta updates: {price, new_total_qty} — 0 means remove level
        - BookRebuilder applies deltas to maintain a current L2 view

      Test: apply 1000 deltas, then compare the L2 view to a fresh L3 book
      that processed the same events. They should match exactly.

  6:
    task: "Order lifecycle state machine"
    detail: >
      Add lifecycle tracking to the ExchangeGateway:
        - std::unordered_map<OrderId, OrderStatus> active_orders_
        - Status transitions: New → Acked → PartialFill → Filled/Cancelled/Rejected
        - Query: gateway.order_status(OrderId) → OrderStatus

      This is what a strategy uses to know if its order is still live.
      Test: submit an order, fill it partially, cancel the remainder.
      Verify status transitions are correct at each step.
```

---

## Done When

```yaml
done_when:
  - All four order types (limit, market, IOC, FOK) are implemented and tested
  - Post-only rejection works correctly
  - Event model (Ack/Fill/Cancel/Reject) is formalized and emitted correctly
  - Tick data replays through the engine without errors
  - L2 book rebuild from snapshot+delta matches L3 ground truth
  - You can explain the FIX protocol tags 35=D and 35=8 in a sentence each
  - You can explain adverse selection with a concrete example
```

---

## Interview Signal

```yaml
what_this_demonstrates:
  - You know the full order lifecycle — not just "submit order, get fill"
  - You can implement IOC/FOK correctly — a common HFT interview question
  - You understand market data feed mechanics — directly required by the JD
  - You know what FIX is and can discuss it at a high level
```

---

## Open Questions

```yaml
open_questions:
  - What is co-location and why do HFT firms pay for it?
  - What is a dark pool? How does its matching differ from a lit exchange?
  - What is NBBO (National Best Bid and Offer)? Relevant for US equities.
  - What is latency arbitrage? How does queue position relate to it?
  - What are the main differences between futures, equities, and crypto market structure?
    (Dubai + Amsterdam firm likely trades multiple asset classes)
```
