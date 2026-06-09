<div align="center">

# 🎯 Latency Hunter Toolkit

**A quant-engineering review plugin for solo traders — no alpha, no keys, no models. Bring your own edge.**

[![License: MIT](https://img.shields.io/badge/License-MIT-22c55e.svg)](LICENSE)
[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-plugin-d97757.svg)](https://docs.claude.com/en/docs/claude-code)
[![version](https://img.shields.io/badge/version-0.1.0-3b82f6.svg)](#roadmap)
[![skills](https://img.shields.io/badge/skills-2-8b5cf6.svg)](#the-two-skills)

**[中文](README.md) | English**

</div>

---

> The half-a-Sharpe your backtest inflates, live trading collects in full.
> One extra lock, one false-sharing event, one heap allocation on the hot path — your tail latency pays the bill.
> This toolkit won't find your alpha. It just keeps you from fooling yourself on the engineering.

## What this is

**Latency Hunter Toolkit** is a Claude Code **plugin** for the **solo quant (one-person company)**. The author persona, "Latency Hunter," is exactly what it sounds like: someone who stalks latency and traps, with zero tolerance for optimistic assumptions.

It is **not a strategy library, not a signal source, not a model.** It does one thing and one thing only: **due diligence on your quant engineering code** — surfacing the engineering pitfalls that inflate backtests, lose real money in production, and slow down your hot path — and writing you a graded "health report."

The core ethos:

> **No alpha · No keys · No models — engineering review only. Bring your own edge.**

It never touches your API keys, never predicts returns, never gives you entry/exit signals. It answers exactly three questions:

- Can this value **actually be computed at that point in time?** (look-ahead / leakage)
- Can this fill **actually happen in live trading?** (execution realism / cost)
- Why is this hot path **slow?** (lock contention / false sharing / cache miss / allocation / syscall)

It currently ships two skills, both governed by the same ironclad rules: **severity-graded, anchored to `file:line`, never speculative.**

## Install

### Option A — Claude Code plugin marketplace (recommended)

```bash
# 1. Add this repo as a plugin marketplace
/plugin marketplace add paidaxing1234/latency-hunter-toolkit

# 2. Install the plugin (both skills + both slash commands)
/plugin install latency-hunter
```

Once installed, both skills **trigger automatically** when you talk about backtests or hot paths, or you can invoke them explicitly via slash commands:

```bash
/backtest-guard   ./strategy        # audit a backtest/strategy directory
/latency-audit    ./engine/src      # audit a C++ hot path
```

### Option B — manual copy into your skills directory (fallback)

If you'd rather skip the marketplace and just grab the skills themselves:

```bash
# Clone the repo
git clone https://github.com/paidaxing1234/latency-hunter-toolkit.git

# Copy both skills into your user-level skills directory
mkdir -p ~/.claude/skills
cp -r latency-hunter-toolkit/skills/backtest-guard ~/.claude/skills/
cp -r latency-hunter-toolkit/skills/latency-audit  ~/.claude/skills/
```

Restart Claude Code, then just say "check this backtest for look-ahead bias" or "why is this hot path slow" to trigger them.

## The two skills

### 1. `backtest-guard` — the backtest lie detector

> Put a backtest/strategy on trial and interrogate it line by line for every trap that makes the curve look great and live trading take it all back.

**What it is** — a lie detector for quant backtests. A backtest's default assumptions are **optimistic** (zero cost, zero latency, perfect foresight, always filled); live trading's default assumptions are **hostile**. This skill exposes every optimistic assumption and grades its severity — it does **not** optimize your returns.

It covers four families, **48 checks** in total (full catalog in `skills/backtest-guard/references/backtest-pitfalls.md`):

| Family | What it catches |
|--------|-----------------|
| ① Look-ahead / data leakage | negative `shift`, same-bar signal-and-fill price peeking, whole-sample `fit` preprocessing, `bfill` injecting future values, `center=True` rolling, repainting indicators, missing point-in-time data, shuffled time-series splits… |
| ② Overfitting / data snooping / survivorship | no out-of-sample, grid search reporting only the best params, p-hacking, backtesting history on *today's* index members, too many free parameters, tiny samples, no multiple-testing correction… |
| ③ Execution realism / cost | zero commission, zero slippage, using intrabar high/low as fill price, assuming infinite liquidity, zero-latency limit-touch fills, ignoring liquidation/funding/borrow costs… |
| ④ Return-stat / metric integrity | wrong annualization factor (252 on crypto), arithmetic sum masquerading as compounding, wrong Sharpe denominator, mixing realized vs mark-to-market equity… |

**How it triggers**

- Automatically: drop in strategy / backtest / signal / data-loading / param-search code and ask "can I trust this backtest / why doesn't live match the backtest / check for look-ahead / is it overfit?"
- Trigger words: backtest audit, look-ahead bias, lookahead, data leakage, overfitting, survivorship bias, curve fitting, data snooping.
- Explicit: `/backtest-guard ./your-strategy`

**Sample health report (excerpt)**

```
══════════════════════════════════════════
   Backtest Health Report · backtest-guard
══════════════════════════════════════════
Subject: my-strategy/     Files scanned: 12     LoC: ~3,400

Summary: 🔴 Fatal 2 · 🟠 High 2 · 🟡 Medium 1 · 🔵 Low 0
Verdict: "This looks more like a fine-tuned overfit to the 2021 bull
          run than a strategy."

[🔴 FATAL] Negative shift injects future values
  Location: src/features.py:84
  Issue:    df['mom'] = df['close'].shift(-5) pulls the t+5 close to t,
            and that column feeds the X feature matrix (features.py:131).
  Fix:      Any column feeding features must shift(>=1); keep future
            returns in y/label only — never in X.

[🟠 HIGH] Zero commission + zero slippage
  Location: backtest/engine.py:–– (no commission/slippage found anywhere)
  Issue:    Fills booked at raw close; an intraday strategy turning over
            ~900x/yr has its costs completely ignored.
  Fix:      Model taker fees + half-spread slippage; run a cost-doubling
            sensitivity test.
══════════════════════════════════════════
```

> The report only gives a **directional read** ("which way performance moves once the bias is removed") and **never promises a concrete return figure.** Every finding is anchored to `file:line`; if there's no evidence, it's marked "not found / needs manual confirmation."

### 2. `latency-audit` — the hot-path hunter

> Put your C++ low-latency engine's hot path on the operating table and pick out every defect that steals microseconds and jitters your tail latency.

**What it is** — a reviewer for the hot path of a C++ low-latency / HFT trading engine. It first locates the real hot path (tick callback → signal → order send → matching → serialization → lock-free queue), then checks for the engineering problems that are invisible in the mean but lethal at **p99/p99.9 tail latency**, and writes you a "hot-path health report."

It covers three families:

| Family | What it catches |
|--------|-----------------|
| ① Concurrency / locks / false sharing | `mutex`/lock contention on the hot path, false sharing (shared counters not aligned to a cache line), spurious wakeups, over-strict atomic memory ordering… |
| ② Memory / cache / allocation | heap allocation on the hot path (`new`/`malloc`/implicit `std::string`/`std::vector` growth), cache misses, pointer chasing, unreserved capacity, copies instead of zero-copy… |
| ③ Syscall / IO / serialization / network / branch | syscalls on the hot path, unbatched network IO, serialization copies, unpredictable branches, missing `__builtin_expect`/`[[likely]]`, logging that blocks the hot path… |

**How it triggers**

- Automatically: drop in C++ low-latency / HFT engine hot-path code and ask "why is this hot path slow / check for lock contention / is there false sharing / where's the hidden heap allocation?"
- Trigger words: latency audit, hot path, lock contention, false sharing, cache miss, zero-copy, tail latency.
- Explicit: `/latency-audit ./engine/src`

**Sample health report (excerpt)**

```
══════════════════════════════════════════
   Hot-Path Health Report · latency-audit
══════════════════════════════════════════
Subject: engine/src/   Hot path: on_tick → signal → send_order

Summary: 🔴 Fatal 1 · 🟠 High 2 · 🟡 Medium 1
Verdict: "The mean looks pretty; p99.9 is paying off this lock and
          this heap allocation."

[🔴 FATAL] Lock held on the hot path
  Location: engine/src/book.cpp:212
  Issue:    on_tick() takes a std::lock_guard over the whole order-book
            update, contending with the matching thread; tail latency
            jitters under tick storms.
  Fix:      Move the lock off the hot path — SPSC lock-free queue /
            sequence-number publishing.

[🟠 HIGH] False sharing (shared counter not cache-line aligned)
  Location: engine/src/stats.hpp:30
  Issue:    Frequently written counters[] share one cache line, causing
            cache-line ping-pong across threads.
  Fix:      alignas(64) one counter per line, or thread-local accumulate
            then aggregate.
══════════════════════════════════════════
```

> The report recommends verifying with `perf` / `clang-tidy` and **only points a direction — it never promises a microsecond figure.** Whether it's truly a hot path, and how much faster your fix actually is, is whatever your own machine measures.

## Directory structure

```
latency-hunter-toolkit/
├── .claude-plugin/
│   ├── plugin.json           # plugin manifest (name / version / description)
│   └── marketplace.json      # marketplace manifest (for /plugin marketplace add)
├── commands/
│   ├── backtest-guard.md     # /backtest-guard slash command
│   └── latency-audit.md      # /latency-audit slash command
├── skills/
│   ├── backtest-guard/
│   │   ├── SKILL.md                          # protocol + 4 pitfall families + report template
│   │   └── references/
│   │       └── backtest-pitfalls.md          # 48-pitfall encyclopedia (why → smell → detect → fix)
│   └── latency-audit/
│       ├── SKILL.md                          # protocol + 3 families + report template
│       └── references/                       # hot-path pitfall references
├── LICENSE                   # MIT
├── README.md                 # 中文 (default)
└── README.en.md              # English (this file)
```

## Roadmap

`v0.1.0` is the foundation: two engineering-review skills. From here it grows along the "full-stack engineering review for the solo quant" line — **still no alpha, no keys, no models**:

- [ ] **`exchange-connector-forge`** — audit/scaffold exchange connectors: WebSocket reconnect & heartbeat, rate-limit & weight handling, order-state-machine consistency, `recvWindow`/clock sync, disconnect replay & idempotency.
- [ ] **`risk-killswitch-guard`** — audit live risk/circuit-breaker logic: position caps, max-drawdown breakers, per-trade/per-day loss thresholds, kill-switch reachability.
- [ ] **`order-state-machine-audit`** — audit the full order lifecycle state machine: races and missing states across open/partial/cancel/timeout.
- [ ] **`data-pipeline-integrity`** — market-data/factor pipeline integrity: gaps, timezone/DST, adjustment conventions, point-in-time persistence.

Want a skill we haven't built? Open an issue. The hunter is always looking for the next bit of latency.

## Compliance & disclaimer

**Latency Hunter Toolkit is a set of engineering-review tools, not an investment tool.**

- **Not investment advice.** It does **not predict returns, assess profitability, recommend assets, or give entry/exit/position advice.** "Passing the audit" ≠ "the strategy makes money" — it only means none of the pitfalls in this catalog were found.
- **No numbers promised.** Phrases like "bias impact" or "tail latency" in a report are **directional reads only** (typically: out-of-sample will underperform the backtest / a potential latency source exists). It **never promises or quantifies any specific return, loss, or microsecond figure.** Latency numbers are whatever your own `perf`/benchmarks measure.
- **The review has boundaries.** Conclusions depend on the code and visible context provided; risks in data not provided, external dependencies, or shifting market structure are beyond this tool's reach.
- **Your risk.** All live-trading decisions and capital risk are yours alone. Before committing real funds or adding leverage, validate in isolation with small size first.

## License

[MIT](LICENSE) © 2026 paidaxing (Latency Hunter)

---

<div align="center">

*No alpha. No keys. No models. Just the engineering review that keeps your edge honest.*

</div>
