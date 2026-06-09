# 订单簿盘口数据质量陷阱百科（Orderbook Sanity Pitfalls Encyclopedia）

> `SKILL.md` 指向的「完整数据质量陷阱百科」。按三大类组织——① 订单簿/盘口、② K线/tick、③ 通用时间戳对齐——逐条给出：为什么坑（why）→ 数据/代码里长啥样（code smell，Python 反例）→ 怎么检测（detect，可执行校验/断言）→ 怎么修（fix，正确做法）。
>
> **严重度标记**：`【fatal】` 致命，结论直接作废 / 实盘必错（典型为未来函数、单位错乱、幸存者偏差）；`【high】` 高危，系统性污染指标与对齐；`【medium】` 中危，量级失真但方向不一定反。
>
> **用法**：命中模式只是嫌疑，定位到 `文件:行` 读懂语义再定罪；按严重度汇总进数据体检报告，致命/高危优先；致命项清零前不建议把这批数据喂进信号或回测。

---

## 总览速查表

| # | 陷阱 | 严重度 | 类别 |
|---|------|--------|------|
| 1.1 | 本地簿与快照失同步（snapshot/diff desync 未重快照） | fatal | ① 订单簿/盘口 |
| 1.2 | 盘口交叉/锁定（crossed/locked book, bid ≥ ask） | fatal | ① |
| 1.3 | 增量更新丢失/乱序（dropped / out-of-order diffs） | fatal | ① |
| 1.4 | 深度截断当完整簿（depth truncation） | high | ① |
| 1.5 | 陈旧盘口当实时（stale top-of-book） | high | ① |
| 1.6 | 单边枯竭/一档为空（one-side empty） | medium | ① |
| 1.7 | 价格档位 tick/精度不对齐（float key / tick misalign） | medium | ① |
| 2.1 | K线缺口 / 缺失 bar（kline gap） | high | ② K线/tick |
| 2.2 | 重复 bar（duplicate kline） | high | ② |
| 2.3 | 时间戳乱序（out-of-order kline timestamps） | high | ② |
| 2.4 | interval 对齐错误（bar boundary misalignment） | high | ② |
| 2.5 | 使用未闭合 bar（unclosed/forming bar leakage） | fatal | ② |
| 2.6 | 未来时间戳（future timestamps） | high | ② |
| 2.7 | OHLC 不变式被破坏（OHLC invariant violation） | high | ② |
| 2.8 | 活跃 symbol 零成交量 bar（zero volume on active symbol） | medium | ② |
| 2.9 | 价格离群尖刺（price outlier spike / fat finger） | high | ② |
| 2.10 | tick/trade 乱序（out-of-order trades） | high | ② |
| 2.11 | 重复成交（duplicate trades） | high | ② |
| 2.12 | tradeId 缺口（trade sequence gap） | high | ② |
| 2.13 | 时间戳漂移与非单调（timestamp drift / non-monotonic） | medium | ② |
| 3.1 | 时间戳单位混淆（ms/s/ns/us unit confusion） | fatal | ③ 通用时间戳对齐 |
| 3.2 | 时区错误（timezone / naive-aware mismatch） | high | ③ |
| 3.3 | 时钟漂移（clock skew local vs exchange） | high | ③ |
| 3.4 | 跨交易所/跨源对齐错误（cross-exchange alignment） | high | ③ |
| 3.5 | symbol 归一化错误（symbol normalization） | high | ③ |
| 3.6 | 停盘缺数据 vs 真缺口混淆（halt vs real gap） | medium | ③ |
| 3.7 | universe 幸存者偏差（survivorship bias） | fatal | ③ |

---

# ① 订单簿 / 盘口（Orderbook & Top-of-Book）

这一类的共同特征：**本地维护的订单簿与交易所真实状态悄悄失同步**。增量推送（diff/depth update）一旦丢一条、乱一次序、或快照衔接错位，本地簿就开始偏离——而它**照样能算出 best_bid/best_ask、midprice、价差**，只是全错。统一防御口径：**序列号逐条校验连续，一旦断裂立即丢弃本地簿、重拉 snapshot 对齐；每次更新后断言 `best_bid < best_ask`。** 订单簿的「自动重连 + 序列缺口重快照」协议详见姊妹 skill `connector-forge` 的连接器蓝图。

## 1.1 本地簿与快照失同步（snapshot/diff 衔接错误，序列断裂未重快照）【fatal】

**why** —— 维护本地订单簿要先订增量、再拉 snapshot、丢弃 snapshot 之前的增量、再衔接。任何一步错（先拉快照后订增量丢中间更新、未按 `lastUpdateId` 丢弃旧增量、序列断裂不重拉），本地簿就与交易所永久错位，且不报错——你基于一个幻象盘口报价。

**code smell**

```python
# 反例：拉了 snapshot 就直接用，之后增量来者不拒，从不校验序列衔接
book = rest.depth_snapshot(symbol)          # REST 快照
async for u in ws_depth(symbol):            # 增量
    apply_update(book, u)                    # 不校验 u['U']/u['u'] 与 lastUpdateId 的衔接
```

**detect** —— 检查 snapshot 与首条增量的衔接：Binance 规则应满足 `first.U <= lastUpdateId+1 <= first.u`，后续 `u_prev + 1 == U_cur`。grep 应用增量处是否有序列连续性断言；无 `lastUpdateId`/`pu`/`prevSeq` 校验即高度可疑。

**fix**

```python
# 正确：订增量入缓冲 → 拉 snapshot → 丢弃 <= lastUpdateId 的增量 → 衔接 → 逐条校验连续
buf = []
async for u in ws_depth(symbol):
    buf.append(u)                                       # 先边订边缓冲
snap = rest.depth_snapshot(symbol); last = snap['lastUpdateId']
book = build(snap)
for u in buf:
    if u['u'] <= last: continue                         # 丢弃快照前增量
    assert u['U'] <= last + 1 <= u['u'], "衔接断裂 → 重来"
    apply_update(book, u); last = u['u']
# 之后每条：if u['U'] != last + 1 → 丢弃本地簿、重拉 snapshot（resync）
```

## 1.2 盘口交叉 / 锁定（crossed / locked book：bid ≥ ask）【fatal】

**why** —— `best_bid >= best_ask` 在正常市场不该出现，它是本地簿失同步、删档错误或买卖两边更新错位的**确诊信号**。此时 midprice、价差、报价信号全是垃圾，基于它下单可能成交在错误价位。

**code smell**

```python
mid = (book.best_bid + book.best_ask) / 2     # 从不检查 bid < ask
spread = book.best_ask - book.best_bid        # 可能为负却照用
```

**detect** —— 每次更新后断言 `book.best_bid < book.best_ask`；批量数据扫 `(bid >= ask).sum()`。crossed 持续存在 = 本地簿坏了，需重快照。

**fix**

```python
def best(book):
    bid, ask = book.bids.max_price(), book.asks.min_price()
    if bid is not None and ask is not None and bid >= ask:
        raise BookCrossed(bid, ask)   # fail-loud：触发重拉 snapshot，绝不静默用
    return bid, ask
```

## 1.3 增量更新丢失 / 乱序（dropped / out-of-order diffs）【fatal】

**why** —— WS 增量经网络可能丢包或乱序到达。漏掉一条 update，本地某档位量价就永久停留在旧值；乱序应用则把新值覆盖回旧值。本地簿静默漂移。

**code smell**

```python
async for u in ws_depth(symbol):
    apply_update(book, u)    # 不看序列号，丢了/乱了都照单全收
```

**detect** —— 维护 `expected_seq`，每条 `assert u.seq == expected_seq + 1`；缺口计数 > 0 即有丢失；乱序检测 `u.seq <= last_applied_seq`。

**fix**

```python
if u['U'] != last_u + 1:          # 缺口或乱序
    logger.warning("depth gap → resync"); resync(symbol)   # 丢弃本地簿、重拉 snapshot
    return
apply_update(book, u); last_u = u['u']
```

## 1.4 深度截断当完整簿（depth truncation）【high】

**why** —— 只订了有限档（`@depth5`/`@depth20`）或 REST `limit=5`，却把它当整本订单簿用来算深度、VWAP 冲击成本、大单可成交量——超出可见档的流动性被当成不存在，冲击成本/可成交量严重低估。

**code smell**

```python
book = rest.depth(symbol, limit=5)             # 只有 5 档
fillable = sum(lvl.qty for lvl in book.asks)   # 当成全部卖盘深度
```

**detect** —— 检查订阅档数 / REST `limit` 与用途是否匹配；用「截断深度」算冲击成本/大单可成交量即误用。grep `@depth5|@depth10|@depth20|limit=5`。

**fix** —— 需要全深度就用 diff-stream 维护完整本地簿（见 1.1）；否则显式只把它当「top-N 快照」算 best 价，不外推全簿流动性，并在文档标注深度上限。

## 1.5 陈旧盘口当实时（stale top-of-book）【high】

**why** —— WS 半开断连、行情线程卡住、symbol 不活跃，导致盘口长时间不更新，本地 `best_bid/ask` 仍是几秒/几分钟前的旧值。策略以为实时，实则在对一个过期价格报价。

**code smell**

```python
mid = (book.best_bid + book.best_ask) / 2    # 不看 book 最后更新时间
```

**detect** —— 给本地簿记 `last_update_ts`（单调时钟），`now - last_update_ts > 阈值` 即陈旧；结合心跳看门狗（见 connector-forge）。

**fix**

```python
if monotonic() - book.last_update_ts > STALE_SEC:
    raise BookStale(symbol)     # 标记不可交易、触发重连，绝不用陈旧盘口下单
```

## 1.6 单边枯竭 / 一档为空（one-side empty / liquidity exhaustion）【medium】

**why** —— 某一侧（买或卖）档位被吃空或推送残缺，`best_ask` 为空。代码若不处理 None，要么崩、要么用上一档残值算出离谱的 midprice/价差。

**code smell**

```python
mid = (book.bids[0].price + book.asks[0].price) / 2   # asks 为空时 IndexError 或用脏值
```

**detect** —— 取 best 档前是否判空；统计 `best_bid is None or best_ask is None` 的比例。

**fix** —— best 档为空时 midprice 返回 None 并标记该 symbol 暂不可报价，而非用残值硬算；区分「真单边枯竭」与「推送残缺」（后者触发重快照）。

## 1.7 价格档位精度 / tick-size 不对齐（float key / tick misalignment）【medium】

**why** —— 用浮点价格当订单簿档位 key，浮点误差（`0.1 + 0.2 != 0.3`）导致同一档被当两档、`qty==0` 删档因不相等而删不掉，本地簿出现幽灵档位、量价漂移。

**code smell**

```python
book[float(price)] = qty               # 浮点 key
if qty == 0: del book[float(price)]    # 因浮点不等而删不掉 → 幽灵档位
```

**detect** —— grep 订单簿是否用 `float(price)` 当 dict key；检查 `qty==0` 删档逻辑是否可靠命中。

**fix**

```python
# 价格转「整数 ticks」当 key，删档精确命中
key = round(price / tick_size)
if qty == 0: book.pop(key, None)
else:        book[key] = qty
```

---

# ② K线 / tick（OHLCV & Trades）

这一类的共同特征：**时间序列的连续性、唯一性、有序性、闭合性被破坏**。下游指标（MA/ATR/动量/VWAP）和回测默认输入是「干净、连续、有序、已闭合」的 bar/tick，一旦假设不成立，收益与波动率失真、信号错位、look-ahead 泄漏。统一防御口径：**入库前重建期望时间网格、以业务主键去重、强制按权威键排序、只采纳已闭合 bar。**

## 2.1 K线缺口 / 缺失 bar（kline gap）【high】

**why** —— 连续周期里某些时间桶没有 bar。下游指标（MA/ATR/动量）和回测会把缺口两端当成相邻 bar，导致收益/波动率失真，信号在缺口处错位。尤其在低流动性 symbol 或交易所维护窗口高发。

**code smell**

```python
# 反例：直接对拿到的 bar 序列做 diff/pct_change/rolling，从不检查时间索引连续性
bars['ret'] = bars['close'].pct_change()          # 缺口两端被当相邻 bar
bars['ma'] = bars['close'].rolling(20).mean()
plt.plot(range(len(bars)), bars['close'])         # 用 range(len) 做 x 轴而非真实时间戳
bars = bars.reindex(grid).ffill()                 # reindex 后默认 ffill 掩盖缺口
```

**detect** —— 对 `open_time` 排序后计算相邻差 `delta = ts[i+1] - ts[i]`，期望 `delta == interval_ms`，统计 `delta != interval_ms` 的位置；或用期望网格求差。区分整段缺（交易所停服）与零星缺。

```python
import pandas as pd

def find_kline_gaps(open_time_ms: pd.Series, interval_ms: int) -> pd.DataFrame:
    ts = open_time_ms.sort_values().reset_index(drop=True)
    delta = ts.diff()
    bad = delta[(delta.notna()) & (delta != interval_ms)]
    return pd.DataFrame({"gap_start": ts[bad.index - 1].values,
                         "gap_end": ts[bad.index].values,
                         "missing_bars": (bad / interval_ms - 1).astype("Int64").values})

# 期望网格法：缺的就是 gap
expected = pd.date_range(start, end, freq=interval)
missing = expected.difference(actual_index)
```

**fix** —— 显式重建期望时间网格并 reindex；缺口策略分类处理：停盘/维护期标记为 NaN 并在指标里 skip，而非盲目 ffill；回测时遇到大段缺口应拒绝该区间或拆段；记录每个 symbol 的 gap 报告供人工核查。

```python
grid = pd.date_range(start, end, freq=interval, tz="UTC")
bars = bars.reindex(grid)                          # 缺口处保留 NaN，不 ffill
# 指标在 NaN 上自动传播/skip，回测对大段缺口拆段或拒绝
gap_report = bars["close"].isna().sum()            # 落进数据质量报告
```

---

## 2.2 重复 bar（duplicate kline）【high】

**why** —— 同一 `open_time` 出现多条记录（WS 重推、REST 翻页边界重叠、补数与实时流交叠）。聚合时成交量被重复计入，VWAP/累积量虚高；去重不当还会让两条值不同的 bar 二选一引入随机性。

**code smell**

```python
# 反例：append 拼接多个数据源后不 dedup
df = pd.concat([hist, realtime])                   # 边界重叠未处理
df = df.set_index("open_time")                     # 不检查 index.is_unique
vol_total = df["volume"].sum()                     # 重复 bar 的量被重复累加
```

**detect** —— `df.duplicated(subset=['symbol','open_time']).sum() > 0`；或 `groupby(open_time).size().max() > 1`。进一步对比重复行的 OHLCV 是否一致——值不同说明是「部分更新 vs 终值」的混淆，比纯重复更危险。

```python
dups = df.duplicated(subset=["symbol", "open_time"], keep=False)
assert not dups.any(), f"{dups.sum()} duplicate klines"
# 值是否一致(纯重复) vs 不一致(部分更新混入终值)
conflict = (df[dups].groupby(["symbol", "open_time"])[["open","high","low","close","volume"]]
            .nunique().max(axis=1) > 1)
```

**fix** —— 对实时流以 `(symbol, interval, open_time)` 为主键 upsert，保留 `is_closed=true` 的终值版本；合并历史与实时用 `keep='last'` 并以闭合 bar 优先；入库加唯一约束 `(symbol, interval, open_time)` 让重复直接被拒。

```python
df = df.sort_values("is_closed")                   # 闭合版本排后
df = df.drop_duplicates(subset=["symbol", "interval", "open_time"], keep="last")
# DDL: UNIQUE (symbol, interval, open_time) —— 重复在入库层被拒
```

---

## 2.3 时间戳乱序（out-of-order kline timestamps）【high】

**why** —— bar 不是按 `open_time` 单调递增到达（乱序补包、多 WS 分片合流、并发写入）。任何 `rolling/shift/diff` 都假设时间有序，乱序会让指标计算出完全错误的相邻关系，且 look-ahead 检测失效。

**code smell**

```python
# 反例：拿到流式 bar 直接 append 进容器就算
bars = []
def on_kline(k): bars.append(k)                    # 假设网络保序
# 多线程各自 append 同一容器无排序
```

**detect** —— `assert df['open_time'].is_monotonic_increasing`；或 `(df['open_time'].diff() < 0).any()` 为 True 即乱序。流式场景维护 `last_ts`，新 `bar.open_time < last_ts` 立即告警。

```python
assert df["open_time"].is_monotonic_increasing, "out-of-order klines"

last_ts = -1
def on_kline(k):
    global last_ts
    if k["open_time"] < last_ts:
        log.warning("out-of-order bar: %s < %s", k["open_time"], last_ts)
    last_ts = max(last_ts, k["open_time"])
```

**fix** —— 入库前按 `open_time` 强制排序；流式用有序缓冲（按 ts 插入）或 watermark 机制延迟一个窗口再 emit；落库后建索引并在读取时 `ORDER BY open_time`；乱序到达的迟到 bar 走 upsert 而非 append。

```python
df = df.sort_values("open_time").reset_index(drop=True)
# 流式: 维护按 ts 排序的缓冲, watermark 延迟一个窗口再 emit, 迟到 bar upsert
```

---

## 2.4 interval 对齐错误（bar boundary misalignment）【high】

**why** —— bar 的 `open_time` 没对齐到 interval 边界（1h bar 的 `open_time` 出现 10:37 而非 10:00），常因本地重采样起点不对、时区偏移、或交易所与本地 epoch 基准不同。导致跨源对齐失败、resample 出半根 bar、聚合周期错位。

**code smell**

```python
# 反例：resample 不设 origin/offset
df.resample("1H").agg({"close": "last"})           # 默认 origin 可能不对齐 epoch
ts = ts.dt.floor("1H")                             # 用了本地时区的 floor
# 把 5m bar 当成精确 300000ms 整除, 但实际起点偏移
# 不同交易所日界(UTC vs 交易所本地)混用
```

**detect** —— 检查 `(open_time % interval_ms) == 0`（对 UTC 对齐的交易所）；1h/1d bar 还要确认是否 UTC 00:00 对齐还是交易所结算时间对齐。对比同 symbol 跨源同一周期 bar 的 `open_time` 是否逐根相等。

```python
off_grid = (df["open_time"] % interval_ms) != 0
assert not off_grid.any(), f"{off_grid.sum()} bars off interval grid"
# 跨源逐根比对
mismatch = (src_a["open_time"].values != src_b["open_time"].values)
```

**fix** —— 重采样显式指定 `origin='epoch'`（或交易所约定的 origin），并统一到 UTC；聚合前断言 `open_time` 落在 interval 网格上；跨交易所聚合先归一到同一日界定义；文档化每个交易所的 bar 对齐基准。

```python
df.resample("1H", origin="epoch").agg(...)         # 显式 origin, 对齐 epoch
# 1d bar 明确是 UTC 00:00 对齐还是交易所结算时间, 跨源前统一日界定义
```

---

## 2.5 使用未闭合 bar（unclosed / forming bar leakage）【fatal】

**why** —— 把当前仍在形成的 bar（`is_closed=false` / `k.x=false`）当成已完成 bar 喂给信号或入库。该 bar 的 close/high/low 会随后续 tick 变化——回测里这是典型未来函数，实盘会用一个「最终值未知」的 bar 触发交易，导致回测虚高、实盘对不上。

**code smell**

```python
# 反例：Binance WS kline 不判断 k.x 就用 k.c
def on_message(msg):
    k = msg["k"]
    close = k["c"]                                  # 没判断 k["x"](是否闭合)
    signal = compute(close)                         # bar 中途就用 close 触发信号
# 把 ticker/最新价直接当 bar close; 落库不区分 closed/forming
```

**detect** —— 检查数据流中是否过滤 `is_closed` / `x == true`；若某 symbol 的最后一根 bar 的 `close_time > now()` 说明它还没闭合却被使用；实盘 vs 回测同一信号触发时刻是否一致（实盘提前触发 = 用了未闭合 bar）。

```python
assert all(msg["k"]["x"] for msg in adopted_bars), "forming bar adopted as final"
unclosed = df[df["close_time"] > now_ms()]
assert unclosed.empty, "last bar not yet closed but used"
```

**fix** —— 只在 `k.x == true`（bar 闭合）时才采纳为正式 bar 并触发信号；实时展示可用 forming bar 但明确标记 `provisional` 且永不入信号/入库为终值；回测严格只用已闭合 bar 的收盘数据，信号在 bar close 后下一根开盘执行。

```python
def on_message(msg):
    k = msg["k"]
    if not k["x"]:
        update_provisional(k)                       # 仅展示, 不入信号/不入库为终值
        return
    finalize_bar(k)                                 # 闭合才采纳, 信号下一根开盘执行
```

---

## 2.6 未来时间戳（future timestamps）【high】

**why** —— bar/tick 的时间戳大于当前时间（本地时钟慢、交易所时钟快、单位换算错把 us 当 ms 放大、补包混入）。会污染「最新数据」判断，触发用未来数据计算，或让 dedup/排序逻辑误判。

**code smell**

```python
# 反例：信任服务器时间从不和本地比对
ts = msg["E"]                                       # 直接用, 无 sanity 上界
# ms/s 混算后时间被放大 1000 倍仍当合法
```

**detect** —— 断言 `ts <= now_ms + clock_skew_tolerance`（如 +5s）；统计 ts 超过 now 的记录数；对历史回放数据，任何 `ts > 文件生成时间` 都可疑。

```python
TOL_MS = 5_000
def sane_ts(ts_ms: int) -> bool:
    return ts_ms <= now_ms() + TOL_MS

future = df[df["open_time"] > now_ms() + TOL_MS]
assert future.empty, f"{len(future)} future-dated rows"
```

**fix** —— 入口加时间上界校验，超界的记录隔离到 quarantine 而非丢进主流；定期 NTP 校时并测交易所 serverTime 与本地差（Binance `/time`）；单位换算后再次跑上下界 sanity check。

```python
if not sane_ts(ts):
    quarantine.append(row)                          # 隔离而非污染主流
else:
    main_stream.append(row)
```

---

## 2.7 OHLC 不变式被破坏（OHLC invariant violation）【high】

**why** —— 正确的 bar 必须满足 `low <= min(open, close)` 且 `high >= max(open, close)` 且 `low <= high`。被破坏说明数据损坏、字段错位（o/h/l/c 列顺序映射反了）、或聚合 bug。下游一切基于 high/low 的逻辑（突破、ATR、止损回测）全错。

**code smell**

```python
# 反例：解析交易所数组 [t,o,h,l,c,v] 时索引写错
t, o, c, h, l, v = row                              # h/l 或 o/c 位置调换
# 自建重采样用 first/last/max/min 但聚合分组错
# 不同源字段命名不同直接按位拼
```

**detect** —— 向量化断言：`(low <= open) & (low <= close) & (high >= open) & (high >= close) & (high >= low)` 全为 True；统计违反行。还要查 `high == low` 但 `volume > 0` 的可疑零波动 bar。

```python
ok = ((df.low <= df.open) & (df.low <= df.close) &
      (df.high >= df.open) & (df.high >= df.close) & (df.high >= df.low))
assert ok.all(), f"{(~ok).sum()} OHLC invariant violations"
suspicious = df[(df.high == df.low) & (df.volume > 0)]   # 零波动却有量
```

**fix** —— 在数据校验层硬性 reject 违反不变式的 bar；字段映射用命名而非位置，并对每个交易所写一次性映射单测；自建聚合后立即跑不变式断言；违反的 bar 标记 `corrupt` 并触发重新拉取。

```python
# 命名映射, 不按位置
FIELDS = ["open_time", "open", "high", "low", "close", "volume"]
bar = dict(zip(FIELDS, row))
df.loc[~ok, "quality"] = "corrupt"                  # 标记并触发重拉
```

---

## 2.8 活跃 symbol 零成交量 bar（zero volume on active symbol）【medium】

**why** —— 主流活跃 symbol 出现 `volume == 0`（且常伴 `o == h == l == c`）的 bar，通常是交易所用上一根收盘价填充的「无成交占位 bar」，或数据源补缺时填的假 bar。当作真实 bar 会让波动率被低估、VWAP 失真、流动性判断错误。

**code smell**

```python
# 反例：对 volume 不做合理性检查
vwap = (df.close * df.volume).sum() / df.volume.sum()   # volume=0 拉平/除零
# 把占位 bar 当真实静默期, 波动率被低估
```

**detect** —— 筛 `volume == 0` 的 bar 占比；对活跃 symbol 该比例应极低，偏高说明数据源在填充。结合 `o == h == l == c` 且 `volume == 0` 强信号判定为占位/合成 bar。

```python
zero_rate = (df.volume == 0).mean()
synthetic = (df.volume == 0) & (df.open == df.high) & (df.high == df.low) & (df.low == df.close)
assert zero_rate < 0.01, f"zero-volume rate {zero_rate:.2%} too high for active symbol"
```

**fix** —— 区分「真实零成交」（冷门 symbol/盘后）与「填充占位」；活跃 symbol 的零量 bar 视为可疑，优先从另一源校验或标记 `synthetic`；计算量加权指标时排除合成 bar；记录零量率作为数据质量指标。

```python
df.loc[synthetic, "quality"] = "synthetic"
real = df[df["quality"] != "synthetic"]
vwap = (real.close * real.volume).sum() / real.volume.sum()   # 排除合成 bar
```

---

## 2.9 价格离群尖刺（price outlier spike / fat finger / bad print）【high】

**why** —— 单根 bar 或单笔 tick 出现远离邻域的极端价（乌龙指、坏报价、精度/小数点错位、闪崩瞬间未过滤）。会引爆基于极值的止损、污染滚动统计、让回测在不可成交的价位「成交」，严重虚增/虚减收益。

**code smell**

```python
# 反例：无任何离群过滤直接入库
df.to_sql(...)                                      # 坏报价直接进主表
# 用绝对阈值而非相对邻域判断; 回测在 high/low 极值点假设能成交
```

**detect** —— 相邻 bar return 的绝对值超过 N 倍滚动标准差（robust 用 MAD/中位数）即标记；或单 bar `high/low` 相对 `close` 偏离 > X%（按 symbol 波动率定阈）。tick 级：单笔价偏离前后中位数过大。

```python
import numpy as np
def hampel_outliers(x: pd.Series, window=20, n_sigma=5):
    med = x.rolling(window, center=True).median()
    mad = (x - med).abs().rolling(window, center=True).median()
    return (x - med).abs() > n_sigma * 1.4826 * mad   # MAD 转标准差

flag = hampel_outliers(df["close"])
```

**fix** —— 入口做 robust 离群检测（MAD/Hampel filter）并隔离待核；不直接删除而是标记，交叉用第二数据源确认是真行情（闪崩）还是坏报价；回测成交假设要保守（不假设吃到瞬时极值）；保留原始数据另存清洗后版本。

```python
df.loc[flag, "quality"] = "outlier"                 # 标记不删除
# 用第二数据源交叉确认: 真闪崩则保留, 坏报价则修正; 原始与清洗版分开存
```

---

## 2.10 tick / trade 乱序（out-of-order trades by ts / tradeId）【high】

**why** —— 逐笔成交未按 `transact_time` 或 `tradeId` 单调到达（多分片 WS、网络重排、补包穿插）。逐笔重建的 OHLC、累积 delta、订单流不平衡、撮合方向推断全部依赖严格时序，乱序会让重建结果错误且不可复现。

**code smell**

```python
# 反例：WS aggTrade/trade 到达即处理不缓冲
def on_trade(t): rebuild(t)                          # 假设到达即有序
# 只按 ts 排序却忽略同 ts 多笔需用 tradeId 二级排序
trades.sort_values("ts")                             # 同 ts 多笔顺序未定
```

**detect** —— 维护 `last_trade_id` / `last_ts`，新成交 `id <= last_id` 或 `ts < last_ts` 即乱序告警；离线对 trades 按 `(ts, tradeId)` 排序后与原始顺序对比逆序对数量。

```python
sorted_idx = trades.sort_values(["transact_time", "tradeId"]).index
inversions = (sorted_idx.values != trades.index.values).sum()
assert inversions == 0, f"{inversions} out-of-order trades"
```

**fix** —— 用 `(transact_time, tradeId)` 复合键排序，`tradeId` 为权威单调主键；流式加有序缓冲 + watermark 容忍小幅乱序后再 emit；重建逐笔指标前强制按权威键排序并断言单调。

```python
trades = trades.sort_values(["transact_time", "tradeId"]).reset_index(drop=True)
assert trades["tradeId"].is_monotonic_increasing
```

---

## 2.11 重复成交（duplicate trades）【high】

**why** —— 同一 `tradeId` 被处理多次（WS 重连后重放、REST 翻页 `fromId` 边界重叠、补数与实时交叠）。成交量/成交额/逐笔 delta 被重复累加，VWAP 与持仓推断偏差，回测换手率虚高。

**code smell**

```python
# 反例：翻页用 fromId 时把上一页最后一条又算进下一页
page = client.trades(fromId=last_id)                # 含 last_id 本身, 边界重叠
all_trades += page
vol += sum(t["qty"] for t in page)                  # 不以 tradeId 幂等, 重复累加
```

**detect** —— `trades.duplicated(subset='tradeId').sum() > 0`；或维护已见 `tradeId` 的集合/布隆过滤器，命中即重复。分页拉取时检查相邻页 id 区间是否重叠。

```python
dup = trades.duplicated(subset="tradeId")
assert not dup.any(), f"{dup.sum()} duplicate trades"
# 流式幂等
seen = set()
def ingest(t):
    if t["tradeId"] in seen: return                 # 幂等丢弃
    seen.add(t["tradeId"]); apply(t)
```

**fix** —— 以 `tradeId` 做幂等去重（集合/DB 唯一约束）；分页用严格开区间 `fromId = lastId + 1`；重连后用最后已确认 `tradeId` 续接而非全量重放；累加类计算保证幂等。

```python
page = client.trades(fromId=last_id + 1)            # 严格开区间, 不重叠
trades = trades.drop_duplicates(subset="tradeId", keep="first")
# DDL: UNIQUE (symbol, tradeId)
```

---

## 2.12 tradeId 缺口（trade sequence gap / dropped trades）【high】

**why** —— 连续 `tradeId` 出现跳号（WS 丢包、断线期间漏掉成交、限频被丢弃）。意味着部分成交永久缺失，逐笔重建的成交量与本地订单簿增量更新会偏离真实状态，订单簿失同步。

**code smell**

```python
# 反例：只消费 WS 不校验 id 连续性
def on_trade(t): apply(t)                            # 假设 WS 不丢包
# 断线重连不回补缺口区间
```

**detect** —— 对排序后 `tradeId` 计算 diff，期望恒为 1（连续交易所）；`diff > 1` 即缺口，缺口大小 = 漏掉笔数。统计每段缺口并定位时间窗。

```python
ids = trades["tradeId"].sort_values()
gap = ids.diff()
missing = gap[gap > 1]
total_dropped = (missing - 1).sum()
assert missing.empty, f"{int(total_dropped)} trades dropped across {len(missing)} gaps"
```

**fix** —— 检测到 id 缺口立即用 REST `historicalTrades`/`aggTrades` 按 id 区间回补；订单簿增量缺口则丢弃本地簿并重新拉 snapshot 对齐（参考 `connector-forge` 序列缺口重快照）；记录缺口率作为连接器健康指标。

```python
for _, row in missing.items():
    backfill = client.historical_trades(fromId=int(prev_id) + 1, limit=int(row) - 1)
    merge(backfill)
# 订单簿增量缺口: drop 本地簿 -> 重拉 depth snapshot -> 对齐 lastUpdateId
```

---

## 2.13 时间戳漂移与单调被破坏（timestamp drift / non-monotonic event time）【medium】

**why** —— 同一流内事件时间整体缓慢偏移（交易所时钟漂移）或本地接收时间与事件时间脱节，且偶发回退导致非单调。基于事件时间的窗口聚合、延迟测量、跨源对齐都会被慢慢拉偏，难以察觉。

**code smell**

```python
# 反例：混用本地接收时间和交易所事件时间
df["ts"] = time.time() * 1000                       # 用本地时间给事件打戳
window = df.resample("1min", on="ts")               # 实际混了 recv/event 两套时间
# 从不监控两者差值随时间变化
```

**detect** —— 持续记录 `local_recv_ms - event_ms` 的序列，其均值/趋势漂移即时钟问题；事件时间序列出现回退（非单调）单独告警；对比多交易所同一事件的 `event_ms` 偏移。

```python
lag = df["recv_ms"] - df["event_ms"]
drift = lag.rolling(1000).mean().diff().abs().max()     # 漂移趋势
assert df["event_ms"].is_monotonic_increasing, "non-monotonic event time"
assert drift < DRIFT_TOL_MS, f"clock drift detected: {drift}ms"
```

**fix** —— 明确区分并分别存储 `event_time` 与 `recv_time`，聚合用 `event_time`、延迟监控用差值；定期 NTP 校本地时钟、用交易所 serverTime 校漂移；非单调事件走有序缓冲；漂移超阈触发告警与重连。

```python
df = df[["event_ms", "recv_ms", "price", "qty"]]    # 两套时间分别存
agg = df.set_index(pd.to_datetime(df["event_ms"], unit="ms", utc=True)).resample("1min")
latency_p99 = (df["recv_ms"] - df["event_ms"]).quantile(0.99)
```

---

# ③ 通用时间戳对齐（Timestamp & Alignment）

这一类的共同特征：**跨字段、跨源、跨交易所做对齐时，时间单位 / 时区 / 时钟 / 符号 / universe 的口径不统一**。它们往往「数值合法」却静默错误，是最难抓也最致命的一类。统一防御口径：**入口归一到单一内部表示（epoch ms / ns + UTC tz-aware + 规范 symbol），任何换算封装在唯一函数里并跑范围 sanity check，跨源一律按时间戳 join 而非按行号 zip。**

## 3.1 时间戳单位混淆（ms / s / ns / us unit confusion）【fatal】

**why** —— 不同交易所/接口时间戳单位不同（秒 vs 毫秒 vs 微秒 vs 纳秒）。混用会让时间放大或缩小 10³~10⁹ 倍，bar 落到 1970 年或遥远未来，所有对齐/排序/窗口彻底错乱，且容易静默通过（数值仍是合法整数）。

**code smell**

```python
# 反例：硬编码换算散落各处
ts = msg["E"] * 1000                                # 到处 *1000 / /1000
dt = datetime.fromtimestamp(ts_ms)                  # 直接喂毫秒值, 落到遥远未来
pd.to_datetime(df["ts"])                            # 不指定 unit, 猜错单位
# 不同源拼接前不归一单位
```

**detect** —— 用量级判断：2020s 的秒级约 `1.7e9`、毫秒约 `1.7e12`、微秒约 `1.7e15`、纳秒约 `1.7e18`；把可疑 ts 转成日期看是否落在合理年份（1970/2286 是经典错单位症状）。

```python
def infer_unit(ts: int) -> str:
    digits = len(str(int(abs(ts))))
    return {10: "s", 13: "ms", 16: "us", 19: "ns"}.get(digits, "UNKNOWN")

# 转成年份做 sanity
yr = pd.to_datetime(ts, unit="ms", utc=True).year
assert 2009 <= yr <= 2100, f"suspicious year {yr}: wrong unit?"   # 1970/2286 = 错单位
```

**fix** —— 在数据入口统一归一为单一内部单位（建议 epoch ms 或 ns），并对每个数据源显式声明源单位；`pd.to_datetime` 永远带 `unit`；转换后跑年份范围 sanity check；封装单一 `to_internal_ms()` 避免散落换算。

```python
_FACTOR = {"s": 1000, "ms": 1, "us": 1e-3, "ns": 1e-6}
def to_internal_ms(ts: int, src_unit: str) -> int:
    ms = int(ts * _FACTOR[src_unit])                # 唯一换算入口
    assert 2009 <= pd.to_datetime(ms, unit="ms", utc=True).year <= 2100
    return ms
```

---

## 3.2 时区错误（timezone / naive-aware mismatch）【high】

**why** —— 把交易所 UTC 时间当本地时间、naive 与 aware datetime 混用、或按本地时区做日界切分。导致 bar 整体偏移数小时、日线日界错位、夏令时切换处出现重复或缺失小时，跨源对齐和「当日收益」统计全错。

**code smell**

```python
# 反例：naive 时间 + 本地时区
now = datetime.now()                                # 不带 tz
df["dt"] = pd.to_datetime(df["ts"], unit="ms")      # 不 tz_localize, naive
daily = df.resample("1D").last()                    # 按 server 本地时区切日界
# 字符串时间不带时区信息直接解析
```

**detect** —— 检查所有 datetime 是否 tz-aware（naive 即风险）；日线 bar 的日界是否对齐 UTC 00:00 还是预期结算时区；DST 切换日是否出现 23/25 小时异常。

```python
assert df["dt"].dt.tz is not None, "naive datetime — timezone risk"
# DST 异常: 某些日子小时数不是 24
hours_per_day = df.set_index("dt").resample("1D").size()
assert (hours_per_day == 24).all(), "DST anomaly: 23/25-hour day"
```

**fix** —— 全链路内部统一用 UTC tz-aware 时间，只在展示层转本地时区；明确每个交易所/品种的结算日界并文档化；禁止 naive datetime 入计算；DST 用 IANA tz 数据库正确处理。

```python
df["dt"] = pd.to_datetime(df["ts"], unit="ms", utc=True)   # 全程 UTC aware
display = df["dt"].dt.tz_convert("Asia/Shanghai")          # 只在展示层转
# 结算日界用 IANA tz, 不手算偏移
```

---

## 3.3 时钟漂移（clock skew between local and exchange）【high】

**why** —— 本地时钟与交易所时钟偏差。签名请求会因 `timestamp` 超出 `recvWindow` 被拒、延迟测量失真、「最新数据是否过期」判断错误。对低延迟实盘尤其致命——下单时间戳一旦偏移就直接被拒单。

**code smell**

```python
# 反例：下单 timestamp 直接用本地时间不做偏移补偿
params["timestamp"] = int(time.time() * 1000)       # 不补偿漂移
params["recvWindow"] = 60000                         # 设很大来"掩盖"漂移
# 从不测交易所 serverTime 差
```

**detect** —— 周期性请求交易所 `/time` 与本地比，`offset = serverTime - localTime`；offset 绝对值持续 > 阈值（如几百 ms）即漂移；签名请求 `-1021` timestamp 报错是典型症状。

```python
def measure_offset(client) -> int:
    t0 = now_ms(); server = client.time()["serverTime"]; t1 = now_ms()
    rtt = t1 - t0
    return server - (t0 + rtt // 2)                  # 扣掉半个 RTT

offset = measure_offset(client)
assert abs(offset) < 500, f"clock skew {offset}ms — orders may be rejected (-1021)"
```

**fix** —— 启动与周期性测量 offset 并在签名请求里补偿（`localTime + offset`）；本机跑 NTP/chrony 严格校时；`recvWindow` 保持合理而非靠放大掩盖；漂移超阈告警并暂停下单。

```python
params["timestamp"] = now_ms() + offset             # 补偿漂移
params["recvWindow"] = 5000                          # 合理值, 不靠放大掩盖
if abs(offset) > SKEW_ALERT: pause_trading()
```

---

## 3.4 跨交易所 / 跨源对齐错误（cross-exchange alignment）【high】

**why** —— 不同交易所同一「时刻」的 bar 因日界定义、bar 对齐基准、上市时间、停盘窗口不同而无法逐根对齐。直接做价差/套利/相关性会把错位的 bar 配对，产生虚假价差信号和错误对冲比。

**code smell**

```python
# 反例：两交易所 bar 按行号 zip 对齐
spread = a["close"].values - b["close"].values      # 按位置相减, 忽略各自缺口
for ka, kb in zip(a, b): ...                         # 一方缺 bar 即整体错位
# 无视上市/退市时间差
```

**detect** —— 按 UTC `open_time` 做 outer join 后统计只在一方存在的时间点数量；检查两序列长度不等、时间索引交集占比偏低；价差序列出现规律性脉冲常是对齐错位。

```python
m = a.merge(b, on="open_time", how="outer", indicator=True)
only_a = (m["_merge"] == "left_only").sum()
only_b = (m["_merge"] == "right_only").sum()
overlap = (m["_merge"] == "both").mean()
assert overlap > 0.95, f"low alignment overlap {overlap:.2%}; only_a={only_a} only_b={only_b}"
```

**fix** —— 统一归一到 UTC 同一 interval 网格，按时间戳 join 而非位置；只在双方都有闭合 bar 的交集上计算；一方缺 bar 时不强行配对（标记或剔除）；对齐前统一日界与 bar 起点定义。

```python
both = a.merge(b, on="open_time", how="inner", suffixes=("_a", "_b"))  # 仅交集
spread = both["close_a"] - both["close_b"]          # 时间戳对齐后再算
```

---

## 3.5 symbol 归一化错误（symbol normalization）【high】

**why** —— 同一标的在不同交易所/接口符号不同（`BTCUSDT` vs `BTC-USDT` vs `BTC/USDT` vs `XBTUSD`；现货 vs 永续 vs 季度合约；改名/重新上市）。归一化错误会把不同合约的数据混在一起，或把同一标的拆成两份，污染聚合与跨源对齐。

**code smell**

```python
# 反例：字符串拼接/裁剪硬编码符号转换
internal = sym.replace("-", "").replace("/", "")    # 硬编码裁剪
key = base + quote                                   # 不区分 spot/perp/futures
# 忽略交易所改名与 contract 重命名; USDT 与 USDC 本位混用同一 symbol
```

**detect** —— 建符号映射表后检查是否存在未映射或一对多/多对一冲突；同一「BTC」下出现来自不同合约类型的数据；符号集合在不同源间求交集异常偏小。

```python
unmapped = set(raw_symbols) - set(SYMBOL_MAP)
assert not unmapped, f"unmapped symbols: {unmapped}"
# 一对多/多对一冲突
collisions = [k for k, v in groupby_internal.items() if len({x.market for x in v}) > 1]
```

**fix** —— 建立交易所符号 ↔ 内部规范 symbol 的映射表（含市场类型/本位/合约到期），所有数据入口先归一；区分 spot/perp/dated 不同内部 key；符号改名维护别名历史；映射缺失时 fail-loud 而非静默归并。

```python
# 内部 key 含市场类型与本位, 不靠裁剪字符串
SYMBOL_MAP = {("binance", "BTCUSDT"): "BTC-USDT.PERP.BINANCE", ...}
def to_internal(exch, raw):
    try: return SYMBOL_MAP[(exch, raw)]
    except KeyError: raise ValueError(f"unmapped {exch}:{raw}")   # fail-loud
```

---

## 3.6 停盘缺数据 vs 真缺口混淆（halt / maintenance vs real gap）【medium】

**why** —— 数据为空既可能是交易所维护/合约下市/标的停牌（合法无数据），也可能是采集器丢数据（真缺口）。两者处理方式相反——前者应标记停盘不交易，后者应回补。混淆会导致在停盘期用错填充值或对真缺口视而不见。

**code smell**

```python
# 反例：对所有空洞统一 ffill 或统一报错
df = df.reindex(grid).ffill()                       # 停盘期被填充喂给策略
# 不维护交易所维护/合约生命周期日历
```

**detect** —— 把缺口时间窗与交易所维护公告/合约上市退市/标的停牌日历对照；缺口集中且全市场同时 = 维护，单 symbol 零星 = 采集问题。检查缺口期是否真有交易对状态变更。

```python
def classify_gap(gap_start, gap_end, symbol, maint_calendar):
    if any(w.covers(gap_start, gap_end) for w in maint_calendar.get(symbol, [])):
        return "halted"
    market_wide = count_symbols_with_gap(gap_start) > MARKET_WIDE_THRESHOLD
    return "maintenance" if market_wide else "collection_gap"   # 单 symbol 零星 = 采集问题
```

**fix** —— 维护交易所维护窗口与合约生命周期日历，缺口先归因：停盘则标 `halted` 不交易也不填充、真缺口则触发回补；两类在数据集中用不同标记位区分；策略层显式处理 `halted` 状态。

```python
kind = classify_gap(...)
if kind == "collection_gap":
    backfill(symbol, gap_start, gap_end)            # 真缺口回补
else:
    df.loc[mask, "status"] = "halted"               # 停盘标记, 不填充, 策略层 skip
```

---

## 3.7 universe 幸存者偏差（survivorship bias in symbol universe）【fatal】

**why** —— 回测/选股的 symbol universe 只用「当前还在交易」的标的，剔除了已退市/下架/爆雷的合约。这是经典幸存者偏差，系统性高估收益（亏到退市的全被排除），让策略在历史上看起来稳赚而实盘失效。

**code smell**

```python
# 反例：用今天的活跃列表跑历史回测
universe = client.exchange_info()["symbols"]        # 当前活跃 symbol
universe = [s for s in universe if s["status"] == "TRADING"]
backtest(universe, start="2018-01-01")              # 历史上已退市的全被排除
# 按当前市值/成交量倒排选 top-N 跑历史; DB 只存仍在交易的 symbol
```

**detect** —— 检查回测 universe 是否随时间变化（point-in-time）还是固定为今天的列表；universe 里是否包含历史上已退市的 symbol；若所有回测标的至今仍存活，几乎必有幸存者偏差。

```python
delisted = {s for s in historical_symbols if s not in current_symbols}
assert delisted & set(backtest_universe), "universe excludes all delisted — survivorship bias"
# point-in-time: 各时点成分应不同
assert universe_at("2018-01-01") != universe_at("2024-01-01"), "static universe"
```

**fix** —— 使用 point-in-time universe——每个历史时点的成分由当时实际可交易标的决定，包含后来退市的；保留退市/下架 symbol 的历史数据与退市时间；选样按当时（而非当下）的流动性/市值；退市标的在退市点按规则平仓而非剔除。

```python
def universe_at(ts):
    return [s for s in all_symbols                  # 含已退市
            if s.listed <= ts and (s.delisted is None or s.delisted > ts)]

# 退市标的在 s.delisted 点按规则平仓, 不从历史样本剔除
```

---

## 怎么用这份清单

1. 按 `SKILL.md` 的**审查协议**定位数据加载/清洗/对齐代码区，对照三大类逐条用上面的 `detect` 断言跑一遍。
2. 每条都要**区分真坑与合法用法**——零成交量可能是冷门 symbol 的真静默，`shift(-1)` 在构造 label 时合法；命中只是嫌疑，定位到 `文件:行` 读懂语义再定罪。
3. 把所有 `detect` 断言固化成**入库前的数据质量门禁**（gap / dup / 乱序 / OHLC 不变式 / 单位年份 / 未来戳 / 闭合性），不达标的批次隔离到 quarantine 而非进主流。
4. 按严重度汇总进**数据体检报告**：致命（未闭合 bar、单位混淆、幸存者偏差）优先；**致命项清零前，不建议把这批数据喂进信号或回测。**
5. 订单簿/盘口类（① 类目）的连接器侧可靠性（重连、心跳、序列缺口重快照、本地簿对齐）配合姊妹 skill `connector-forge` 一起用。

> 本文件随实战补充。① 订单簿/盘口类目待展开；发现新坑、误报或交易所特定模式，欢迎提 PR（见仓库 README 的贡献指南）。