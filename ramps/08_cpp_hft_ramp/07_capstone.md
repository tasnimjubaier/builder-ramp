# Stage 07 — Capstone

**Status:** not started
**Estimated time:** 4–6 days
**Type:** integration + polish
**Depends on:** Stages 01–06 (all)

The capstone integrates everything into one coherent, presentable codebase. This is the take-home project, the GitHub portfolio artifact, and the system design interview reference. It proves you didn't just learn the concepts in isolation — you assembled them into something that runs, produces numbers, and can be explained end-to-end.

The output is `trading_engine/` — clean, buildable, documented — plus a written system design document and a recorded or live walkthrough you can give in an interview.

---

## What You're Building

- A complete pipeline: tick replay → order book → strategy → fills → P&L report
- A CLI driver (`./trading_engine --strategy mm --data ticks.csv --output report.csv`)
- A `DESIGN.md` that explains every architectural decision
- A performance benchmark document with real numbers
- A two-minute verbal walkthrough script

---

## Integration Checklist

```yaml
components_to_wire:
  tick_source:
    what: "CsvTickSource reading from a file argument"
    connects_to: "Backtest simulation loop"

  order_book:
    what: "OrderBook with L2 + L3 views, all order types, matching engine"
    connects_to: "ExchangeGateway, StrategyContext"

  exchange_gateway:
    what: "Accepts strategy orders, routes to matching engine, emits events"
    connects_to: "RiskGuard → OrderBook → event queue → strategies"

  risk_guard:
    what: "max_position, max_order_size, max_drawdown, kill_switch"
    connects_to: "ExchangeGateway (wraps every submission)"

  strategy_runner:
    what: "Routes events to all registered strategies, manages lifecycle"
    connects_to: "Strategy callbacks, StrategyContext"

  strategies:
    what: "MarketMaker and MomentumStrategy — selectable via CLI flag"
    connects_to: "StrategyContext, RiskGuard"

  position_ledger:
    what: "FIFO P&L, realized/unrealized tracking"
    connects_to: "Fill events from ExchangeGateway"

  performance_calc:
    what: "Sharpe, drawdown, win rate, slippage"
    connects_to: "PositionLedger time series"

  report_writer:
    what: "CSV output: per-trade log + summary metrics"
    connects_to: "PositionLedger, PerformanceCalc"

  cli_driver:
    what: "main.cpp with flag parsing: --strategy, --data, --output, --latency, --fee-bps"
    connects_to: "All of the above"
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Audit and clean the codebase"
    detail: >
      Compile with -Wall -Wextra -Wpedantic. Fix every warning.
      Run with -fsanitize=address,undefined. Fix every violation.
      Run with -fsanitize=thread on any concurrent code. Fix races.

      This step is not optional. A codebase with warnings or sanitizer failures
      is unpresentable as a portfolio artifact.

  2:
    task: "Build the CLI driver"
    detail: >
      Parse flags manually (no external libraries — keeps the project dependency-free):
        --strategy [mm|momentum|both]
        --data <path_to_csv>
        --output <report_output_path>
        --latency-ns <int> (default 1000)
        --fee-bps <int> (default 5)
        --max-position <int> (default 10000)
        --depth <int> (L2 levels to show, default 5)
        --verbose (print book snapshot every N ticks)

      Usage line: ./trading_engine --help

  3:
    task: "Generate a realistic 500,000-event dataset"
    detail: >
      Extend DataGenerator to produce a more realistic price series:
        - Use a geometric Brownian motion model for the mid price
          (mid[t] = mid[t-1] * exp(mu * dt + sigma * sqrt(dt) * Z), Z ~ N(0,1))
        - Add a trending phase (mu > 0) and a mean-reverting phase (mu < 0)
        - Realistic spread: 1–3 ticks for a liquid instrument
        - Volume distribution: log-normal, spike at round price levels

      Save to data/ticks_500k.csv. This is the canonical test dataset.

  4:
    task: "Run the full pipeline and produce a report"
    detail: >
      Run:
        ./trading_engine --strategy mm     --data data/ticks_500k.csv --output reports/mm.csv
        ./trading_engine --strategy momentum --data data/ticks_500k.csv --output reports/momentum.csv
        ./trading_engine --strategy both   --data data/ticks_500k.csv --output reports/both.csv

      Reports should include:
        - Per-trade log (timestamp, side, price, qty, fill_price, pnl)
        - Summary: total_pnl, sharpe, max_drawdown, win_rate, total_fills, avg_slippage

      Fix anything that crashes, produces NaN, or gives nonsensical numbers.

  5:
    task: "Benchmark the full pipeline"
    detail: >
      Measure: how long does the backtester take to process 500k ticks?
      Break it down:
        - Time in tick parsing
        - Time in order book update
        - Time in strategy on_book_update
        - Time in fill simulation

      Target: < 5 seconds total for 500k ticks on a modern laptop.
      If slower, profile and identify the bottleneck.

  6:
    task: "Write DESIGN.md"
    detail: >
      Cover these sections:
        Architecture overview: one diagram (ASCII is fine) showing all components
        Key design decisions:
          - Why SPSC ring buffer for feed → book communication
          - Why price-time priority with map + deque + hash index
          - Why Strategy is virtual dispatch, RiskGuard is not
          - Why FIFO for P&L vs LIFO or average cost
          - Why conservative fill simulation (last-in-queue assumption)
        Limitations of the current implementation:
          - Single instrument only
          - No real network layer
          - Fill simulation simplifications
        What would change for production:
          - Kernel bypass networking (DPDK or OpenOnload)
          - Custom memory allocator, no heap on hot path
          - Thread pinning and CPU affinity
          - Persistence layer for order and trade history

      This document is what you read from in a system design interview.

  7:
    task: "Write the verbal walkthrough script"
    detail: >
      A 2-minute script for: "Walk me through your trading engine project."

        Opening (15s): What it is and why you built it.
        Architecture (30s): Components, data flow, key design choices.
        Order book (20s): Data structure, why, benchmark number.
        Strategy layer (20s): API design, how a quant would add a new strategy.
        Backtester (20s): Event loop, fill simulation, performance metrics.
        Numbers (15s): Sharpe of MM strategy, tick processing speed.

      Practice until you can deliver it without notes in under 2 minutes.
```

---

## Done When

```yaml
done_when:
  - ./trading_engine --help works and documents all flags
  - Full pipeline runs on 500k ticks without sanitizer violations
  - Reports are generated for both strategies with sensible numbers
  - Benchmark document has real measurements for all pipeline stages
  - DESIGN.md covers all sections listed above
  - Verbal walkthrough is practiced and under 2 minutes
  - GitHub repo is clean: .gitignore excludes build/, data/, reports/; README links to DESIGN.md
```

---

## Interview Readiness Check

```yaml
questions_you_should_answer_confidently:
  language:
    - "Explain RAII. Give me a concrete example."
    - "What's the difference between std::unique_ptr and std::shared_ptr?"
    - "When would you use a template over virtual dispatch?"
    - "What is false sharing? How do you prevent it?"

  order_book:
    - "Implement a price-time priority order book. What data structures would you use?"
    - "How do you cancel an order in O(1)?"
    - "What is the difference between L2 and L3 market data?"
    - "What is the spread? What is adverse selection?"

  hft_systems:
    - "Why does cache locality matter for an order book?"
    - "What is a lock-free ring buffer and when would you use one?"
    - "Explain the full lifecycle of an order from submission to fill."
    - "What is a kill switch and why is it a hard requirement?"

  strategy_and_backtesting:
    - "How would you design an interface for quant researchers to write strategies?"
    - "What is lookahead bias in backtesting?"
    - "What is the Sharpe ratio? What does a Sharpe of 2.0 mean?"
    - "What is the difference between realized and unrealized P&L?"

  design:
    - "Walk me through your trading engine project."
    - "What would you change to make this production-ready?"
    - "How would you scale this to 50 instruments simultaneously?"
    - "What is the latency budget from feed event to order submission?"
```

---

## Notes

<!-- any observations from the integration run — bugs fixed, surprises, performance findings -->
