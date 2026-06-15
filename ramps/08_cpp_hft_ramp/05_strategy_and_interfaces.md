# Stage 05 — Strategy Abstractions & Quant-Facing Interfaces

**Status:** not started
**Estimated time:** 4–5 days
**Type:** additive
**Depends on:** Stage 04 (exchange gateway, event model, order lifecycle)

The JD has two distinct asks: implement HFT strategies, and design "user-friendly, expressive and efficient trading interfaces for quick development and deployment of complex trading ideas by quant researchers." These are different skills. One is implementation. The other is API design — and it's specifically aimed at researchers who may not be C++ experts. This stage builds both: a clean strategy abstraction layer, and two working strategy implementations on top of it.

---

## What You're Building

- A `Strategy` base class with callback-based event handling
- A `StrategyContext` that gives strategies access to the book and gateway without exposing internals
- Two concrete strategies: a naive market-maker and a momentum strategy
- A multi-strategy runner that manages lifecycle and routes events

---

## Concepts This Stage Teaches

```yaml
concepts:
  Strategy as an event-driven state machine:
    what: >
      A strategy doesn't poll for data. It reacts to events:
        on_book_update(L2Snapshot)  — market state changed
        on_fill(OrderFill)          — one of my orders was filled
        on_ack(OrderAck)            — my order is in the book
        on_cancel(OrderCancel)      — my order was cancelled
        on_reject(OrderReject)      — my order was rejected
      The strategy maintains internal state and decides whether to act on each event.
    why: >
      This is the architecture of every real trading system. Polling introduces latency.
      Callbacks minimize the path from "market event" to "order decision."

  Strategy interface design for quant researchers:
    what: >
      The JD explicitly asks for interfaces that let quant researchers develop and
      deploy ideas quickly. That means: minimal boilerplate, clear naming, typed
      inputs that can't be misused, and sensible defaults.
      A quant should be able to implement a new strategy by overriding 2–3 methods,
      not by understanding memory layout or threading models.
    why: >
      This is an engineering design challenge, not just implementation.
      In the interview, being able to articulate why your API makes certain choices
      (e.g., passing L2Snapshot by const ref instead of shared_ptr) demonstrates
      the seniority the JD is looking for.

  Policy-based design with templates:
    what: >
      Instead of inheritance-based runtime polymorphism, use template policies:
        template<typename RiskPolicy, typename ExecutionPolicy>
        class Strategy { ... };
      This allows zero-cost composition — the compiler inlines policy methods.
    why: >
      Virtual function calls cost ~5ns. In a hot path executing 100k events/sec,
      that's 500μs/sec of overhead from dispatch alone. Templates eliminate it.
      Whether to use virtual dispatch or templates is a common design interview question.

  Signal generation:
    what: >
      A signal is a directional view: buy, sell, or flat. Signals are derived from
      indicators computed on market data (price, volume, order flow imbalance, etc.)
      The strategy acts on signals — it doesn't trade directly on raw data.
    examples: >
      Momentum signal: if the last N mid-price changes are all positive, signal = buy.
      OFI (Order Flow Imbalance): (bid_qty_delta - ask_qty_delta) / total_delta.
        OFI > threshold → signal = buy, OFI < -threshold → signal = sell.

  Market-making basics:
    what: >
      A market-maker simultaneously quotes a bid below mid and an ask above mid.
      Spread earned on a round-trip (buy low, sell high) is the gross profit.
      Risk: inventory accumulates if the market moves directionally (adverse selection).
      Management: inventory skewing — tighten the side you want to reduce.
    implementation: >
      target_bid = mid - half_spread - skew(inventory)
      target_ask = mid + half_spread + skew(inventory)
      Cancel and repost on every significant book update.

  Risk controls as first-class objects:
    what: >
      Every strategy must have enforced limits:
        max_position: absolute inventory limit
        max_order_size: single order size cap
        max_drawdown: cumulative loss limit
        kill_switch: halt all activity immediately
      These are not optional. The JD explicitly lists "risk-management frameworks."
    implementation: >
      RiskGuard wraps every order submission. Strategy calls context.submit(order).
      RiskGuard checks limits before forwarding to ExchangeGateway.
      If any limit is breached, the order is rejected and the strategy is paused.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Define the Strategy base class and StrategyContext"
    detail: >
      In src/strategy/, write:

        class StrategyContext {
        public:
          void submit(Order order);
          void cancel(OrderId id);
          const L2Snapshot& book() const;
          Timestamp now() const;
          Position position() const;    // net position in base asset
          PnL realized_pnl() const;
        };

        class Strategy {
        public:
          virtual ~Strategy() = default;
          virtual void on_book_update(const L2Snapshot& book) {}
          virtual void on_fill(const OrderFill& fill) {}
          virtual void on_ack(const OrderAck& ack) {}
          virtual void on_cancel(const OrderCancel& cancel) {}
          virtual void on_reject(const OrderReject& reject) {}
          virtual std::string name() const = 0;
        protected:
          StrategyContext* ctx_ = nullptr;    // set by the runner
        };

      StrategyContext gives the strategy everything it needs to act.
      It hides the ring buffer, the matching engine, and threading entirely.
      A quant subclasses Strategy, overrides the on_* methods, calls ctx_->submit().

  2:
    task: "Implement RiskGuard"
    detail: >
      In src/risk/, write RiskGuard that wraps StrategyContext:
        struct RiskLimits {
          Quantity max_position;
          Quantity max_order_size;
          int64_t  max_drawdown_usd;    // in fixed-point cents
          bool     kill_switch = false;
        };

        class RiskGuard {
        public:
          RiskGuard(StrategyContext& ctx, RiskLimits limits);
          bool submit(Order order);   // returns false if blocked
          void trigger_kill_switch(); // cancels all open orders, halts strategy
        };

      Test: set max_position = 1000. Submit orders totalling 1100. Verify the 11th
      order is blocked. Trigger kill switch. Verify all open orders are cancelled.

  3:
    task: "Implement a market-making strategy"
    detail: >
      In src/strategy/, write MarketMaker : public Strategy:

        class MarketMaker : public Strategy {
          Quantity half_spread_ticks_ = 2;
          Quantity quote_size_ = 100;
          OrderId  current_bid_id_, current_ask_id_;
          bool     bid_live_ = false, ask_live_ = false;
        public:
          std::string name() const override { return "MarketMaker"; }
          void on_book_update(const L2Snapshot& book) override;
          void on_fill(const OrderFill& fill) override;
          void on_cancel(const OrderCancel& cancel) override;
        };

      on_book_update logic:
        1. Compute mid = (best_bid + best_ask) / 2
        2. Cancel current bid and ask if they exist
        3. Post new bid at mid - half_spread, ask at mid + half_spread
        4. If inventory is long, reduce the ask size (inventory skewing)

  4:
    task: "Implement a momentum strategy"
    detail: >
      In src/strategy/, write MomentumStrategy : public Strategy:

        class MomentumStrategy : public Strategy {
          std::deque<Price> mid_history_;   // last N mid prices
          int     lookback_ = 20;
          bool    position_open_ = false;
          OrderId open_order_id_;
        public:
          std::string name() const override { return "Momentum"; }
          void on_book_update(const L2Snapshot& book) override;
          void on_fill(const OrderFill& fill) override;
        };

      on_book_update logic:
        1. Compute current mid, append to mid_history_
        2. If history has N entries:
           - If all N deltas are positive and no position: submit market buy
           - If all N deltas are negative and long position: submit market sell
        3. Trim history to last N entries

  5:
    task: "Implement a multi-strategy runner"
    detail: >
      In src/, write StrategyRunner:
        - Holds a vector of Strategy*
        - On each event, routes to all registered strategies
        - Manages lifecycle: start(), stop(), pause(strategy_name)
        - Tracks per-strategy P&L and position

      Register both MarketMaker and MomentumStrategy. Run the feed simulator.
      Print per-strategy P&L at the end. This is the "portfolio" the JD mentions.

  6:
    task: "Quant-facing API review"
    detail: >
      Write a one-page design note (strategy_api_design.md in this folder):
        - What decisions did you make in the Strategy interface?
        - Why is StrategyContext an injected dependency and not a global?
        - Why are on_* methods virtual and not templates?
          (Answer: researcher-facing API — runtime polymorphism is acceptable here
           because it's not the hot path. The submission path through RiskGuard is
           the hot path and CAN be templated.)
        - What would you change if you had 10 quant researchers using this daily?

      This document is interview preparation. You'll be asked to describe your design.
```

---

## Done When

```yaml
done_when:
  - Strategy base class is clean — a new strategy requires overriding 2–3 methods only
  - RiskGuard enforces all limits and kill switch works correctly
  - MarketMaker posts quotes, updates on book changes, and skews on inventory
  - MomentumStrategy generates signals and submits orders correctly
  - StrategyRunner manages both strategies and prints per-strategy P&L
  - strategy_api_design.md documents your design decisions
  - You can explain the virtual-vs-template tradeoff in the context of this API
```

---

## Interview Signal

```yaml
what_this_demonstrates:
  - You can design a quant-facing API — an explicit JD requirement
  - You know the difference between researcher-facing and hot-path code
  - You understand market-making mechanics — the most common HFT strategy type
  - You can implement risk controls as first-class architectural objects
  - You can discuss API design tradeoffs, not just implementation
```

---

## Open Questions

```yaml
open_questions:
  - What is a TWAP/VWAP execution algorithm? How does it differ from a strategy?
  - What is order flow imbalance (OFI) and how is it calculated from L2 data?
  - How do firms handle correlated strategies that might trade the same instrument?
  - What is parameter optimization and why is overfitting a problem in strategy backtesting?
  - What is the Sharpe ratio? How do firms use it to evaluate strategy performance?
```
