# Stage 06 — Backtesting Framework

**Status:** not started
**Estimated time:** 4–5 days
**Type:** additive
**Depends on:** Stage 05 (strategy layer, event model, P&L tracking)

The JD says: "maintain and enhance backtesting frameworks to evaluate trading strategies over historical data." This is a named responsibility, not a nice-to-have. Backtesting is the quant research loop — you can't ship a strategy without it. This stage builds an event-driven backtester that replays tick data, runs strategies through the full simulation, and produces a meaningful performance report.

---

## What You're Building

- An event-driven simulation loop with deterministic ordering
- A historical tick data replayer (CSV format, then binary for performance)
- A P&L and position ledger with trade-level accounting
- A performance report generator (Sharpe, max drawdown, win rate, avg fill)
- An integration test: run MarketMaker and Momentum against the same dataset

---

## Concepts This Stage Teaches

```yaml
concepts:
  Event-driven simulation:
    what: >
      A backtester is a simulation. Time is virtual — you advance it by replaying
      historical events. On each event, you update the book, route to strategies,
      process any orders they submit, update P&L, and advance to the next event.
      The loop is single-threaded and deterministic.
    why: >
      Determinism is what makes backtesting reliable. The same data + same strategy
      must produce the same result every run. Non-determinism (race conditions,
      random order processing) makes backtesting useless.
    loop: >
      while events_remain:
        event = tick_source.next()
        set_simulation_time(event.timestamp)
        update_order_book(event)
        emit_book_update_to_strategies()
        process_strategy_orders()   // any orders submitted in response
        update_position_ledger()

  Lookahead bias:
    what: >
      A strategy must not see future data. In a poorly built backtester, a strategy
      might compute a signal using data that wasn't available at the simulated time.
      This produces unrealistically good results that don't hold live.
    how_to_prevent: >
      Strict event ordering by timestamp. Strategy callbacks fire only after the
      relevant event is fully processed. No "peek ahead" in the tick source.

  Fill simulation:
    what: >
      In live trading, fills depend on queue position, liquidity, and counterparty.
      In a backtester, you approximate:
        - Limit order: filled if the simulated book crosses the order's price
        - Market order: filled at the current best bid/ask
        - Partial fill: if available quantity is less than order quantity
      Conservative assumption: assume limit orders fill at the back of the queue
      (you're last, not first) unless the entire level is consumed.
    why: >
      Optimistic fill assumptions (assume you're always first in queue) produce
      unrealistically high backtest P&L. Conservative assumptions give a realistic
      lower bound. The gap between them is the "execution quality uncertainty."

  Position and P&L accounting:
    what: >
      Position: net quantity held. Long = positive, short = negative.
      Realized P&L: locked in on a closing fill. (sell_price - buy_price) * qty.
      Unrealized P&L: mark-to-market of the current position at mid price.
      Total P&L = realized + unrealized.
    accounting_method: >
      Use FIFO (First In, First Out) for realized P&L calculation:
        Each buy creates a lot. Each sell closes the oldest open lot first.
        Realized P&L per lot = (close_price - open_price) * lot_qty.

  Performance metrics:
    what: >
      Sharpe ratio: (mean return - risk_free_rate) / std_dev(returns).
        > 1.0: acceptable. > 2.0: good. > 3.0: excellent for HFT.
      Max drawdown: largest peak-to-trough decline in cumulative P&L.
      Win rate: fraction of trades that are profitable.
      Average fill price vs mid: slippage measure.
      Turnover: total traded notional / avg position size.
    why: >
      These are the numbers quants use to evaluate strategies. Being able to
      generate and explain them is part of the "collaborate with quantitative
      researchers" responsibility in the JD.

  Simulation fidelity vs speed tradeoff:
    what: >
      High fidelity: model queue position, partial fills, latency, fees.
        Slow to build, accurate results.
      Low fidelity: assume all orders fill immediately at mid, no fees.
        Fast to build, optimistic results.
    practical_approach: >
      Build low fidelity first (this stage). Add fidelity incrementally:
        - Add exchange fees in the P&L ledger
        - Add simulated latency (delay strategy orders by N microseconds)
        - Add conservative queue position assumption
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Define the simulation loop"
    detail: >
      In src/backtest/, write Backtest:

        class Backtest {
        public:
          Backtest(std::unique_ptr<TickSource> source,
                   std::vector<Strategy*> strategies);
          void run();
          BacktestReport report() const;
        private:
          void process_tick(const Tick& tick);
          void process_strategy_orders();
          void update_pnl();
        };

      TickSource is an interface:
        class TickSource {
        public:
          virtual std::optional<Tick> next() = 0;
          virtual bool has_more() const = 0;
        };

      Implement CsvTickSource reading from your Stage 04 CSV format.

  2:
    task: "Implement position ledger with FIFO accounting"
    detail: >
      In src/backtest/, write PositionLedger:
        - open_lots: std::deque<Lot>  where Lot = {price, quantity, timestamp}
        - on_fill(Fill): if buy, push_back new lot. If sell, close lots from front.
        - realized_pnl(): sum of closed lot gains
        - unrealized_pnl(current_mid): open lots marked to current mid
        - position(): sum of open lot quantities

      Test: buy 100 at 100.00, buy 50 at 101.00, sell 120.
        First lot (100 @ 100.00) closes fully: +0 × 120... work through the math.
        Remainder (20) from second lot.
        Verify realized P&L calculation is correct.

  3:
    task: "Implement fill simulation in the backtester"
    detail: >
      In process_strategy_orders():
        - Market orders: fill immediately at current best bid/ask
        - Limit orders: added to a pending_orders list
          On each tick, check if any pending limit orders are now crossable
          Conservative: fill only if the entire price level is consumed by the tick
          (i.e., assume you're last in queue at that level)
        - IOC: fill what's available immediately, cancel rest
        - Reject if max_position or max_order_size would be breached

  4:
    task: "Implement performance metrics"
    detail: >
      In src/backtest/, write PerformanceCalc:

        struct BacktestReport {
          int64_t  realized_pnl_cents;
          int64_t  unrealized_pnl_cents;
          double   sharpe_ratio;
          double   max_drawdown_pct;
          double   win_rate;
          int64_t  total_fills;
          int64_t  total_orders_submitted;
          double   avg_fill_slippage_ticks;
          Timestamp start_time;
          Timestamp end_time;
        };

      Compute returns as a time series (P&L at each minute boundary).
      Sharpe = mean(returns) / std_dev(returns) * sqrt(annualization_factor).
      Max drawdown = max over all windows of (peak_pnl - trough_pnl) / peak_pnl.

  5:
    task: "Generate a synthetic dataset and run both strategies"
    detail: >
      Write a DataGenerator in src/feed/ that produces:
        - 100,000 tick events
        - Prices that trend, then mean-revert, then trend again
          (so momentum strategy wins in trend phases, market-maker wins in flat phases)
        - Realistic spread and volume distribution

      Run Backtest with both MarketMaker and MomentumStrategy on this dataset.
      Print BacktestReport for each. Compare their Sharpe ratios and max drawdowns.

  6:
    task: "Add latency simulation"
    detail: >
      Add a latency parameter to the backtester: strategy_latency_ns (default: 1000ns).
      When a strategy submits an order in response to a tick at time T,
      the order is not processed until simulation time T + strategy_latency_ns.

      Re-run both strategies with latency=0 and latency=1000ns. Compare P&L.
      For a market-making strategy, latency should noticeably degrade performance.
      This demonstrates why the firm cares about microsecond optimization.

  7:
    task: "Add exchange fees to P&L"
    detail: >
      Add fee_bps (basis points) to BacktestConfig. 1 bps = 0.01%.
      Apply fees on each fill: realized_pnl -= fill_notional * fee_bps / 10000.
      Re-run. Verify market-maker P&L is more sensitive to fees than momentum
      (market-maker has higher turnover).
```

---

## Done When

```yaml
done_when:
  - Backtest runs deterministically — same data + same strategy = same output every time
  - FIFO P&L accounting is correct (verified by hand on a small test case)
  - BacktestReport includes all metrics: Sharpe, drawdown, win rate, slippage
  - Both strategies run against the synthetic dataset and produce different profiles
  - Latency simulation demonstrably degrades market-maker performance
  - Fee simulation works and affects high-turnover strategies more
  - You can explain lookahead bias and how your backtester prevents it
```

---

## Interview Signal

```yaml
what_this_demonstrates:
  - You can build the tool quants use daily — a direct JD responsibility
  - You understand fill simulation fidelity tradeoffs — senior-level thinking
  - You know the performance metrics (Sharpe, drawdown) without Googling them
  - You can discuss why backtesting results differ from live performance
```

---

## Open Questions

```yaml
open_questions:
  - What is walk-forward optimization? How does it reduce overfitting vs in-sample tuning?
  - What is transaction cost analysis (TCA)? What does a firm measure with it?
  - What is the difference between a paper trader and a backtester?
  - How do you handle corporate actions (splits, dividends) in an equity backtester?
  - What is regime detection and why do strategies need to handle it?
```
