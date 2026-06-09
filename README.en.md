<div align="center">

# 🎯 Latency Hunter Toolkit

**A quant-engineering review plugin for solo traders — no alpha, no keys, no models. Bring your own edge.**

[![License: MIT](https://img.shields.io/badge/License-MIT-22c55e.svg)](LICENSE)
[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-plugin-d97757.svg)](https://docs.claude.com/en/docs/claude-code)
[![version](https://img.shields.io/badge/version-0.2.0-3b82f6.svg)](#roadmap)
[![skills](https://img.shields.io/badge/skills-3-8b5cf6.svg)](#the-three-skills)

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

It currently ships three skills: two **review-type** (backtest-guard / latency-audit, governed by the same ironclad rules — **severity-graded, anchored to `file:line`, never speculative**) plus one **generator-type** (connector-forge, which freezes the "connector forging protocol" into a runnable skeleton so your connector never leaks from the foundation up).

## Install

### Option A — Claude Code plugin marketplace (recommended)

```bash
# 1. Add this repo as a plugin marketplace
/plugin marketplace add paidaxing1234/latency-hunter-toolkit

# 2. Install the plugin (three skills + three slash commands)
/plugin install latency-hunter
```

Once installed, all three skills **trigger automatically** when you talk about backtests, hot paths, or connectors, or you can invoke them explicitly via slash commands:

```bash
/backtest-guard   ./strategy        # audit a backtest/strategy directory
/latency-audit    ./engine/src      # audit a C++ hot path
/connector-forge  ./my-bot          # generate/repair an exchange connector skeleton
```

### Option B — manual copy into your skills directory (fallback)

If you'd rather skip the marketplace and just grab the skills themselves:

```bash
# Clone the repo
git clone https://github.com/paidaxing1234/latency-hunter-toolkit.git

# Copy all three skills into your user-level skills directory
mkdir -p ~/.claude/skills
cp -r latency-hunter-toolkit/skills/backtest-guard  ~/.claude/skills/
cp -r latency-hunter-toolkit/skills/latency-audit   ~/.claude/skills/
cp -r latency-hunter-toolkit/skills/connector-forge ~/.claude/skills/
```

Restart Claude Code, then just say "check this backtest for look-ahead bias", "why is this hot path slow", or "generate a Binance connector / my WS keeps dropping, add reconnect" to trigger them.

## The three skills

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

### 3. `connector-forge` — the connector forge

> Freezes "how to forge a connector that doesn't leak, from the foundation up" into an executable protocol, and one-shot generates a runnable Binance / OKX market-data + trading connector skeleton (or audits an existing connector for reliability gaps).

**What it is** — the first two skills are **review-type** (they nitpick); this one is a **generator-type scaffold.** A connector is the **foundation** of a quant system: it doesn't make money directly, but **one leak and the whole thing collapses** — 90% of "live doesn't match the backtest", "mysterious disconnects", and "occasional order failures" originate in this layer, and they're all **silent failures** (the connection looks healthy while the data has long been dead or wrong). This skill freezes one ironclad rule: **declare it dead and rebuild explicitly, never run sick.**

What it generates / repairs is engineering, not alpha:

| Dimension | What it forges |
|-----------|----------------|
| ① Won't connect / keeps dropping | exponential backoff + full jitter reconnect, heartbeat **watchdog** (monotonic-clock death detection, defeats TCP half-open "zombie" connections), early reconnect before Binance's 24h forced close, OKX `'ping'/'pong'` application-layer heartbeat |
| ② Data doesn't match | re-subscribe the **whole "expected subscription set" and verify acks** after reconnect, **per-message continuity check** on sequenced increments, gap → discard → re-pull snapshot and replay, local order book `qty==0` deletes a level + checksum backstop |
| ③ Signature / clock errors | Binance HMAC-SHA256 (query+body) / OKX prehash→Base64 + passphrase, timestamps based on **calibrated exchange time** (defeats -1021 / 50102), `listenKey` renewal every 30 min / OKX `login` renewal |
| ④ Rate limit / idempotency / security | a token bucket priced by **weight, not request count**, capped at 80% of the limit, per-order `clientOrderId` idempotency + "verify-then-resend" on timeout, keys injected **only from env** + fail-fast + log masking (never hardcoded) |

**How it triggers**

- Automatically: say "generate / write me a Binance/OKX connector", "scaffold a market-data subscription", "my WS keeps dropping, add reconnect", or report a fault: "signature error -1022/50113", "timestamp error -1021/50102", "listenKey expired, not receiving fills", "no data after reconnect", "local order book doesn't match the exchange".
- Trigger words: connector, connector-forge, exchange connector, market data feed, websocket reconnect, heartbeat, listenKey, ws login, orderbook snapshot, sequence gap, rate limit token bucket, clock drift.
- Explicit: `/connector-forge ./my-bot`

**Generated connector skeleton (directory layout)**

```
exchange_connector/
├── config.py            # ENV enum, endpoint map (testnet/live), token-bucket quotas (marked TODO: defer to exchangeInfo)
├── credentials.py       # read key/secret/passphrase from os.environ, fail-fast, mask() for logs
├── clock.py             # server-time sync, offset maintenance, drift alerts
├── ratelimit.py         # async token bucket priced by weight
├── ws_market.py         # market WS: combined-stream subscribe + expected set + ping-pong watchdog + backoff reconnect + re-subscribe
├── orderbook.py         # snapshot+increment stitching, sequence-gap detection, qty==0 deletes, checksum
├── ws_user.py           # user data stream: listenKey renewal / OKX login + REST reconciliation after disconnect
├── rest_client.py       # signing, order placement (clientOrderId idempotency), 5xx verification, error-class mapping table
├── reconnect.py         # exponential backoff + full jitter, heartbeat watchdog (monotonic)
├── shutdown.py          # graceful shutdown on SIGTERM/SIGINT
└── README.md            # which env vars to run, testnet→live switch steps, TODO checklist
```

> Delivered with a checkbox-by-checkbox **connector self-check checklist** (implemented / TODO / N/A) so you or code-reviewer can see at a glance where the foundation still leaks. Key security is top priority: **never hardcoded** — only env-injection points + fail-fast; the skeleton **defaults to testnet**, and switching to live requires explicit confirmation. The **full templates** for both exchanges — endpoint routing, signing, rate limits, renewal — are in **`skills/connector-forge/references/connector-blueprint.md`**.

## Directory structure

```
latency-hunter-toolkit/
├── .claude-plugin/
│   ├── plugin.json           # plugin manifest (name / version / description)
│   └── marketplace.json      # marketplace manifest (for /plugin marketplace add)
├── commands/
│   ├── backtest-guard.md     # /backtest-guard slash command
│   ├── latency-audit.md      # /latency-audit slash command
│   └── connector-forge.md    # /connector-forge slash command
├── skills/
│   ├── backtest-guard/
│   │   ├── SKILL.md                          # protocol + 4 pitfall families + report template
│   │   └── references/
│   │       └── backtest-pitfalls.md          # 48-pitfall encyclopedia (why → smell → detect → fix)
│   ├── latency-audit/
│   │   ├── SKILL.md                          # protocol + 3 families + report template
│   │   └── references/                       # hot-path pitfall references
│   └── connector-forge/
│       ├── SKILL.md                          # connector forge: 6-step protocol + 2-exchange cheat sheet + reliability checklist
│       └── references/
│           └── connector-blueprint.md        # full Binance/OKX connector templates (endpoints/signing/rate limits/renewal)
├── LICENSE                   # MIT
├── README.md                 # 中文 (default)
└── README.en.md              # English (this file)
```

## Roadmap

`v0.2.0` packs another layer onto the foundation: two **review-type** skills (backtest-guard / latency-audit) plus one **generator-type** skill (connector-forge). From here it grows along the "full-stack engineering review for the solo quant" line — **still no alpha, no keys, no models**:

- [x] **`connector-forge`** — ✅ shipped (v0.2.0): one-shot generate/repair a production-grade Binance·OKX market-data + trading connector skeleton — WebSocket reconnect & heartbeat, rate-limit token bucket, signing & clock sync, listenKey/login renewal, re-subscribe, sequence-gap re-snapshot, keys via env; also audits existing connectors for reliability gaps.
- [ ] **`risk-killswitch-guard`** — audit live risk/circuit-breaker logic: position caps, max-drawdown breakers, per-trade/per-day loss thresholds, kill-switch reachability.
- [ ] **`order-state-machine-audit`** — audit the full order lifecycle state machine: races and missing states across open/partial/cancel/timeout.
- [ ] **`orderbook-sanity`** — local order-book consistency review: snapshot/increment stitching, sequence gaps, checksum validation, fixed-point price keys, level-deletion logic.
- [ ] **`risk-config-lint`** — static check of risk config: leverage/position/stop-loss thresholds that are missing, mutually contradictory, or hardcoded to dangerous defaults.
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
