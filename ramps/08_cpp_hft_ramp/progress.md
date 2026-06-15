# C++ HFT Ramp — Progress

```yaml
status: not_started
started: ""
completed: ""
total_stages: 7
stages_done: 0
```

---

## Stages

```yaml
stages:
  01_modern_cpp:
    status: not_started   # [ not_started | in_progress | complete ]
    started: ""
    completed: ""
    estimate: "5–7 days"
    artifact: "trading_engine/ — compiled project with RAII classes, templates, smart pointers"
    notes: ""

  02_systems_and_performance:
    status: not_started
    started: ""
    completed: ""
    estimate: "4–5 days"
    artifact: "Benchmarked hot paths, memory-aligned structs, lock-free queue prototype"
    notes: ""

  03_data_structures_algos:
    status: not_started
    started: ""
    completed: ""
    estimate: "5–7 days"
    artifact: "Order book implementation — bid/ask sides, L2 reconstruction, price-time priority"
    notes: ""

  04_exchange_mechanics:
    status: not_started
    started: ""
    completed: ""
    estimate: "3–4 days"
    artifact: "Simulated matching engine — limit/market/cancel, fill reports, feed replay"
    notes: ""

  05_strategy_and_interfaces:
    status: not_started
    started: ""
    completed: ""
    estimate: "4–5 days"
    artifact: "Strategy base class + two sample strategies (market-making, momentum)"
    notes: ""

  06_backtesting_framework:
    status: not_started
    started: ""
    completed: ""
    estimate: "4–5 days"
    artifact: "Event-driven backtest loop, P&L tracker, position ledger, CSV report"
    notes: ""

  07_capstone:
    status: not_started
    started: ""
    completed: ""
    estimate: "4–6 days"
    artifact: "Full pipeline — tick replay → order book → strategy → fills → P&L report"
    notes: ""
```

---

## Unlocks

```yaml
unlocks:
  hft_interview_screen:  "stages 01–03 required"
  system_design_round:   "stages 02–04 required"
  take_home_project:     "stages 01–05 required"
  full_loop_readiness:   "all 7 stages"
```

---

## Notes

<!-- freeform — blockers, surprises, things to revisit -->
