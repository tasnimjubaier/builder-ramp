# C++ HFT Ramp — Learning Sequence

This ramp targets the Software Engineer C++ role at a Dubai/Amsterdam HFT firm. The job asks for professional-grade modern C++ and working knowledge of HFT systems — strategies, exchange mechanics, low-latency optimization, backtesting, and risk. That is five distinct skill clusters stacked on top of each other. This ramp builds them in order.

The ramp is additive. One codebase — a minimal trading engine skeleton — grows through all 7 stages. The capstone is a working order-book + strategy backtester that demonstrates every layer.

---

## Structure

```yaml
stages:
  01_modern_cpp:              "C++17/20 fluency — the language at production level"
  02_systems_and_performance: "Memory, latency, profiling — the HFT execution model"
  03_data_structures_algos:   "Order book, priority queues, ring buffers — trading primitives"
  04_exchange_mechanics:      "Market microstructure — order types, matching, feeds"
  05_strategy_and_interfaces: "Strategy abstractions, signal interfaces, quant-facing APIs"
  06_backtesting_framework:   "Historical simulation, event loop, P&L accounting"
  07_capstone:                "Full order-book backtester with pluggable strategy interface"
```

---

## What This Unlocks

```yaml
unlocks:
  hft_interview_screen:  "Stages 01–03 required — coding round is pure C++ + DSA"
  system_design_round:   "Stages 02–04 required — latency, exchange mechanics, feed handling"
  take_home_project:     "Stages 01–05 — strategy interface + working simulation"
  full_loop_readiness:   "All 7 stages — you can talk fluently to quants and engineers"
```

---

## Completion Criteria

```yaml
done_when:
  - You write modern C++17 idiomatically — RAII, move semantics, smart pointers, templates
  - You understand what cache misses, false sharing, and memory layout mean for hot paths
  - You can implement a price-time priority order book from scratch in under 2 hours
  - You can explain the full lifecycle of an order — from send to fill — including exchange internals
  - You can design a strategy interface that a quant researcher could use without knowing C++
  - You have a working backtester that replays tick data, runs a strategy, and outputs P&L
  - You can explain risk controls: position limits, drawdown stops, kill switches
```

---

## Approach

```yaml
rules:
  - Additive — one codebase (trading_engine/), grows through all stages
  - Every stage produces compilable, runnable code
  - No external trading libraries — build the primitives yourself so you understand them
  - Use C++17 throughout; introduce C++20 features where they clarify intent
  - The codebase is the interview portfolio — keep it clean and explainable
  - Market data simulation replaces live feed — no API keys needed
```

---

## Prerequisites

```yaml
assumed:
  - C++ syntax basics — you can write classes, loops, functions
  - Compiled a C++ program before
  - Basic data structures — you know what a stack, queue, and map are conceptually

not_assumed:
  - HFT domain knowledge — built from scratch in stages 03–04
  - Low-latency systems — taught in stage 02
  - Financial math — not required for this role at this level
```
