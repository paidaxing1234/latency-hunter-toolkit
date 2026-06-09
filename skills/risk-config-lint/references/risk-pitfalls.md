# 风控缺陷百科（Risk Config & Pitfalls Encyclopedia）

> SKILL.md 指向的「完整风控缺陷百科」。`risk-config-lint` 用它逐条审查实盘风控配置与下单链路的工程缺陷。两大类共 30 条，逐条给出：为什么坑（why）→ 配置/代码长啥样（code smell，反例）→ 怎么检测（detect）→ 怎么修（fix，正确配置/校验片段）。代码以 Python / JSON 为主。
>
> **严重度标记**：`【fatal】` 致命，一次事故即可爆仓/穿仓/资金失窃，必须阻断上线；`【high】` 高危，系统性放大风险敞口或让风控静默失效；`【medium】` 中危，应急/复位流程缺陷，事故时贻误战机或反复触发。
>
> **贯穿全篇的第一性原理**：所有硬限额必须在订单离开本机前**同步、阻断式**拦截（pre-trade gate）；任何风控失败一律 **fail-closed**（拒单/停机），绝不 fail-open（无限额继续下单）。

---

## 总览速查表

| # | 缺陷 | 严重度 | 类别 |
|---|------|--------|------|
| 1.1 | 缺失单笔下单上限（max order notional/qty） | fatal | ① 限额/回撤/杠杆/kill-switch/防胖手指 |
| 1.2 | 缺失总敞口/组合净敞口上限（gross/net cap） | fatal | ① |
| 1.3 | 缺失单品种上限（per-symbol position cap） | high | ① |
| 1.4 | 缺失集中度限制（single-name/sector concentration） | high | ① |
| 1.5 | 缺失日内/最大回撤止损（daily loss & max drawdown） | fatal | ① |
| 1.6 | 回撤口径错误（peak vs initial / 含浮亏） | high | ① |
| 1.7 | 无自动平仓 / 触发后仍需人工干预 | fatal | ① |
| 1.8 | 无界杠杆 / 无杠杆上限校验 | fatal | ① |
| 1.9 | 无单品种杠杆上限（per-symbol leverage cap） | high | ① |
| 1.10 | 缺失全局 kill-switch | fatal | ① |
| 1.11 | kill-switch 无自动触发条件（仅手动） | high | ① |
| 1.12 | kill-switch 触发后无人工 override / 无安全复位 | medium | ① |
| 1.13 | 无每秒最大下单数限制（max order rate） | high | ① |
| 1.14 | 无最大挂单数 / 在途订单数限制 | high | ① |
| 1.15 | 无 fat-finger 价格边界（price sanity vs mark） | fatal | ① |
| 1.16 | 无 fat-finger 数量边界 + 缺最小/最大单量 | high | ① |
| 1.17 | 风控只在 post-trade 拦截，缺 pre-trade gate | fatal | ① |
| 2.1 | 无维持保证金检查（MMR not validated） | fatal | ② 保证金/爆仓/单点/运维/密钥 |
| 2.2 | 无爆仓/强平模拟（缺强平价与压力测试） | fatal | ② |
| 2.3 | 允许负现金 / 负可用余额（无透支拦截） | fatal | ② |
| 2.4 | 告警只走单一渠道（单点告警） | high | ② |
| 2.5 | 无心跳 / deadman switch | fatal | ② |
| 2.6 | 配置无 schema 校验 | high | ② |
| 2.7 | 密钥写进风控配置（明文 API key/secret） | fatal | ② |
| 2.8 | 风控限额未在下单前真正强制执行（摆设） | fatal | ② |
| 2.9 | testnet / live 标志缺失或混淆 | fatal | ② |
| 2.10 | 环境未隔离（测试/实盘共用 DB/Redis/账户） | high | ② |
| 2.11 | 限额硬编码不可热加载 | high | ② |
| 2.12 | 限额加载失败时 fail-open | fatal | ② |
| 2.13 | 风控异常被吞掉静默失效（try/except pass） | fatal | ② |
| 2.14 | 多策略/多账户共享限额未隔离（连坐爆仓） | high | ② |

---

# ① 限额 / 回撤 / 杠杆 / kill-switch / 防胖手指

这一类的共同特征：**把「最多能下多大、最多能亏多少、最多几倍杠杆、出事怎么一键停」变成订单离开本机前的硬约束**。任何一条缺失，都意味着策略 bug、脏数据或异常行情可以无阻碍地把风险打满。统一防御口径：**所有硬限额在 pre-trade gate 同步拦截，任一不过即拒单；触发熔断必须程序化闭环执行，不依赖「人正好在线」。**

## 1.1 缺失单笔下单上限（max order notional/qty）【fatal】

**why** —— 没有单笔上限时，一个策略 bug、错误的信号缩放、或者反序列化出的脏数据可以瞬间打出远超账户净值的巨单，直接爆仓或触发交易所强平。这是 fat-finger 类事故最直接的入口。

**code smell**

```python
# 反例：直接照信号下单，无任何单笔上限
def place_order(symbol, signal_qty, price):
    order = Order(symbol, qty=signal_qty, price=price)
    return exchange.submit(order)  # signal_qty 可能是 NaN/巨大值
```

```yaml
# 反例：配置里压根没有 max_order_* 字段
strategy:
  symbol: BTCUSDT
  leverage: 10
```

**detect** —— 在下单路径/订单构造函数里搜 `quantity`、`size`、`notional` 的赋值，确认提交前是否有 `if order.notional > MAX_ORDER_NOTIONAL: reject`。配置里查找 `max_order_notional` / `max_order_qty` 是否存在且非 None/0/inf。grep `place_order|submit_order|new_order` 看是否有上限校验分支。

**fix** —— 在 pre-trade 强制校验单笔上限，按 notional（价格×数量）而非纯数量判定，并对 NaN/负数/inf 一并拒单：

```python
MAX_ORDER_NOTIONAL = Decimal('50000')  # USD，按账户规模配置

def validate_order(order, price):
    notional = abs(order.qty) * price
    if not notional.is_finite() or notional <= 0:
        raise RejectOrder(f'invalid notional {notional}')
    if notional > MAX_ORDER_NOTIONAL:
        raise RejectOrder(f'order notional {notional} > cap {MAX_ORDER_NOTIONAL}')
```

上限应来自集中配置并随账户净值动态缩放，禁止硬编码在散落的策略里。

## 1.2 缺失总敞口/组合净敞口上限（portfolio gross/net exposure cap）【fatal】

**why** —— 单笔有上限不代表总敞口安全。多策略/多品种叠加后，总名义敞口可能数倍于净值。市场一根大阴线就能击穿维持保证金。没有组合层面的 gross/net 上限，等于把多策略风险叠加成了系统性风险。

**code smell**

```python
# 反例：每个策略独立风控，从不汇总组合总敞口
class Strategy:
    def on_signal(self, sig):
        if self.local_position < self.local_limit:  # 只看自己
            self.place_order(...)
# 10 个策略 × 各自满仓 = 组合 10x 杠杆，无人察觉
```

**detect** —— 查找是否有一个聚合所有持仓+在途订单的 portfolio exposure 计算，并在下单前用 `projected_gross > MAX_GROSS` 拦截。若每个策略各自为政、没有全局 position aggregator，即为缺陷。检查 gross（绝对值求和）和 net（带方向求和）是否都有上限。

**fix** —— 引入全局 PortfolioRiskManager，下单前模拟成交后的组合 gross/net 敞口：

```python
MAX_GROSS_EXPOSURE = equity * Decimal('3')   # 总名义≤3倍净值
MAX_NET_EXPOSURE   = equity * Decimal('1.5')

def pre_trade_check(new_order, positions, open_orders, equity):
    proj = project_exposure(positions, open_orders, new_order)
    if proj.gross > MAX_GROSS_EXPOSURE:
        raise RejectOrder('gross exposure cap breached')
    if abs(proj.net) > MAX_NET_EXPOSURE:
        raise RejectOrder('net exposure cap breached')
```

敞口口径要包含未成交挂单（in-flight），否则可被连续下单绕过。

## 1.3 缺失单品种上限（per-symbol position cap）【high】

**why** —— 组合总敞口达标，但全部押在一个流动性差的山寨币上，照样会因滑点/插针被打穿，且无法平仓。单品种上限是集中度风控的第一道闸。缺失它，组合上限会被单一品种吃满。

**code smell**

```python
# 反例：所有品种共用一个全局上限，山寨币和 BTC 一视同仁
GLOBAL_CAP = 100000
if position_notional[symbol] < GLOBAL_CAP:  # SHIBUSDT 也能上 10万
    place_order(...)
```

**detect** —— 查找 per-symbol 的 position cap（按名义或按该品种 ADV/盘口深度的百分比）。若只有全局上限、没有 `position[symbol] > per_symbol_cap[symbol]` 检查，即缺失。重点看流动性差的小币种是否和 BTC 用同一个上限。

**fix** —— 按品种分级配置上限，最好挂钩流动性（如不超过该品种日均成交额或盘口 N 档深度的一定比例）：

```python
PER_SYMBOL_CAP = {
    'BTCUSDT': 200000,
    'ETHUSDT': 150000,
    '*':       20000,   # 未列出品种的保守默认
}
def symbol_cap(sym):
    return PER_SYMBOL_CAP.get(sym, PER_SYMBOL_CAP['*'])

if proj_position_notional(sym) > symbol_cap(sym):
    raise RejectOrder(f'{sym} per-symbol cap breached')
```

关键：未知品种必须 fall back 到保守默认值，而非无上限。

## 1.4 缺失集中度限制（single-name / sector concentration limit）【high】

**why** —— 即便每个品种都在自己上限内，组合也可能 80% 集中在一个币/一个赛道（如全是 L1 公链币）。相关性极高时一起暴跌，分散化是假象。缺失集中度比例限制，风险敞口会悄悄聚集。

**code smell**

```python
# 反例：只有绝对额风控，无相对集中度
# 净值 100k，结果 95k 全压 BTC，剩 5k 分 9 个币——“分散”是假的
for sym in signals:
    if notional[sym] < PER_SYMBOL_CAP[sym]:
        place_order(sym, ...)
```

**detect** —— 查找是否计算 `position[symbol] / total_portfolio` 的占比并设上限，以及是否有按 sector/cluster 的聚合占比限制。只有绝对额上限、没有相对占比上限，即为缺陷。

**fix** —— 增加相对占比与赛道聚合上限：

```python
MAX_SINGLE_NAME_PCT = Decimal('0.25')   # 单品种≤组合25%
MAX_SECTOR_PCT      = Decimal('0.40')   # 单赛道≤40%

def concentration_check(sym, proj_notional, gross, sector_of):
    if gross > 0 and proj_notional / gross > MAX_SINGLE_NAME_PCT:
        raise RejectOrder(f'{sym} single-name concentration breach')
    sector_notional = sum(n for s, n in positions if sector_of[s] == sector_of[sym])
    if gross > 0 and sector_notional / gross > MAX_SECTOR_PCT:
        raise RejectOrder('sector concentration breach')
```

## 1.5 缺失日内/最大回撤止损（daily loss & max drawdown stop）【fatal】

**why** —— 没有回撤熔断，策略在异常行情或自身失效时会持续放大亏损直到爆仓。日内止损（DLL）和最大回撤止损是把「今天最多亏多少」和「历史峰值回撤多少就停手」变成硬约束的最后保险，缺失即等于无底洞。

**code smell**

```python
# 反例：只有单笔策略止损，没有账户级日内/回撤熔断
def trading_loop():
    while True:
        sig = get_signal()
        place_order(sig)   # 不管今天已经亏了多少，照下不误
```

**detect** —— 查找是否周期性计算当日 realized+unrealized PnL 并与 daily_loss_limit 比较、触发后停止开新仓/平仓。查找是否跟踪 equity peak 并计算回撤百分比。若只在回测里有止损、实盘下单循环里没有账户级回撤检查，即为缺陷。

**fix** —— 账户级双闸：日内亏损限额 + 最大回撤限额，触发即进入只减仓/停机状态：

```python
DAILY_LOSS_LIMIT = equity_sod * Decimal('0.03')   # 当日≤亏3%
MAX_DRAWDOWN_PCT = Decimal('0.15')                # 自峰值回撤≤15%

def risk_gate(state):
    if state.daily_pnl <= -DAILY_LOSS_LIMIT:
        state.halt('daily loss limit hit'); return False
    dd = (state.equity_peak - state.equity) / state.equity_peak
    if dd >= MAX_DRAWDOWN_PCT:
        state.halt('max drawdown hit'); return False
    return True
```

触发后应只允许减仓单通过，且需人工/冷却后才能复位。

## 1.6 回撤口径错误：peak vs initial / 已实现 vs 含浮亏【high】

**why** —— 回撤算法把分母用错（拿初始资金而非历史峰值），或只算 realized 不含 unrealized 浮亏，会让止损形同虚设——账户已从峰值深跌，但因为还在初始资金之上、或浮亏没计入，熔断永远不触发。这是「有止损却没保护」的隐形 bug。

**code smell**

```python
# 反例1：用初始资金当分母，赚过又亏回来检测不到
dd = (initial_capital - equity) / initial_capital
# 反例2：equity 只含已实现，浮亏不算
equity = cash + realized_pnl            # 漏掉 unrealized
# 反例3：重启后 peak 被重置
self.equity_peak = self.equity          # 启动即归零，回撤永远=0
```

**detect** —— 审查回撤公式：分母必须是滚动历史 equity peak 而非 initial_capital；equity 必须是 mark-to-market（含未实现盈亏与资金费），而非仅已实现。检查 equity_peak 是否随新高单调更新、是否在重启后从持久化状态恢复（否则重启把 peak 重置回当前值，回撤瞬间归零）。

**fix** —— 用 mark-to-market equity，peak 取滚动历史最大值并持久化：

```python
def update_equity(state, cash, positions, marks, funding):
    state.equity = cash + sum(p.qty * marks[p.sym] for p in positions) \
                   + state.realized_pnl - funding  # 含浮亏
    if state.equity > state.equity_peak:
        state.equity_peak = state.equity
        persist(state.equity_peak)               # 落盘，重启可恢复
    state.drawdown = (state.equity_peak - state.equity) / state.equity_peak
```

口径统一：所有回撤/止损判定都基于同一个 mark-to-market equity 源。

## 1.7 无自动平仓 / 风控触发后仍需人工干预【fatal】

**why** —— 回撤/止损只发告警、不自动平仓，等于把救命操作押在「人正好在线且反应够快」上。半夜插针、网络抖动、人在睡觉时，告警无人响应，损失继续扩大直到爆仓。风控的执行必须是程序闭环。

**code smell**

```python
# 反例：触发只告警，平仓全靠人
if drawdown >= MAX_DD:
    logger.error('DRAWDOWN BREACH!')
    send_telegram('快平仓！')   # 人在睡觉……
```

**detect** —— 查找止损/熔断触发后的动作：是 `send_alert()` 还是 `flatten_positions()`？若触发路径终点只是日志/通知，没有真正的市价/IOC 平仓 + 撤单逻辑，即为缺陷。检查平仓是否处理部分成交、撤单失败、平仓单本身被拒的重试。

**fix** —— 触发即程序化平仓，撤掉所有挂单并用激进单型清仓，带重试与确认：

```python
def emergency_flatten(state, exchange):
    exchange.cancel_all_orders()                 # 先撤挂单防新增敞口
    for pos in state.positions:
        side = 'SELL' if pos.qty > 0 else 'BUY'
        for attempt in range(MAX_RETRY):
            r = exchange.submit_reduce_only(pos.sym, side,
                                            abs(pos.qty), type='IOC')
            if confirmed_filled(r): break
            time.sleep(BACKOFF)
    state.enter_safe_mode()                       # 平完进只读/停机
    send_alert('AUTO-FLATTEN executed')          # 告警是事后通知，不是手段
```

reduce_only 保证平仓单不会反向加仓。

## 1.8 无界杠杆 / 无杠杆上限校验【fatal】

**why** —— 杠杆没有上限，或上限只配在交易所端而代码层不校验，一旦策略请求 50x/100x，维持保证金极薄，市场轻微逆行就强平。杠杆是放大爆仓速度的乘数，必须在本地 pre-trade 有硬上限。

**code smell**

```python
# 反例：杠杆从配置/信号直接透传，无校验
lev = config.get('leverage', 20)
exchange.set_leverage(symbol, lev)   # 配置写 100 就 100x
```

**detect** —— 搜索 `leverage` / `set_leverage` 调用，确认是否有本地上限常量校验，而不是把任意值直接透传交易所。检查「有效杠杆 = gross 敞口 / equity」是否被持续监控并设上限，而非只看下单时的名义 leverage 参数。

**fix** —— 本地硬上限 + 监控有效杠杆：

```python
MAX_LEVERAGE = 10

def set_leverage_safe(exchange, sym, requested):
    lev = min(int(requested), MAX_LEVERAGE)
    if requested > MAX_LEVERAGE:
        logger.warning(f'leverage {requested} capped to {lev}')
    exchange.set_leverage(sym, lev)

def effective_leverage(gross_notional, equity):
    return gross_notional / equity if equity > 0 else float('inf')
# 持续校验：effective_leverage 超阈值即减仓
```

## 1.9 无单品种杠杆上限（per-symbol leverage cap）【high】

**why** —— 全局杠杆上限合理，但对波动率/流动性差异巨大的品种一刀切。给小币种和 BTC 同样的 10x，小币种一根插针就强平。杠杆上限必须按品种风险分级。

**code smell**

```python
# 反例：所有品种统一 10x
for sym in symbols:
    exchange.set_leverage(sym, 10)   # DOGE 和 BTC 同样 10x
```

**detect** —— 查找杠杆配置是否区分品种。若 set_leverage 对所有 symbol 用同一个常量、没有按品种波动率/流动性分级，即缺陷。对照交易所对该品种的最大允许杠杆与风险分层（tiered margin）是否被尊重。

**fix** —— 按品种分级，未知品种用最保守值：

```python
PER_SYMBOL_MAX_LEV = {
    'BTCUSDT': 10,
    'ETHUSDT': 8,
    '*':       3,    # 长尾品种保守
}
def max_lev(sym):
    return PER_SYMBOL_MAX_LEV.get(sym, PER_SYMBOL_MAX_LEV['*'])

exchange.set_leverage(sym, min(requested, max_lev(sym)))
```

理想做法是按品种 ATR/历史波动率动态推导上限。

## 1.10 缺失全局 kill-switch【fatal】

**why** —— 出现系统性异常（行情源中断、连接器疯狂重连、风控指标乱跳、发现策略逻辑 bug）时，必须有一个开关能在毫秒级停掉所有下单并撤单。没有全局 kill-switch，事故发生时只能逐个停进程/拔网线，期间损失不可控。

**code smell**

```python
# 反例：下单路径没有任何全局开关检查
def submit_order(order):
    return exchange.send(order)   # 想停？只能 kill -9 进程
```

**detect** —— 查找下单路径入口是否检查一个全局开关（如 Redis key、原子标志、文件信号量）。若 submit_order 不读任何全局 `trading_enabled` 状态，即无 kill-switch。确认 kill-switch 同时阻止新单并触发撤单。

**fix** —— 在所有下单入口前置一个低延迟全局开关（Redis/共享内存原子标志），并提供一键触发：

```python
def submit_order(order, kill):
    if kill.is_active():          # 读 Redis/atomic flag
        raise TradingHalted('kill-switch active')
    return exchange.send(order)

def trigger_kill_switch(reason, exchange, kill):
    kill.activate(reason)         # 立即阻断所有新单
    exchange.cancel_all_orders()  # 撤掉在途挂单
    log.critical(f'KILL-SWITCH: {reason}')
```

kill-switch 读取要在热路径上极快，且对所有进程/策略全局可见。

## 1.11 kill-switch 无自动触发条件（仅手动）【high】

**why** —— kill-switch 存在但只能人工拉闸，等于把「何时该停」交给人的反应速度。行情源静默、订单拒单率飙升、PnL 异常跳变这类系统性故障往往几秒内造成大损失，靠人发现已经晚了。必须有自动触发条件。

**code smell**

```python
# 反例：kill-switch 只有手动接口
@app.post('/admin/kill')
def kill():
    kill_switch.activate('manual')
# 行情源已经断了 30 秒，无人知晓，仓位裸奔
```

**detect** —— 查找是否有监控线程根据指标（行情心跳超时、订单错误率、撤单失败率、PnL 偏离、持仓与交易所对账不一致）自动触发 kill-switch。若 kill-switch 只有一个人工 HTTP 端点、没有任何自动判据，即为缺陷。

**fix** —— 增加自动触发监控，多条独立判据任一命中即拉闸：

```python
def risk_monitor(state, kill):
    if now() - state.last_md_tick > MD_STALE_SEC:
        kill.activate('market data stale')
    if state.order_reject_rate(window=60) > 0.3:
        kill.activate('order reject rate spike')
    if abs(state.pnl_jump_1s) > PNL_JUMP_LIMIT:
        kill.activate('abnormal pnl jump')
    if state.position_reconcile_mismatch:
        kill.activate('position mismatch vs exchange')
```

自动触发与手动触发共用同一执行路径（撤单+停机）。

## 1.12 kill-switch 触发后无人工 override / 无安全复位流程【medium】

**why** —— 两个方向都危险：一是 kill-switch 没有人工急停 override（只能等自动条件），二是触发后能被自动/随意复位重新开仓。前者贻误战机，后者更致命——故障根因没排除就被自动重启，反复触发反复亏损（震荡熔断）。复位必须受控。

**code smell**

```python
# 反例：冷却结束自动复位，根因没排查
if kill.active and now() - kill.ts > COOLDOWN:
    kill.deactivate()   # 自动重开 → 再次触发 → 循环亏损
```

**detect** —— 查找 kill-switch 的复位路径：是否要求人工确认 + 系统健康校验才能恢复交易，还是冷却时间一到就自动重新开仓。检查是否有「触发计数」，短时间多次触发应升级为强制人工介入。同时确认存在独立于自动逻辑的人工急停入口。

**fix** —— 复位需人工确认 + 健康检查，多次触发升级锁死，并提供独立人工急停：

```python
def request_resume(operator, state, kill):
    if not kill.active:
        return
    if kill.trip_count_today >= MAX_TRIPS:
        raise NeedsManualReview('too many trips, lock until reviewed')
    if not state.health_ok():
        raise NotHealthy('resolve root cause before resume')
    if not operator.confirmed:
        raise NeedsConfirmation('human override required')
    kill.deactivate(by=operator.id)

# 独立人工急停，永远可用，不依赖自动逻辑
@app.post('/admin/panic')
def panic(): kill.activate('manual panic')
```

## 1.13 无每秒最大下单数限制（max order rate / throttle）【high】

**why** —— 策略 bug（如循环里反复下单）、消息风暴或重连后补发，会在一秒内打出成百上千单，轻则触发交易所限频被封 IP/降权，重则在限频前已建立大量意外敞口。本地令牌桶限频是防止「下单洪水」的闸。

**code smell**

```python
# 反例：信号一来就连发，无任何节流
for sig in signals:        # 一个 tick 可能几百个信号
    exchange.submit(build_order(sig))   # 瞬间打爆限频
```

**detect** —— 查找下单前是否过本地速率限制器（令牌桶/漏桶/滑动窗口计数）。若直接循环 submit、没有 rate limiter，即缺陷。确认限频是 per-account 且涵盖所有策略汇总，而不仅是单策略内部。

**fix** —— 全账户共享的令牌桶限频，超限则排队或拒单：

```python
class TokenBucket:
    def __init__(self, rate, burst):
        self.rate, self.tokens, self.cap = rate, burst, burst
        self.ts = time.monotonic()
    def allow(self):
        now = time.monotonic()
        self.tokens = min(self.cap, self.tokens + (now - self.ts) * self.rate)
        self.ts = now
        if self.tokens >= 1:
            self.tokens -= 1; return True
        return False

order_bucket = TokenBucket(rate=20, burst=40)  # ≤20单/秒

def submit_order(order):
    if not order_bucket.allow():
        raise RateLimited('order rate cap')
    return exchange.send(order)
```

限频阈值应低于交易所硬限，留余量给撤单/查询权重。

## 1.14 无最大挂单数 / 在途订单数限制（max open orders）【high】

**why** —— 做市/网格类策略若不限制挂单总数，重连后重复挂单、撤单失败堆积，会让在途订单无限增长。即使每笔很小，挂单总名义敞口可能巨大；交易所一侧也有挂单数上限，超了直接拒单导致策略瘫痪。

**code smell**

```python
# 反例：不断挂单，从不统计在途总量
def refresh_quotes(levels):
    for lvl in levels:
        exchange.place_limit(...)   # 旧单没撤干净就叠新单
```

**detect** —— 查找是否跟踪当前 open orders 数量与总挂单名义，并在挂新单前校验上限。若挂单后不登记、撤单后不注销、没有 `len(open_orders) < MAX_OPEN_ORDERS` 检查，即缺陷。重点看重连/重订阅后是否会重复挂单。

**fix** —— 维护在途订单台账，挂单前校验数量与总名义双上限：

```python
MAX_OPEN_ORDERS   = 50
MAX_OPEN_NOTIONAL = 200000

def can_place(order, price, book):
    if len(book.open_orders) >= MAX_OPEN_ORDERS:
        return False, 'too many open orders'
    proj = book.open_notional + abs(order.qty) * price
    if proj > MAX_OPEN_NOTIONAL:
        return False, 'open notional cap'
    return True, ''
```

挂单/成交/撤单都要及时更新台账，并定期与交易所 openOrders 对账纠偏。

## 1.15 无 fat-finger 价格边界（price sanity vs mark/last）【fatal】

**why** —— 限价单价格离市场过远（多打个 0、价格取反、用了陈旧/错误的盘口）会以荒谬价格成交，瞬间巨亏；市价单在薄盘里更会吃穿整个订单簿。没有「下单价 vs 参考价偏离上限」的边界校验，一次胖手指就能毁账户。

**code smell**

```python
# 反例：限价直接用信号给的价，无偏离校验
order = LimitOrder(symbol, qty, price=signal.price)  # 多打个0也照下
exchange.submit(order)
```

**detect** —— 查找下单前是否用 mark/last/盘口中间价做参考、校验 `abs(price/ref - 1) <= MAX_DEVIATION`。若价格直接来自信号/配置不做合理性检查，即缺陷。检查参考价本身是否新鲜（陈旧 mark 会让边界失效）。

**fix** —— 用新鲜参考价做偏离边界，超界拒单；市价单加保护限价：

```python
MAX_PRICE_DEVIATION = Decimal('0.05')   # 偏离参考价≤5%
REF_STALE_MS = 1000

def price_sanity(order, ref_price, ref_ts):
    if now_ms() - ref_ts > REF_STALE_MS:
        raise RejectOrder('reference price stale')
    if ref_price <= 0:
        raise RejectOrder('invalid reference price')
    dev = abs(order.price / ref_price - 1)
    if dev > MAX_PRICE_DEVIATION:
        raise RejectOrder(f'fat-finger price dev {dev:.2%}')
```

市价单改用带保护价的 IOC 限价，避免吃穿薄盘。

## 1.16 无 fat-finger 数量/名义边界 + 缺最小/最大单量（notional sanity）【high】

**why** —— 数量层面胖手指：单位错误（张 vs 币 vs USDT）、信号未做 round、NaN/负数/inf 漏过校验，会下出离谱数量。同时缺最小单量会被交易所拒（minNotional/stepSize/lotSize）导致策略卡死，缺最大单量则放大单笔风险。数量边界与价格边界同等重要。

**code smell**

```python
# 反例：数量直接除出来，不校验不取整
qty = target_notional / price        # 可能 NaN/inf/超小数位
exchange.submit(Order(symbol, qty))  # 交易所拒单或下出天量
```

**detect** —— 查找下单前对 qty 的校验：是否检查 NaN/inf/负数、是否对齐 stepSize/lotSize、是否满足交易所 minNotional、是否有 max qty 上限。若 qty 直接由 `position_value/price` 算出未做任何 clamp/round/校验，即缺陷。

**fix** —— 数量做完整 sanity：有限性 + 对齐步长 + min/max notional 双边界：

```python
MIN_NOTIONAL = Decimal('10')
MAX_QTY      = Decimal('1000')

def qty_sanity(symbol, qty, price, step, min_notional=MIN_NOTIONAL):
    if not qty.is_finite() or qty <= 0:
        raise RejectOrder(f'invalid qty {qty}')
    qty = (qty // step) * step               # 对齐 stepSize
    if qty <= 0:
        raise RejectOrder('qty below step size')
    notional = qty * price
    if notional < min_notional:
        raise RejectOrder(f'below min notional {notional}')
    if qty > MAX_QTY:
        raise RejectOrder(f'qty {qty} exceeds max')
    return qty
```

交易所的 stepSize/minNotional/lotSize 应从 exchangeInfo 拉取而非硬编码。

## 1.17 风控只在 post-trade 拦截，缺 pre-trade gate【fatal】

**why** —— 事后风控（成交后才发现超限再去平仓）等于「先开枪再瞄准」：超限敞口已真实建立，平仓时还要付滑点和手续费，极端行情下根本平不出来。所有硬限额（敞口/杠杆/单笔/价格/数量/限频）必须在订单离开本机前同步拦截。

**code smell**

```python
# 反例：先发单，成交回调里才查超限
def submit_order(order):
    return exchange.send(order)        # 无前置校验

def on_fill(fill):                     # 成交后才发现
    if exposure() > MAX_EXPOSURE:
        alert('超限了！')              # 敞口已建立，木已成舟
```

**detect** —— 审查下单路径：风控检查是在 submit 之前（pre-trade）还是在 on_fill/对账回调里（post-trade）。若 submit_order 没有同步风控 gate、只有成交后异步的「超限告警/事后平仓」，即为根本性缺陷。pre-trade 检查必须是阻断式（拒单），不能是异步告警。

**fix** —— 建立统一的 pre-trade 风控 gate，所有订单提交前同步过一遍全部硬限额，任一不过即拒单：

```python
def submit_order(order, price, state):
    # —— pre-trade gate：全部同步、阻断式 ——
    validate_order(order, price)          # 单笔上限/有限性
    qty_sanity(order.symbol, order.qty, price, step_of(order.symbol))
    price_sanity(order, state.ref_price[order.symbol], state.ref_ts[order.symbol])
    pre_trade_check(order, state.positions, state.open_orders, state.equity)  # 敞口/集中度
    check_effective_leverage(order, state)
    if not order_bucket.allow():
        raise RateLimited('order rate cap')
    if kill.is_active():
        raise TradingHalted('kill-switch')
    # 全部通过才真正发单
    return exchange.send(order)
```

post-trade 对账仍保留，但作为「第二道兜底+纠偏」，绝不替代 pre-trade 拦截。

---

# ② 保证金 / 爆仓 / 单点 / 运维 / 密钥

这一类的共同特征：**风控的「地基」与「运维生命线」**——保证金/强平算得对不对、进程死了有没有人知道、配置写错了能不能拦下、密钥会不会泄露、限额到底有没有真的生效。它们不像限额那样直观，却往往是实盘事故的最终根因。统一防御口径：**所有加载/校验/依赖失败一律 fail-closed；密钥永不进配置与日志；环境用单一枚举强绑定并物理隔离。**

## 2.1 无维持保证金检查（下单/持仓时不校验维持保证金率）【fatal】

**why** —— 合约/杠杆账户若不在下单和持仓监控前计算维持保证金率（Maintenance Margin Ratio），就无法预判强平距离。一旦行情逆向，账户会在毫无预警下被交易所强平，损失从可控滑点变成整笔保证金归零。这是 buy-side 风控最致命的缺口之一。

**code smell**

```python
# 反例：下单只校验余额够不够，从不看维持保证金率
def add_position(symbol, qty, price):
    if available_balance > qty * price:   # 只看名义够不够
        exchange.submit(Order(symbol, qty, price))
    # 没有 mmr / maint_margin / liquidation_distance 任何概念
    # leverage 直接乘进 size，无对应保证金占用校验
```

**detect** —— 搜 `maintenance|maint_margin|mmr|margin_ratio|available_margin|liquidation` 在风控/下单路径是否出现；检查 place_order/add_position 前是否有 `if margin_ratio < threshold: reject`；确认有按交易所阶梯保证金表（tiered margin / leverage bracket）计算的逻辑，而非固定值。

**fix** —— 在下单前置风控中强制计算 projected_margin_ratio，低于安全阈值直接拒单，持仓监控线程按 tick 重算：

```python
SAFE_MARGIN_BUFFER = Decimal('1.3')   # 维持保证金的 1.3 倍安全垫

def margin_check(equity, position_notional, symbol):
    mmr = maint_margin_rate(symbol, position_notional)   # 查阶梯保证金表
    required = position_notional * mmr * SAFE_MARGIN_BUFFER
    if equity < required:
        raise RejectOrder(f'insufficient margin: equity {equity} < {required}')
    margin_ratio = equity / (position_notional * mmr)
    return margin_ratio
```

阶梯保证金表随交易所规则定期同步，不要硬编码单一 mmr。

## 2.2 无爆仓/强平模拟（缺少强平价与压力测试）【fatal】

**why** —— 不预先计算每个持仓的强平价（liquidation price）、不做极端行情（如 ±20% 瞬时跳空、闪崩）下的爆仓模拟，就无法回答「此刻我离爆仓还有多远」。多空对冲、组合保证金场景下尤其危险——单腿强平会连锁触发整组爆仓。没有模拟就等于盲开杠杆。

**code smell**

```python
# 反例：风控只看当前未实现盈亏，不看"再跌 X% 会怎样"
def check_risk(state):
    if state.unrealized_pnl < -ALERT_THRESHOLD:
        alert('浮亏告警')
    # 没有 liq_price / stress_test / what_if_pnl，跳空闪崩无情景模拟
```

**detect** —— 搜 `liquidation_price|liq_price|stress|scenario|adverse_move|shock`；确认是否存在按 [-30%,-20%,-10%,+10%...] 价格冲击重算账户权益与保证金率的循环；检查是否在开仓时就记录并监控每仓的 liq_price 与当前价的距离百分比。

**fix** —— 开仓即计算 liq_price 并监控距离，加定时压力测试任务：

```python
def stress_test(positions, equity, shocks=(-0.30, -0.20, -0.10, 0.10, 0.20)):
    worst = []
    for shock in shocks:
        eq = equity + sum(p.qty * p.mark * shock for p in positions)
        mr = portfolio_margin_ratio(positions, eq, shock)
        worst.append((shock, eq, mr, mr < 1.0))   # mr<1 即爆仓
    return worst

def liq_distance(pos):
    return (pos.mark - pos.liq_price) / pos.mark   # 距强平百分比，过近即减仓
```

组合保证金账户要做跨腿联动强平模拟，不能只看单腿。

## 2.3 允许负现金 / 负可用余额（无透支与保证金占用前置拦截）【fatal】

**why** —— 若下单只做事后对账、不在下单前扣减并校验可用资金，并发下单或重复下单会让账户进入负现金/超额杠杆状态。交易所侧可能直接拒单导致策略状态与实盘不一致，或在某些保证金模式下放大风险敞口，最终演变成强平或穿仓欠款。

**code smell**

```python
# 反例：下单后才更新余额，可用不扣冻结，并发共用无锁
def place(order):
    exchange.submit(order)
    self.cash -= order.cost      # 事后扣，可为负，无 assert cash >= 0
# available 没减去 open orders reserved，多笔并发一起击穿到负
```

**detect** —— 搜余额扣减处是否有 `if available < required: reject` 的前置门；检查冻结资金 reserved/frozen/locked 是否计入 available；查并发场景下余额更新是否有原子操作或乐观锁；构造重复/并发下单的测试看是否能击穿到负余额。

**fix** —— 下单前原子性「预扣冻结」并校验，余额显式三态（total/available/frozen）：

```python
def reserve_margin(book, required):
    # Redis 原子 / DB 行锁，保证并发一致
    with book.lock():
        if book.available - book.frozen < required:
            raise RejectOrder('insufficient available margin')
        book.frozen += required           # 先冻结再发单
    assert book.available >= 0            # 硬断言，负现金 fail-closed
```

成交/撤单回调里再把冻结转为占用或释放，杜绝事后才发现负现金。

## 2.4 告警只走单一渠道（单点告警，渠道挂了风控等于失明）【high】

**why** —— 风控告警若只发到一个渠道（如只发企业微信或只发一个 Telegram bot），该渠道限流、token 失效、网络抖动或第三方故障时，致命风险事件（接近强平、风控触发、连接中断）会被静默丢弃。运维以为「没消息就是没事」，实则是告警通道死了。

**code smell**

```python
# 反例：硬编码单一 webhook，发失败只 log 不升级
def send_alert(msg):
    requests.post(TELEGRAM_WEBHOOK, json={'text': msg})  # 唯一通道，无 fallback
    # critical 也走同一个 IM，无短信/电话强通道，无重试
```

**detect** —— 搜 `send_alert|notify|webhook|telegram|bark|sms`，看是否只有单一目标；检查告警发送失败是否有 fallback 渠道与重试；确认 critical 级告警是否有强通道（短信/电话/PagerDuty）而非仅 IM。

**fix** —— 多渠道冗余扇出，按严重度分级路由，失败重试+降级+落盘补发：

```python
CHANNELS = {
    'critical': [sms_channel, phone_channel, telegram_channel],  # 强通道在前
    'warning':  [telegram_channel, im_channel],
}
def send_alert(level, msg):
    sent = False
    for ch in CHANNELS[level]:
        for _ in range(RETRY):
            if ch.send(msg): sent = True; break
        if sent and level != 'critical': break   # critical 多通道都发
    if not sent:
        persist_pending_alert(level, msg)         # 全失败落盘，下次启动补发
```

## 2.5 无心跳 / deadman switch（进程或风控线程死了无人知）【fatal】

**why** —— 风控/交易进程可能因死锁、OOM、异常退出、网络分区而「静默死亡」，但持仓还在市场上裸奔。没有心跳上报和 deadman switch（死人开关），外部监控无法察觉进程已停摆，持仓在无风控状态下持续暴露，等人工发现时往往已强平或大幅亏损。

**code smell**

```python
# 反例：没有任何心跳上报，进程 crash 后无外部探测
def trading_loop():
    while True:
        do_work()       # 死锁/OOM 后静默停摆，无 heartbeat / watchdog
    # 交易所侧也没启用 cancel-on-disconnect
```

**detect** —— 搜 `heartbeat|watchdog|deadman|dead_man|liveness|TTL`；确认是否有独立进程/外部监控按周期检查心跳是否新鲜，超时是否触发自动撤单或告警；检查交易所侧是否启用了 listenKey 心跳与 cancel-on-disconnect（如 Binance dead man's switch）。

**fix** —— 秒级写心跳，独立 watchdog 检测过期即降级，并启用交易所原生 deadman：

```python
# 风控线程：秒级写心跳
def beat(redis):
    redis.set('hb:risk', now_ts(), ex=5)   # 短 TTL，过期即视为死亡

# 独立 watchdog 进程
def watchdog(redis, exchange):
    if not redis.exists('hb:risk'):        # 心跳过期
        exchange.cancel_all_orders()       # 撤掉所有挂单
        send_alert('critical', 'RISK PROCESS DEAD - cancelled all orders')

# 交易所原生兜底：cancel-all-on-disconnect / dead man's switch
exchange.enable_cancel_on_disconnect(timeout_ms=5000)
```

进程用 systemd/supervisor 守护并配存活探测。

## 2.6 配置无 schema 校验（限额配置随便写，错了也加载）【high】

**why** —— 风控配置（限额、阈值、杠杆上限、白名单）若无 schema 校验就直接加载，一个拼写错误的字段名、缺失的限额项、单位写错（把 0.5 当成 50%）、类型错误（字符串当数字）都会让风控以为「没配=不限制」，从而静默放行本应被拦截的风险。配置错误是实盘事故的高频根因。

**code smell**

```python
# 反例：load 后宽松 get，缺失静默默认，单位/类型不校验
cfg = yaml.safe_load(open('risk.yaml'))
max_notional = cfg.get('max_notional', float('inf'))  # 拼错字段=无限制
leverage     = cfg.get('leverage')                    # 字符串"10"也照用
# 无 pydantic/jsonschema，多写/拼错字段无人察觉
```

**detect** —— 搜配置加载处是否有 schema/pydantic/BaseModel/validate；检查必填限额缺失时行为是报错还是静默默认；确认数值有无范围/单位/正负约束；故意改坏一个字段名跑启动，看是否被拦下。

**fix** —— 用 pydantic 强 schema 校验，必填缺失即失败，未知字段报错，fail-closed：

```python
from pydantic import BaseModel, Field, condecimal

class RiskConfig(BaseModel):
    model_config = {'extra': 'forbid'}              # 未知字段直接报错
    max_order_notional: condecimal(gt=0)            # 必填、正数
    max_leverage: int = Field(gt=0, le=125)
    max_drawdown_pct: condecimal(gt=0, le=1)        # 明确范围，杜绝单位错

def load_risk_config(path):
    try:
        return RiskConfig(**yaml.safe_load(open(path)))
    except Exception as e:
        raise SystemExit(f'risk config invalid, refuse to start: {e}')  # fail-closed
```

校验在启动和热加载时都执行，失败沿用上一份已知良好配置或停机。

## 2.7 密钥写进风控配置（API key/secret 明文进配置或仓库）【fatal】

**why** —— 把交易所 API key/secret、提现权限密钥明文写进风控配置文件、提交进 git、或打进镜像，是直接的资金失窃路径。一旦配置被泄露（仓库公开、日志打印、备份外泄、容器镜像被拉取），攻击者可直接下单/提现，损失不可逆且无追回手段。

**code smell**

```yaml
# 反例：密钥明文进配置、混在普通参数里、还可能被 commit
exchange:
  api_key: "AbCdEf1234567890realkey"
  api_secret: "9f8e7d6c5b4a3realsecret"   # 明文 + 带提现权限
  max_notional: 50000
```

```python
logger.info(f'connecting with key={api_key}')   # 日志泄露完整密钥
```

**detect** —— 搜配置与代码里 `api_secret|api_key|secret|private_key|passphrase|token` 是否有明文长串；跑 git 历史/secret 扫描（gitleaks/trufflehog）；检查密钥是否从环境变量/密钥管理器注入而非写死；确认 API key 权限是否最小化（禁提现、限 IP）。

**fix** —— 密钥走环境变量/密钥管理，配置只存非敏感参数，缺失即 fail-closed，日志脱敏：

```python
import os
api_key    = os.environ['EXCHANGE_API_KEY']      # 缺失即 KeyError，拒绝启动
api_secret = os.environ['EXCHANGE_API_SECRET']
if not api_key or not api_secret:
    raise SystemExit('missing exchange credentials')   # fail-closed

def mask(s): return s[:4] + '****' if s else ''
logger.info(f'connecting with key={mask(api_key)}')    # 脱敏打印
```

`.env` 加 `.gitignore`；交易所侧绑定 IP 白名单、关闭提现权限、最小权限原则；已暴露密钥立即吊销轮换。

## 2.8 风控限额未在下单前真正强制执行（限额只是摆设/事后统计）【fatal】

**why** —— 最危险的形态：配置里有完整限额（单笔上限、持仓上限、日内亏损上限），但下单链路并未在发单前调用这些限额做硬拦截——限额只用于事后报表或告警。结果是策略 bug/参数错误/行情异常时，超额订单照样打到交易所，风控形同虚设，事后才发现已超限。

**code smell**

```python
# 反例：限额加载了，但 place_order 里搜不到 check_limit 调用
limits = load_limits()           # 加载了却不用

def place_order(order):
    return exchange.send(order)   # 没有任何 limits 校验

def risk_check(order):           # 有这函数，但返回值被忽略
    if over_limit(order):
        logger.warning('over limit')   # 只 log，不 return/raise，不拦截
```

**detect** —— 从 place_order 反向追踪：发单前是否必经一个会 raise/return reject 的 pre-trade risk gate；检查 risk_check 的返回是否真的拦住了 send_order（而非被忽略）；构造超限订单跑测试，看是被拒单还是成交后才报警。

**fix** —— 所有下单必经统一 pre-trade gate，校验失败直接拒单且不可绕过，硬/软限制分离：

```python
def place_order(order, price, state):
    ok, reason = risk_gate(order, price, state)   # 唯一收口
    if not ok:
        raise RejectOrder(reason)                 # 硬限制：超过即拒，不可旁路
    return exchange.send(order)

def risk_gate(order, price, state):
    if abs(order.qty) * price > state.limits.max_notional:
        return False, 'max notional'              # 资金/敞口类用硬限制
    if state.daily_pnl <= -state.limits.daily_loss:
        return False, 'daily loss limit'
    return True, ''
```

把「限额检查在下单前生效（超额必拒）」写成强制测试用例；风控 gate 放在最靠近发单的唯一收口处，杜绝旁路下单。

## 2.9 testnet / live 标志缺失或混淆（实盘当测试网跑，或反之）【fatal】

**why** —— 若没有清晰、显式、难以误设的 testnet/live 环境标志，极易把测试策略连到实盘（真实资金被乱单打爆）或把实盘风控连到测试网（以为有保护实则裸奔）。endpoint、API key、风控参数若不随环境强绑定，一行配置写错就是真金白银的事故。

**code smell**

```python
# 反例：endpoint 与标志各自独立设，默认实盘，无启动横幅
testnet = config.get('testnet', False)          # 漏配即默认 LIVE
base_url = config['base_url']                    # 与 testnet 标志可能不一致
# 测试与实盘共用一份配置靠注释切换，hardcode 了实盘 url
```

**detect** —— 搜 `testnet|sandbox|paper|live|prod|is_real|dry_run`；检查 endpoint 与环境标志是否由单一可信来源派生（而非两处分别设）；确认实盘需要显式 opt-in（如 ENV=PROD + 二次确认），默认应是安全的 testnet/dry-run；看启动日志是否醒目打印环境。

**fix** —— 单一枚举派生所有 endpoint/key/参数，实盘显式 opt-in + 横幅 + 二次确认，默认 fail-safe：

```python
from enum import Enum
class Env(Enum):
    TESTNET = 'testnet'; PAPER = 'paper'; LIVE = 'live'

ENDPOINTS = {Env.TESTNET: 'https://testnet...', Env.LIVE: 'https://api...'}

def boot(env: Env):
    if env is Env.LIVE:
        if os.environ.get('CONFIRM_LIVE') != 'YES':
            raise SystemExit('LIVE requires explicit CONFIRM_LIVE=YES')
        print('=' * 40 + '\n  !!! LIVE TRADING - REAL MONEY !!!\n' + '=' * 40)
    return Connector(endpoint=ENDPOINTS[env], creds=creds_for(env))  # key 随 env 强绑定
```

默认 fail-safe 为 testnet/dry-run；CI 阻止把 LIVE 配置打进非生产构建；testnet key 物理上无法连实盘。

## 2.10 环境未隔离（测试/实盘共用数据库/Redis/账户/进程）【high】

**why** —— 测试与生产共用同一 Redis、数据库、消息队列或交易账户时，测试流量会污染实盘风控状态：测试写的持仓/限额/订单状态被实盘读到，或实盘数据被测试覆盖。风控基于被污染的状态做决策，等于在错误的世界观下管真钱。

**code smell**

```python
# 反例：测试和实盘连同一个 Redis / 同一张表 / 同一子账户
redis = Redis(host='shared-redis', db=0)   # 两环境共用，靠 key 前缀软隔离
# 前缀可能漏写；同一交易子账户既跑测试又跑实盘
```

**detect** —— 对比各环境的 db/redis/mq host 与 namespace 是否物理隔离；检查是否靠前缀软隔离及前缀是否可能冲突；确认测试是否可能写入生产数据存储；查交易账户/子账户是否环境独占。

**fix** —— 每个环境物理隔离的实例/账户，连接串由 env 枚举强派生，交易用环境独占子账户：

```python
INFRA = {
    Env.LIVE:    dict(redis='redis://prod-redis:6379/0', db='postgres://prod-db/risk'),
    Env.TESTNET: dict(redis='redis://test-redis:6379/0', db='postgres://test-db/risk'),
}
def connect(env: Env):
    cfg = INFRA[env]                       # 不同 host，物理隔离，非仅前缀
    return Redis.from_url(cfg['redis']), connect_db(cfg['db'])
# 交易账户：testnet/live 各用独占子账户 + 独立凭证，禁止本地直连生产
```

## 2.11 限额硬编码不可热加载（改风控要重启=带敞口停机或不敢改）【high】

**why** —— 限额若硬编码在源码里，调整风控参数必须改代码+重启进程。市场剧变需要紧急收紧限额时，重启意味着带着持仓断风控、撤单、再上线的窗口期裸奔；或因怕重启而干脆不调，让明显过宽的限额继续放行风险。两难本身就是事故温床。

**code smell**

```python
# 反例：限额是源码字面量，改阈值要改代码重新部署
MAX_POSITION = 100          # 改它必须改 .py + 重启
MAX_LEVERAGE = 20
# 没有 reload_config / SIGHUP / 配置中心订阅，运维手册写"改限额需重启"
```

**detect** —— 搜风控阈值是字面量常量还是从 config 读取；检查是否有 reload_config/SIGHUP handler/配置中心 watch；确认改限额的标准操作是「改配置即生效」还是「改代码重启」；看是否有「不重启即可收紧限额」的应急路径。

**fix** —— 限额外置配置，支持热加载（文件监听/SIGHUP/配置中心），校验后原子替换，并设「紧急收紧」快速路径：

```python
import signal
class LiveLimits:
    def __init__(self, path):
        self.path = path; self.reload()
    def reload(self, *_):
        new = load_risk_config(self.path)      # 走 schema 校验
        self._limits = new                     # 原子替换运行时限额
    def tighten(self, key, value):             # 应急：只允许变严
        if value < getattr(self._limits, key):
            object.__setattr__(self._limits, key, value)

limits = LiveLimits('risk.yaml')
signal.signal(signal.SIGHUP, limits.reload)    # kill -HUP 即热加载
```

热加载同样走 schema 校验，加载失败保留旧配置不中断风控。

## 2.12 限额加载失败时 fail-open（加载挂了就当没限额，继续下单）【fatal】

**why** —— 风控里最隐蔽的致命设计：配置/限额加载失败（文件丢失、配置中心不可达、解析异常、热加载出错）时，代码选择「继续运行、暂时不限额」（fail-open）。这意味着恰恰在系统已经异常的时刻，风控自动关闭，订单无限额地涌向市场。风控必须 fail-closed——加载不出限额就不许下单。

**code smell**

```python
# 反例：加载失败 except pass，用空 dict 当无限制
try:
    limits = load_limits()
except Exception:
    limits = {}                      # 拿不到限额就放行——fail-open
max_notional = limits.get('max_notional', float('inf'))  # 缺失=无限制
```

**detect** —— 审查配置/限额加载的异常分支：失败时是停机/拒单（fail-closed）还是继续无限额（fail-open）；搜 `except: pass`、空默认 dict、`limits or {}` 之类把缺失当无限制的写法；构造配置加载失败的故障注入，观察是否还能下单。

**fix** —— 加载/校验失败一律 fail-closed：启动失败拒绝启动，运行中失败保留旧配置，无旧配置则进 cancel-only：

```python
def safe_reload(state, path):
    try:
        new = load_risk_config(path)          # schema 校验
    except Exception as e:
        if state.limits is None:              # 首次启动即失败
            state.enter_cancel_only()         # 只撤不开的安全态
            raise SystemExit(f'no valid limits, cancel-only: {e}')
        send_alert('critical', f'reload failed, keep last-known-good: {e}')
        return                                # 保留上一份已知良好限额，绝不切无限额
    state.limits = new
```

把「加载失败=禁止下单」写成强制测试。

## 2.13 风控异常被吞掉静默失效（风控检查 try/except pass 后照常下单）【fatal】

**why** —— 风控检查函数内部抛异常（除零、空值、外部依赖超时、数据缺失）时，若被宽泛 try/except 吞掉并默认放行，风控就在出错的那一刻静默失效，订单绕过本应有的拦截。这是「看起来有风控、实际遇到 edge case 就关闭」的隐形地雷，比没有风控更危险（因为给了虚假安全感）。

**code smell**

```python
# 反例：风控异常被吞掉，异常路径继续下单
def submit(order):
    try:
        risk_check(order)            # 内部可能除零/空值/超时
    except Exception:
        pass                         # 吞掉异常 → 静默放行
    return exchange.send(order)      # 风控失效照样发单
```

**detect** —— 搜风控路径里的 `except Exception`/`except:`，看异常后是拒单还是放行；检查风控函数返回值在异常时的语义（None/异常是否被当通过）；故障注入：让风控依赖（行情、余额、配置）抛错，观察订单是否仍被发出。

**fix** —— 风控异常一律 fail-closed：抛错即视为「未通过」拒单并告警，依赖缺失按最坏情况处理：

```python
def submit(order, state):
    try:
        ok, reason = risk_check(order, state)
    except Exception as e:
        send_alert('critical', f'risk_check raised, reject: {e}')
        raise RejectOrder(f'risk check error (fail-closed): {e}')   # 异常=拒单
    if not ok:
        raise RejectOrder(reason)

    if state.ref_price.get(order.symbol) is None:    # 依赖缺失
        raise RejectOrder('no reference price (worst-case reject)')  # 当作触发限额
    return exchange.send(order)
```

为每条风控规则加「依赖失败时拒单」的测试。

## 2.14 多策略 / 多账户共享限额未隔离（一个策略吃光全局额度，连坐爆仓）【high】

**why** —— 多个策略或多个账户跑在同一风控下时，若限额只有全局口径、不按策略/账户隔离，一个出问题的策略（参数错、信号爆量、bug 刷单）会吃光共享的持仓/资金/风险额度，挤占甚至拖垮其他正常策略；或一个账户的亏损额度被另一个账户的盈利掩盖，掩盖真实风险敞口。归因与止损都无从下手。

**code smell**

```python
# 反例：只有 global 级限额，多策略共用同一计数器无隔离
GLOBAL_POSITION = 0
def on_order(order):
    global GLOBAL_POSITION
    if GLOBAL_POSITION + order.qty < GLOBAL_LIMIT:   # 谁先下谁吃满
        GLOBAL_POSITION += order.qty                 # 无 per-strategy 归因
        place(order)
```

**detect** —— 检查限额配置与运行时计数是否带 strategy_id/account_id/symbol 维度；搜是否有 per-strategy 的额度封顶与单策略熔断；确认能否按策略/账户独立查询当前敞口与剩余额度；模拟一个策略疯狂下单，看是否会击穿到其他策略的额度。

**fix** —— 限额分层隔离（global > per-account > per-strategy > per-symbol），子额度独立计数封顶，单策略超限只熔断该策略：

```python
def check_layered(order, state):
    s, a = order.strategy_id, order.account_id
    if state.strat_notional[s] + order.notional > state.limits.per_strategy[s]:
        state.halt_strategy(s, 'per-strategy cap')      # 只熔断该策略，不连坐
        raise RejectOrder(f'strategy {s} cap')
    if state.acct_notional[a] + order.notional > state.limits.per_account[a]:
        raise RejectOrder(f'account {a} cap')
    if state.global_notional + order.notional > state.limits.global_cap:
        raise RejectOrder('global cap')

# 一致性校验：子额度之和 ≤ 全局，避免子额度合计超发
assert sum(limits.per_strategy.values()) <= limits.global_cap
```

敞口与盈亏按策略/账户独立归因，便于精准止损。

---

> **审查收尾 checklist**：① 是否存在唯一的 pre-trade gate，所有下单收口于此？② 所有硬限额（单笔/敞口/集中度/杠杆/价格/数量/限频/挂单）是否齐全且阻断式生效？③ 回撤口径是否 mark-to-market + 历史峰值 + 持久化？④ 熔断是否程序化自动平仓而非仅告警？⑤ kill-switch 是否兼具自动触发 + 受控复位 + 独立人工急停？⑥ 保证金/强平是否预算并压力测试？⑦ 心跳/deadman/多渠道告警是否就位？⑧ 配置是否 schema 校验、热加载、且全程 fail-closed？⑨ 密钥是否走 env、脱敏、最小权限？⑩ testnet/live 与环境是否单一枚举强绑定并物理隔离？任一项缺失，按上表严重度定级阻断或告警。

文档完整路径建议：`D:/脑洞大开项目/latency-hunter-toolkit/skills/risk-config-lint/references/risk-pitfalls.md`（以上为完整 markdown 正文，未写入磁盘，按要求直接作为返回值返回）。