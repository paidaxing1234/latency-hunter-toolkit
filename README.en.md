<div align="center">

# 🎯 Latency Hunter Toolkit

**A quant-engineering review plugin for solo traders — no alpha, no keys, no models. Bring your own edge.**

[![License: MIT](https://img.shields.io/badge/License-MIT-22c55e.svg)](LICENSE)
[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-plugin-d97757.svg)](https://docs.claude.com/en/docs/claude-code)
[![version](https://img.shields.io/badge/version-0.3.0-3b82f6.svg)](#roadmap)
[![skills](https://img.shields.io/badge/skills-5-8b5cf6.svg)](#the-five-skills)

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

It currently ships five skills: four **review-type** (backtest-guard / latency-audit / orderbook-sanity / risk-config-lint, governed by the same ironclad rules — **severity-graded, anchored to `file:line`, never speculative**) plus one **generator-type** (connector-forge, which freezes the "connector forging protocol" into a runnable skeleton so your connector never leaks from the foundation up).

## Install

### Option A — Claude Code plugin marketplace (recommended)

```bash
# 1. Add this repo as a plugin marketplace
/plugin marketplace add paidaxing1234/latency-hunter-toolkit

# 2. Install the plugin (five skills + five slash commands)
/plugin install latency-hunter
```

Once installed, all five skills **trigger automatically** when you talk about backtests, hot paths, connectors, market data, or risk controls, or you can invoke them explicitly via slash commands:

```bash
/backtest-guard   ./strategy        # audit a backtest/strategy directory
/latency-audit    ./engine/src      # audit a C++ hot path
/connector-forge  ./my-bot          # generate/repair an exchange connector skeleton
/orderbook-sanity ./data            # audit market-data / ingestion code for quality pitfalls
/risk-config-lint ./risk            # audit risk config / pre-trade risk code
```

### Option B — manual copy into your skills directory (fallback)

If you'd rather skip the marketplace and just grab the skills themselves:

```bash
# Clone the repo
git clone https://github.com/paidaxing1234/latency-hunter-toolkit.git

# Copy all five skills into your user-level skills directory
mkdir -p ~/.claude/skills
cp -r latency-hunter-toolkit/skills/backtest-guard   ~/.claude/skills/
cp -r latency-hunter-toolkit/skills/latency-audit    ~/.claude/skills/
cp -r latency-hunter-toolkit/skills/connector-forge  ~/.claude/skills/
cp -r latency-hunter-toolkit/skills/orderbook-sanity ~/.claude/skills/
cp -r latency-hunter-toolkit/skills/risk-config-lint ~/.claude/skills/
```

Restart Claude Code, then just say "check this backtest for look-ahead bias", "why is this hot path slow", "generate a Binance connector / my WS keeps dropping, add reconnect", "is this market data clean / are there kline gaps", or "check my risk config / is the kill-switch sound" to trigger them.

## The five skills

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

### 4. `orderbook-sanity` — the market-data lie detector

> Put your order-book / kline / tick data (or the ingest, parse, persist, and load code behind it) on trial and expose every engineering pitfall that quietly corrupts the data and poisons your strategy and backtest.

**What it is** — a lie detector for market-data quality. Dirty data is a **silent killer**: crossed books, checksum mismatches, kline gaps, and timestamp drift **don't throw, don't crash** — they just let your factors, signals, and backtests quietly build on sand. The most dangerous data isn't the *obviously* broken kind; it's data that's **numerically valid, correctly typed, yet semantically wrong** — a millisecond timestamp landing in 1970, a still-forming unclosed bar, a local book whose sequence numbers run continuous but were applied incorrectly. It does one thing only: anchor those pitfalls to `file:line`, tell you why the data goes dirty, how to self-verify, and how to fix it — and it **never touches the strategy layer.**

It covers three families (full catalog, detection code, and fixes in `skills/orderbook-sanity/references/orderbook-pitfalls.md`):

| Family | What it catches |
|--------|-----------------|
| ① Order book / quotes | checksum/CRC32 mismatch, snapshot↔increment splice errors, crossed book (`best_bid >= best_ask`), `qty==0` delete-level semantics, depth gaps, sequence gaps… |
| ② Kline / tick | using an unclosed/forming bar (look-ahead leakage), kline gaps / missing bars, duplicate bars / duplicate trades, out-of-order timestamps / tradeId gaps, OHLC invariant violations, outlier price spikes… |
| ③ Generic timestamp / alignment | ms/s/ns/us unit confusion, universe survivorship bias, future timestamps, halt vs real-gap confusion, timezone errors, clock skew, symbol normalization… |

**How it triggers**

- Automatically: drop in market-data samples or ingest/parse/persist/load code and ask "is this data clean / why doesn't live match the backtest / check my klines for gaps / are the timestamps wrong / how can the book be crossed / my local book doesn't match the exchange?"
- Trigger words: orderbook sanity, market data quality, crossed book, checksum mismatch, snapshot delta splice, sequence gap, kline gap, duplicate bar, out-of-order timestamps, unit confusion, clock skew, survivorship bias.
- Explicit: `/orderbook-sanity ./data`

**Sample health report (excerpt)**

```
══════════════════════════════════════════
   Market-Data Health Report · orderbook-sanity
══════════════════════════════════════════
Pipeline: ingest(WS) → parse → persist(db) → load
Scope: feed/ + samples/   Files scanned: 8

Summary: 🔴 Fatal 3 · 🟠 High 2 · 🟡 Medium 1 · 🔵 Low 1
Verdict: "The sequence looks continuous; the checksum stopped matching
          a while ago — this book is built on sand."

[🔴 FATAL] Checksum never verified (local book may be silently desynced)
  Location: src/feed/orderbook_okx.py:–– (subscribes books channel,
            no crc32 comparison anywhere)
  Issue:    OKX sends a checksum on every frame; the code only stitches
            by seq and never compares it.
  Harm:     when levels are applied wrong / truncated, a continuous seq
            can't catch it → the local book silently drifts.
  Fix:      reproduce the checksum verbatim; on mismatch, drop the local
            book and re-pull snapshot; use fixed-point integers as price keys.

[🔴 FATAL] Using an unclosed bar (leakage)
  Location: src/feed/kline_ws.py:88
  Issue:    k.x field unchecked; the forming bar's close feeds the signal.
  Check:    last_bar.close_time > now() means an unclosed bar was used.
  Fix:      accept as final only when k.x==true; mark forming bars
            provisional, never into signals.
══════════════════════════════════════════
```

> A matched pattern is a **lead, not a verdict**: real halt gaps, zero-volume buckets on illiquid symbols, a momentary locked book in fast markets, and genuine cross-exchange spreads are all **real market state** — every pitfall is tagged with a "legitimate exception" to prevent false positives. The report only points a direction and ships runnable self-check assertions; it **never promises "perfectly clean after the fix."**

### 5. `risk-config-lint` — the risk-control checkup

> Put your risk config (`risk_config.json/yaml`) and pre-trade risk code on the operating table and pick out every engineering defect that makes risk control toothless and blows up the account in production.

**What it is** — risk control is the **last gate** on the whole trading path: it earns nothing day to day and has zero presence, but the moment it leaks, it's capital wiped to zero and a margin-debt hole. It hunts a particularly insidious class of defect: **looks protected, actually inert** — the config is full of `max_position`/`kill_switch`, yet the order path never calls them; the stop-loss denominator is wrong so it never fires; a limit-load failure does `except: pass` and keeps trading naked; risk only counts *filled* positions while several concurrent in-flight orders already breach the limit in aggregate. These are *more* dangerous than having no risk control at all, because they hand you **false confidence.** Three judgments run throughout: **pre-trade block vs post-trade alert / fail-closed vs fail-open / programmatic closed loop vs relying on a human.**

It covers two families (full code counter-examples in `skills/risk-config-lint/references/risk-pitfalls.md`):

| Family | What it catches |
|--------|-----------------|
| ① Limits / concurrent in-flight / drawdown / leverage / kill-switch / fat-finger | missing pre-trade gate (only post-trade), missing per-order / gross-exposure cap, unbounded leverage, exposure that omits in-flight orders (concurrency race), missing drawdown stop + wrong denominator, alert-only with no auto-flatten, missing global kill-switch, emergency actions not isolated from business order flow, missing fat-finger guards… |
| ② Margin / liquidation / state trust / single point of failure / ops / secrets | limits not actually enforced pre-trade, fail-open on load failure, swallowed risk exceptions, position/equity state never reconciled, no maintenance-margin / liquidation simulation / negative cash allowed, testnet/live confusion, no heartbeat/deadman + single alert channel, secrets committed to config/repo, no config schema validation… |

**How it triggers**

- Automatically: drop in `risk_config.json` / pre-trade risk / order-validation code and ask "does my risk control have holes / why did live blow past the risk limits without stopping / check my kill-switch / stop-loss / leverage cap / are the limits actually enforced / can concurrent orders breach the limit?"
- Trigger words: risk config, risk limit, risk audit, kill switch, fat finger, liquidation, margin, drawdown stop, position limit, in-flight order.
- Explicit: `/risk-config-lint ./risk`

**Sample health report (excerpt)**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
          Risk Config Lint Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Subject  : risk_config.json + order_gate.py
Verdict  : ❌ Not deployable (4 fatal / 2 high)
One-liner: limits fully configured but never called on the order path;
           pre-trade gate missing and in-flight orders excluded from
           exposure — risk control is toothless
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[FATAL-1] Risk only blocks post-trade; no pre-trade gate
  Location: order_gate.py:42  submit_order()
  State   : no risk check before submit(); breaches only alert in on_fill
  Impact  : the over-limit exposure is already filled; can't exit in an
            extreme move → blow-up
  Fix     : every order passes a unified risk gate synchronously before
            submit; any failure raises RejectOrder

[FATAL-3] Exposure omits in-flight orders (concurrency race)
  Location: order_gate.py:30  check_exposure() reads position only
  State   : check-and-submit non-atomic; pending orders not counted;
            N rapid orders each pass, the aggregate breaches the limit
  Fix     : exposure = filled position + reserved notional of in-flight
            orders; make check-and-submit atomic under a lock or use
            reserve-then-commit; re-test with concurrent orders on testnet
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

> Every finding carries at least `Location / State / Impact / Fix`, and the report gives a hard deployable-or-not verdict. It distinguishes **actually enforced** from **merely configured**: a field in the config doesn't mean risk control fires — it traces backward from `place_order` to a gate that actually `raise`/returns a reject. It **never promises "won't blow up / absolutely safe"** — all FATAL items must be fixed before going live, and after the fix you must re-test each gate on testnet / small size to confirm it really closes.

## Directory structure

```
latency-hunter-toolkit/
├── .claude-plugin/
│   ├── plugin.json           # plugin manifest (name / version / description)
│   └── marketplace.json      # marketplace manifest (for /plugin marketplace add)
├── commands/
│   ├── backtest-guard.md     # /backtest-guard slash command
│   ├── latency-audit.md      # /latency-audit slash command
│   ├── connector-forge.md    # /connector-forge slash command
│   ├── orderbook-sanity.md   # /orderbook-sanity slash command
│   └── risk-config-lint.md   # /risk-config-lint slash command
├── skills/
│   ├── backtest-guard/
│   │   ├── SKILL.md                          # protocol + 4 pitfall families + report template
│   │   └── references/
│   │       └── backtest-pitfalls.md          # 48-pitfall encyclopedia (why → smell → detect → fix)
│   ├── latency-audit/
│   │   ├── SKILL.md                          # protocol + 3 families + report template
│   │   └── references/                       # hot-path pitfall references
│   ├── connector-forge/
│   │   ├── SKILL.md                          # connector forge: 6-step protocol + 2-exchange cheat sheet + reliability checklist
│   │   └── references/
│   │       └── connector-blueprint.md        # full Binance/OKX connector templates (endpoints/signing/rate limits/renewal)
│   ├── orderbook-sanity/
│   │   ├── SKILL.md                          # market-data lie detector: review protocol + 3 families + data health report template
│   │   └── references/
│   │       └── orderbook-pitfalls.md         # market-data pitfall catalog (detection code + code smell + fix)
│   └── risk-config-lint/
│       ├── SKILL.md                          # risk checkup: review protocol + 2 families + risk health report template
│       └── references/
│           └── risk-pitfalls.md              # risk pitfall code counter-examples (limits section + ops section)
├── LICENSE                   # MIT
├── README.md                 # 中文 (default)
└── README.en.md              # English (this file)
```

## Roadmap

`v0.3.0` packs two more layers onto the foundation: this release adds **orderbook-sanity (the market-data lie detector)** and **risk-config-lint (the risk-control checkup)**, expanding the matrix to four reviewers + one generator (backtest-guard / latency-audit / orderbook-sanity / risk-config-lint + connector-forge). From here it grows along the "full-stack engineering review for the solo quant" line — **still no alpha, no keys, no models**:

- [x] **`connector-forge`** — ✅ shipped (v0.2.0): one-shot generate/repair a production-grade Binance·OKX market-data + trading connector skeleton — WebSocket reconnect & heartbeat, rate-limit token bucket, signing & clock sync, listenKey/login renewal, re-subscribe, sequence-gap re-snapshot, keys via env; also audits existing connectors for reliability gaps.
- [x] **`orderbook-sanity`** — ✅ shipped (v0.3.0): reviews order-book/kline/tick data quality — crossed books, checksum mismatch, snapshot/increment splice, sequence/depth gaps, kline gaps/duplicates/out-of-order, unclosed-bar leakage, timestamp unit/timezone/clock drift, symbol normalization, universe survivorship bias; every finding tagged with a "legitimate exception" to separate real pitfalls from real market state.
- [x] **`risk-config-lint`** — ✅ shipped (v0.3.0): reviews risk config and pre-trade risk code — position/exposure caps, drawdown stops, leverage, kill-switch, fat-finger guards, margin/liquidation protection, in-flight order concurrency races, fail-open vs fail-closed, pre-trade vs post-trade, single points of failure and secrets.
- [ ] **`risk-killswitch-guard`** — runtime circuit-breaker review: actual kill-switch reachability, the "3 a.m. spike, nobody awake" failure mode, controlled reset flow (complements the static risk-config-lint, focused on runtime behavior).
- [ ] **`order-state-machine-audit`** — audit the full order lifecycle state machine: races and missing states across open/partial/cancel/timeout.
- [ ] **`data-recorder-audit`** — market-data recording/replay pipeline review: persistence idempotency, resumable recording, replay-vs-live alignment, point-in-time consistency.
- [ ] **`exec-quality-audit`** — execution-quality review: slippage/fill-rate attribution, cancel/replace races, TWAP/VWAP child-order logic, divergence between live fills and backtest fill assumptions.

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
