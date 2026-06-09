# 连接器蓝图与模板库

> 这是 connector-forge 指向的「连接器蓝图与模板库」。所有代码均为**骨架,需填真实密钥与业务逻辑**,严禁直接复制到生产实盘下单。Python 为主,关键处保留英文注释。
> 设计原则:可靠性层(重连/心跳/重订阅/对账/限频)走冷路径,允许用锁与分配;真正的热路径是 `tick → signal → order`,具体热路径优化交给 `latency-audit` 体检。

---

## ① 通用骨架(WSConnector 基类)

一个交易所无关的 WebSocket 连接器基类:封装「指数退避+满抖动重连、心跳看门狗、声明式重订阅、消息分发、令牌桶限频、优雅关闭」六大可靠性支柱。Binance / OKX 专章都继承它,只覆盖差异点(URL、ping/pong 形态、登录签名、订阅格式)。

### 1.1 满抖动指数退避(reconnect backoff)

```python
import random

def backoff_delay(attempt: int, base: float = 0.5, cap: float = 30.0) -> float:
    """Full-jitter exponential backoff (AWS-recommended).

    delay = rand(0, min(cap, base * 2**attempt))
    - 满抖动把重连时刻彻底打散,避免 thundering herd
    - cap 封顶保证最坏情况仍在可接受时间内恢复
    - 调用方在『连接成功』后必须把 attempt 清零
    """
    upper = min(cap, base * (2 ** attempt))
    return random.uniform(0, upper)
```

### 1.2 令牌桶限频(token bucket,按 weight 计费)

```python
import asyncio
from time import monotonic

class TokenBucket:
    """Weight-based token bucket. 注意:按交易所『权重』而非『请求数』扣费。

    骨架:capacity / refill_per_sec 需按交易所文档与你的 VIP 等级填真实值,
    并把上限设在交易所硬限的 ~80% 留安全边际。
    """
    def __init__(self, capacity: float, refill_per_sec: float):
        self.capacity = capacity
        self.tokens = capacity
        self.rate = refill_per_sec
        self._ts = monotonic()
        self._lock = asyncio.Lock()  # 冷路径用锁无妨

    def _refill(self) -> None:
        now = monotonic()
        self.tokens = min(self.capacity, self.tokens + (now - self._ts) * self.rate)
        self._ts = now

    async def acquire(self, weight: float = 1.0) -> None:
        async with self._lock:
            while True:
                self._refill()
                if self.tokens >= weight:
                    self.tokens -= weight
                    return
                # 需要等多久才能凑够 weight 个令牌
                deficit = weight - self.tokens
                await asyncio.sleep(deficit / self.rate)
```

### 1.3 WSConnector 基类骨架

```python
import asyncio
import logging
from time import monotonic
from typing import Any, Callable, Awaitable

log = logging.getLogger("connector")

class WSConnector:
    """Exchange-agnostic WS connector skeleton.

    子类必须覆盖:
      - WS_URL / 多路由 URL
      - _build_subscribe(channel) -> dict | str    构造订阅帧
      - _is_heartbeat(msg) -> bool                 识别 ping/pong
      - _send_heartbeat()                          按交易所形态发心跳
      - _login()(如需鉴权)                        私有频道登录
      - _extract_channel(msg) -> key               分发用的频道键

    骨架:需填真实密钥、真实 URL、真实业务回调与状态对账逻辑。
    """

    WS_URL = "wss://example.invalid/ws"      # 子类覆盖
    HEARTBEAT_INTERVAL = 20.0                 # 主动发心跳间隔(秒)
    HEARTBEAT_TIMEOUT = 60.0                  # 看门狗判死阈值(心跳间隔 2~3 倍)
    AGE_LIMIT = 23 * 3600                     # 主动重连年龄(防 24h 强断)

    def __init__(self):
        self.desired: set = set()             # 期望订阅集合(single source of truth)
        self.handlers: dict[str, Callable[[Any], Awaitable[None]]] = {}
        self.bucket = TokenBucket(capacity=200, refill_per_sec=10)  # 骨架:按交易所填
        self._ws = None
        self._running = False
        self._connected_at = 0.0
        self._last_rx = 0.0                    # 用单调时钟,绝不用墙钟

    # ---------- 子类需覆盖的钩子(以下为占位) ----------
    def _build_subscribe(self, channel) -> Any: raise NotImplementedError
    def _is_heartbeat(self, msg) -> bool: return False
    async def _send_heartbeat(self) -> None: ...
    async def _login(self) -> None: ...        # 公共连接默认空实现
    def _extract_channel(self, msg) -> str: raise NotImplementedError

    # ---------- 订阅 API(声明式:只改 desired,由连接器收敛) ----------
    def on(self, channel: str, handler: Callable[[Any], Awaitable[None]]) -> None:
        self.handlers[channel] = handler

    async def subscribe(self, channel) -> None:
        self.desired.add(channel)
        if self._ws is not None:
            await self._send_one(channel)

    async def _send_one(self, channel) -> None:
        await self.bucket.acquire(weight=1)    # 重订阅突发也走限频,防被封
        frame = self._build_subscribe(channel)
        await self._ws_send(frame)

    async def _resubscribe(self) -> None:
        """重连后把整个 desired 集合重放一遍,并校验 ack(校验逻辑由子类补)。"""
        for channel in list(self.desired):
            await self._send_one(channel)
        log.info("resubscribed %d channels", len(self.desired))

    # ---------- 连接主循环:退避重连 + 多协程编排 ----------
    async def run(self) -> None:
        self._running = True
        attempt = 0
        while self._running:
            try:
                await self._connect()
                attempt = 0                    # 成功立刻清零,否则下次直接顶到 cap
                await self._after_connect()
                await self._read_loop()        # 正常返回 = 对端关闭/看门狗触发
            except asyncio.CancelledError:
                raise
            except Exception as exc:           # noqa: BLE001 — 顶层兜底,记录后重连
                log.warning("connection error: %r", exc)
            finally:
                await self._safe_close()
            if not self._running:
                break
            attempt += 1
            delay = backoff_delay(attempt)
            if attempt >= 8:                   # 连续失败告警,避免静默死亡
                log.error("reconnect attempt=%d, exchange may be down", attempt)
            await asyncio.sleep(delay)

    async def _after_connect(self) -> None:
        self._connected_at = monotonic()
        self._last_rx = monotonic()
        await self._login()                    # 私有频道:先登录后订阅
        await self._resubscribe()

    async def _read_loop(self) -> None:
        """读循环 + 看门狗 + 心跳 + 年龄计时器,任一触发即退出本循环重连。"""
        wd = asyncio.create_task(self._watchdog())
        hb = asyncio.create_task(self._heartbeat_loop())
        age = asyncio.create_task(self._age_timer())
        try:
            async for raw in self._ws_iter():
                self._last_rx = monotonic()    # 任意入站字节都算活性
                msg = self._parse(raw)
                if self._is_heartbeat(msg):
                    continue
                await self._dispatch(msg)
        finally:
            for t in (wd, hb, age):
                t.cancel()

    async def _watchdog(self) -> None:
        while True:
            await asyncio.sleep(self.HEARTBEAT_INTERVAL / 2)
            if monotonic() - self._last_rx > self.HEARTBEAT_TIMEOUT:
                log.warning("heartbeat dead (%.1fs silent), force reconnect",
                            monotonic() - self._last_rx)
                await self._force_close()      # 触发外层 run() 重连
                return

    async def _heartbeat_loop(self) -> None:
        while True:
            await asyncio.sleep(self.HEARTBEAT_INTERVAL)
            try:
                await self._send_heartbeat()
            except Exception:                  # noqa: BLE001
                return                         # 发不出心跳 = 连接已坏,交看门狗收口

    async def _age_timer(self) -> None:
        """主动在 24h 强断前重连。最佳实践:新连接 ready 后再关旧的(overlap)。"""
        await asyncio.sleep(self.AGE_LIMIT)
        log.info("connection aged out, proactive reconnect")
        await self._force_close()

    # ---------- 消息分发 ----------
    async def _dispatch(self, msg) -> None:
        key = self._extract_channel(msg)
        handler = self.handlers.get(key)
        if handler is None:
            log.debug("no handler for channel=%s", key)
            return
        await handler(msg)

    # ---------- 优雅关闭 ----------
    async def shutdown(self) -> None:
        """SIGTERM/SIGINT 入口:停接新意图 → 处理在途 → 关 WS → flush → 落盘。"""
        self._running = False
        # 业务策略:撤未成交挂单 / 交接对账(由上层实现)
        await self._safe_close()
        log.info("connector shut down cleanly")

    # ---------- 传输层占位(子类/具体库实现) ----------
    async def _connect(self) -> None: ...      # 建 socket + TCP_NODELAY
    async def _ws_send(self, frame) -> None: ...
    async def _ws_iter(self): ...              # async generator of raw frames
    def _parse(self, raw): ...                 # json.loads / 原始文本
    async def _safe_close(self) -> None: ...
    async def _force_close(self) -> None: ...
```

---

## ② Binance 专章(USD-M Futures)

> 重要环境迁移:2026-04-23 起 Futures WS 启用 `/public` `/market` `/private` 分路由,旧的未路由 URL 退役。把 base path 做成**可配置**,不要硬编码。

### 2.1 WS 组合 stream 订阅模板

```python
# 组合流端点:一条 socket 复用多订阅,消息体被包成 {"stream":..., "data":...}
#   - 多 symbol 必须用 /stream?streams=...  (而非 /ws/<single>)
#   - 所有 symbol 名一律小写!大写 BTCUSDT@aggTrade 静默返回空数据,无报错
#   - 常用流: <sym>@aggTrade / @bookTicker / @depth@100ms / @kline_1m / @markPrice@1s

PROD_COMBINED = "wss://fstream.binance.com/stream?streams={streams}"
TESTNET_COMBINED = "wss://fstream.binancefuture.com/stream?streams={streams}"

def build_combined_url(streams: list[str], testnet: bool = False) -> str:
    # streams already lowercased by caller
    joined = "/".join(streams)
    base = TESTNET_COMBINED if testnet else PROD_COMBINED
    return base.format(streams=joined)

# 也可连上后用控制帧动态订阅(注意 10 条/秒硬限,务必把多个 param 合进一帧)
def build_subscribe_msg(params: list[str], req_id: int) -> dict:
    """成功回包 {"result":null,"id":...} — null 即成功,别当错误。"""
    return {"method": "SUBSCRIBE", "params": params, "id": req_id}

# 分发:组合流必须按 msg["stream"] 解复用,data 在 msg["data"] 下一层
def demux(msg: dict):
    stream = msg.get("stream")     # e.g. "btcusdt@aggTrade"
    data = msg.get("data")         # 真正 payload
    return stream, data
```

**陷阱**:① symbol 大写 = 空流无报错;② 一帧只塞一个 param 然后狂发会爆 10/秒被踢(Futures 是 10/s,SPOT 才是 5/s,别照抄);③ 连了 `/stream` 却按裸流的 JSON 形状解析(data 多嵌了一层)。

### 2.2 ping/pong 与 24h 重连

```python
# Binance 在 WS 协议层每 3 分钟发一个 ping 控制帧,你须在 10 分钟内回 pong(复制 payload)。
# 这是 RFC6455 控制帧,不是 JSON 消息!成熟 WS 库通常自动回 pong——确认你的库开启了。
#
# 两个独立计时器(已在基类 _watchdog / _age_timer 实现,这里给 Binance 参数):
#   (1) 活性:~5min 无任何数据 → 强制重连
#   (2) 年龄:每条连接到 24h 被无条件强断 → 在 ~23h 主动重连
# 最佳实践:开新 socket → 重订阅 → 确认数据在流 → 再关旧 socket(overlap,避免行情缺口)。
# Futures market stream 没有 serverShutdown 预警事件(不像 spot WS-API),只能靠自己的计时器。

class BinanceMarketWS(WSConnector):
    HEARTBEAT_TIMEOUT = 300.0     # ~5min 活性阈值
    AGE_LIMIT = 23 * 3600         # 23h 主动重连,留余量
    # Binance 协议层 pong 由底层 ws 库自动回应,无需 _send_heartbeat 主动发
    async def _send_heartbeat(self) -> None:
        return  # rely on library auto-pong for protocol ping frames
```

### 2.3 listenKey 获取 + keepalive 定时

```python
import aiohttp

# 用户数据流生命周期:POST 取 key → 连 /ws/<listenKey> → 每 30min PUT 续期 → DELETE 关闭
# 关键:三个调用都只需 X-MBX-APIKEY 头,均【不签名】(无 HMAC),weight 各 1。
# listenKey 有效期 60min;务必在 30min 就续(不要卡 60min,一次慢请求就死)。

LISTENKEY_PATH = "/fapi/v1/listenKey"

class BinanceUserDataWS(WSConnector):
    def __init__(self, api_key: str, rest_base: str, ws_base: str):
        super().__init__()
        self._api_key = api_key            # 骨架:从 env 注入,勿硬编码
        self._rest_base = rest_base
        self._ws_base = ws_base
        self._listen_key: str | None = None

    async def _create_listen_key(self, session: aiohttp.ClientSession) -> str:
        headers = {"X-MBX-APIKEY": self._api_key}
        async with session.post(self._rest_base + LISTENKEY_PATH, headers=headers) as r:
            r.raise_for_status()
            body = await r.json()
            return body["listenKey"]

    async def _keepalive_loop(self, session: aiohttp.ClientSession) -> None:
        headers = {"X-MBX-APIKEY": self._api_key}
        while self._running:
            await asyncio.sleep(30 * 60)   # 30min,不是 60min,留延迟/抖动余量
            try:
                async with session.put(self._rest_base + LISTENKEY_PATH,
                                       headers=headers) as r:
                    if r.status == 400:    # -1125 listenKey 不存在 → 重建
                        await self._rebuild_listen_key(session)
                    else:
                        r.raise_for_status()
            except Exception as exc:       # noqa: BLE001
                log.warning("keepalive failed: %r — will rebuild", exc)
                await self._rebuild_listen_key(session)

    async def _rebuild_listen_key(self, session: aiohttp.ClientSession) -> None:
        """处理 listenKeyExpired 事件 / -1125:重建 key、重连、并 REST 对账。"""
        await self._force_close()
        self._listen_key = await self._create_listen_key(session)
        self.WS_URL = f"{self._ws_base}/ws/{self._listen_key}"
        # 重连后必须对账:补回断连窗口内可能漏掉的 fill / 仓位变化
        #   GET /fapi/v2/account , GET /fapi/v1/openOrders
        log.warning("listenKey rebuilt, reconcile account+openOrders via REST now")
```

**陷阱**:① 卡 60min 续期 = 零余量,一次慢请求流就死;② 带 listenKey 的 WS 仍受 3min ping/24h 强断约束;③ 收到 `{"e":"listenKeyExpired"}` 事件或 PUT 返回 `-1125` 必须重建 + REST 对账,否则握着死 socket 漏掉每一笔成交。

### 2.4 REST HMAC SHA256 签名函数

```python
import hmac
import hashlib
import time
import urllib.parse

# 规则:signature = HMAC_SHA256(secret, totalParams),totalParams = queryString + body 拼接。
# signature 必须是【最后一个】参数;API key 走 X-MBX-APIKEY 头,永不进签名串。
# 最稳的写法:签名 POST 时把【所有参数都放 query string】,body 留空,避免拼接错位。

def sign_query(params: dict, secret: str, server_offset_ms: int = 0) -> str:
    """Return urlencoded query string WITH signature appended last.

    骨架:secret 从 env / KMS 注入;server_offset_ms 来自时钟同步(见 2.6)。
    """
    params = dict(params)                      # 不可变:拷贝,绝不原地改入参
    params["timestamp"] = int(time.time() * 1000) + server_offset_ms
    params.setdefault("recvWindow", 5000)      # 小窗口,别用 60000 掩盖时钟漂移
    query = urllib.parse.urlencode(params)     # 这就是被签的精确字节序列
    sig = hmac.new(secret.encode(), query.encode(), hashlib.sha256).hexdigest()
    return f"{query}&signature={sig}"          # signature 永远拼在最后

def signed_headers(api_key: str) -> dict:
    return {"X-MBX-APIKEY": api_key}           # 仅放 key,绝不放 secret
```

**陷阱**:签名串与上线字节必须逐字节一致;`-1022` 几乎都是「签的字符串 ≠ 发的字符串」(如 query/body 拆分时漏了 `&`)。RSA 签名含 `/` 和 `=`,必须 URL-encode。

### 2.5 权重限频处理(429 / 418 / Retry-After)

```python
# USD-M Futures 请求权重 ~2400/分钟/IP(独立于 spot;别硬编码,用 exchangeInfo 核实)。
# - 权重按【IP】计,同 IP 上多 bot/多 key 共享 2400 预算
# - 每个响应读 X-MBX-USED-WEIGHT-1M,接近上限就主动暂停
# - 429 立即退避;418 = IP 已被自动封禁(2min 起,累犯翻倍最长 3 天)
# 下单数另有【按 UID】的独立配额:X-MBX-ORDER-COUNT-10S / -1M,与权重是两套账。

WEIGHT_CAP = 2400          # 骨架:务必用 GET /fapi/v1/exchangeInfo rateLimits 核实
WEIGHT_SOFT = int(WEIGHT_CAP * 0.8)

async def handle_rate_headers(resp) -> None:
    used = int(resp.headers.get("X-MBX-USED-WEIGHT-1M", 0))
    if used > WEIGHT_SOFT:
        log.warning("weight %d/%d near cap, throttle", used, WEIGHT_CAP)
        # 主动暂停一小段,把请求平滑回限额内

async def on_http_error(resp) -> float:
    """Return seconds to back off. 429=退避避免升级;418=IP 已封,读 Retry-After 等满。"""
    retry_after = float(resp.headers.get("Retry-After", "1"))
    if resp.status == 418:
        log.error("IP BANNED (418), wait %.0fs", retry_after)
    elif resp.status == 429:
        log.warning("rate limited (429), back off %.0fs — do NOT keep retrying", retry_after)
    return retry_after
# 注意:order-count 的 429【不带 Retry-After】,需从 exchangeInfo 的 ORDERS 限额自行计算退避。
```

### 2.6 时钟同步(recvWindow / -1021)

```python
# 服务器规则:接受当 (timestamp < serverTime+1000) 且 (serverTime - timestamp <= recvWindow)。
# VPS 时钟持续漂移,维护一个 offset = serverTime - localTime,每个出站 timestamp 都加上它。

async def sync_clock(session, rest_base: str) -> int:
    """GET /fapi/v1/time(weight 1,无需鉴权)→ 返回 offset(ms),每小时重算一次。"""
    t0 = int(time.time() * 1000)
    async with session.get(rest_base + "/fapi/v1/time") as r:
        server = (await r.json())["serverTime"]
    t1 = int(time.time() * 1000)
    rtt = t1 - t0
    offset = server - (t0 + rtt // 2)          # 扣掉单程延迟估计
    if abs(offset) > 500:
        log.warning("clock drift %dms", offset)
    return offset
# 反模式:用 recvWindow=60000 掩盖漂移(放弃重放保护);只在启动校一次(长跑后累积漂移)。
```

### 2.7 下单示例(MARKET / LIMIT,含幂等与精度)

```python
import uuid

# POST /fapi/v1/order (SIGNED)
#   MARKET: symbol+side+type+quantity        LIMIT: 另加 price+timeInForce
#   - Hedge Mode 必须带 positionSide(LONG/SHORT);One-way 默认 BOTH
#   - newOrderRespType=RESULT 同步拿到成交结果,省一次查单
#   - quantity 是【基础币数量】,不是 USDT!
#   - 务必带 newClientOrderId 做幂等追踪

def build_market_order(symbol: str, side: str, qty: str, hedge: bool = False) -> dict:
    order = {
        "symbol": symbol.upper(),
        "side": side,                          # BUY / SELL
        "type": "MARKET",
        "quantity": qty,                       # 已按 stepSize 取整(见下)
        "newOrderRespType": "RESULT",
        "newClientOrderId": f"lh-{uuid.uuid4().hex[:20]}",  # 幂等键
    }
    if hedge:
        order["positionSide"] = "LONG" if side == "BUY" else "SHORT"
    return order

def build_limit_order(symbol: str, side: str, qty: str, price: str) -> dict:
    return {
        "symbol": symbol.upper(),
        "side": side,
        "type": "LIMIT",
        "timeInForce": "GTC",                  # GTX=post-only / IOC / FOK / GTD
        "price": price,                        # 已按 tickSize 取整
        "quantity": qty,
        "newClientOrderId": f"lh-{uuid.uuid4().hex[:20]}",
    }

# --- 精度与最小名义:必须用 exchangeInfo 的 tickSize/stepSize,别用 pricePrecision ---
from decimal import Decimal, ROUND_DOWN

def round_step(value: str, step: str) -> str:
    """qty = floor(rawQty/stepSize)*stepSize,用 Decimal 避免浮点多出小数。"""
    v, s = Decimal(value), Decimal(step)
    return str((v // s) * s)
# notional = price*qty 必须 >= MIN_NOTIONAL(常见 5 USDT,但 Binance 会改,读 exchangeInfo)

# --- 退出/止盈止损安全栏 ---
#   reduceOnly=true:只缩仓不反向开仓(One-way 有效;Hedge 不接受 reduceOnly,改用 positionSide)
#   closePosition=true:配 STOP_MARKET/TAKE_PROFIT_MARKET 全平,不传 quantity
#   两者互斥;2025-12-09 起 STOP/TP/TRAILING 在 /fapi/v1/order 被拒(-4120),改走 Algo 端点

# --- 5xx / 超时安全:状态未知,绝不盲目重发 ---
async def place_order_idempotent(session, rest_base, secret, api_key, order: dict):
    """5xx/超时 → 用 origClientOrderId 查真实状态,再决定是否重发,严防双倍仓位。"""
    qs = sign_query(order, secret)
    headers = signed_headers(api_key)
    try:
        async with session.post(rest_base + "/fapi/v1/order?" + qs, headers=headers) as r:
            if r.status >= 500:
                raise ConnectionError("5xx: order state UNKNOWN")
            return await r.json()
    except (ConnectionError, asyncio.TimeoutError):
        # 不重发!先按 clientOrderId 查单确认是否已成
        check = {"symbol": order["symbol"], "origClientOrderId": order["newClientOrderId"]}
        cqs = sign_query(check, secret)
        async with session.get(rest_base + "/fapi/v1/order?" + cqs, headers=headers) as r:
            return await r.json()              # 存在即已成,不再重发
```

**陷阱**:① 5xx 后盲目重发 = 头号资金风险(双倍仓位),幂等键 + 查证后再重发是铁律;② Hedge/One-way 模式与 positionSide 不匹配被拒;③ 用 float 除法取整会重新引入多余小数,务必用 Decimal/整数步长。

---

## ③ OKX 专章(V5,永续合约)

> 三条独立 WS 连接,频道路由到正确 base:`/public`(行情)、`/private`(账户/持仓/订单)、`/business`(candle、orders-algo、grid)。candle 在 `/public` 上**收不到任何数据**。

### 3.1 WS public/private/business 订阅 + 30s ping

```python
# 三个端点必须物理分开;频道路由错 = 静默无数据(top onboarding bug)
WS_PUBLIC   = "wss://ws.okx.com:8443/ws/v5/public"
WS_PRIVATE  = "wss://ws.okx.com:8443/ws/v5/private"
WS_BUSINESS = "wss://ws.okx.com:8443/ws/v5/business"
# Demo(模拟盘)用不同 host:wss://wspap.okx.com:8443/ws/v5/{public|private|business}

def route_endpoint(channel: str) -> str:
    if channel.startswith("candle") or channel in {"orders-algo", "algo-advance"} \
            or channel.startswith("grid"):
        return WS_BUSINESS
    if channel in {"account", "positions", "orders", "balance_and_position"}:
        return WS_PRIVATE
    return WS_PUBLIC

def build_subscribe(channel: str, inst_id: str) -> dict:
    # 永续合约 instId 带 -SWAP:BTC-USDT-SWAP(U本位线性)/ BTC-USD-SWAP(币本位反向)
    return {"op": "subscribe", "args": [{"channel": channel, "instId": inst_id}]}
    # 批量:多个 arg 塞一帧,但 payload < 64KB,且 480 次订阅/退订/登录 每小时每连接 上限

# --- OKX 心跳是【应用层文本】'ping'/'pong',不是 RFC6455 控制帧!---
# 30s 无任何推送/客户端消息即被断;在 20~25s 主动发,任意入站消息都重置计时器。
class OKXWS(WSConnector):
    HEARTBEAT_INTERVAL = 20.0      # 20~25s,绝不卡 30s
    HEARTBEAT_TIMEOUT = 30.0

    async def _send_heartbeat(self) -> None:
        await self._ws_send_text("ping")       # 发【裸文本】,不是 JSON

    def _is_heartbeat(self, msg) -> bool:
        return msg == "pong"                    # 收到裸文本 'pong',别 JSON.parse!

    def _parse(self, raw):
        if raw in ("ping", "pong"):             # 文本心跳不走 json.loads
            return raw
        import json
        return json.loads(raw)
```

**陷阱**:① 把 candle 当「公共行情」订到 `/public` → 收不到;② 把入站 `pong` 当 JSON 解析直接崩;③ ping 卡 30s 与服务器超时打架,留 20~25s 余量;④ 重连风暴里整盘重订阅撞 480/小时/连接 上限,要分片到多连接 + 批量。

### 3.2 WS private login 签名(HMAC base64)

```python
import hmac, hashlib, base64, time

# 私有/business(algo)连上后 30s 内必须先发 login,再订阅。
# 关键:prehash 是【固定串】 timestamp + 'GET' + '/users/self/verify'(与 REST 不同!)
# sign = Base64(HMAC_SHA256(secret, prehash))

def okx_ws_login(api_key: str, secret: str, passphrase: str) -> dict:
    ts = str(int(time.time()))                 # 【Unix 秒】字符串,不是 ISO 毫秒!
    prehash = ts + "GET" + "/users/self/verify"
    sign = base64.b64encode(
        hmac.new(secret.encode(), prehash.encode(), hashlib.sha256).digest()
    ).decode()
    return {"op": "login", "args": [{
        "apiKey": api_key,
        "passphrase": passphrase,              # 第三凭证,明文上线(务必 TLS,勿 log)
        "timestamp": ts,
        "sign": sign,
    }]}
    # 等 {"event":"login","code":"0"};失败 60009 / 已登录 60022。登录成功才能订阅!
```

**陷阱**:① WS login 时间戳是 **Unix 秒**,REST 却是 **ISO 毫秒**——混用是头号 WS 鉴权失败原因;② 重连时必须「先 login 成功 → 再重订阅」,顺序反了订阅被拒;③ 30s 后签名过期,别复用缓存 sign。

### 3.3 REST OK-ACCESS-* 头 + prehash

```python
import hmac, hashlib, base64, json
from datetime import datetime, timezone

# 每个鉴权 REST 请求需 4 个头 + Content-Type: application/json:
#   OK-ACCESS-KEY / OK-ACCESS-SIGN / OK-ACCESS-TIMESTAMP / OK-ACCESS-PASSPHRASE
# prehash = timestamp + METHOD(大写) + requestPath + body
#   GET: query 串算进 requestPath,body=''
#   POST: body 是你【实际上线的那串原始 JSON 字节】

def _iso_millis() -> str:
    # ISO-8601 毫秒 UTC,如 2026-06-09T12:34:56.789Z —— 与 WS 的秒格式相反
    return datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%S.%f")[:-3] + "Z"

def okx_rest_headers(method: str, request_path: str, body_obj,
                     api_key: str, secret: str, passphrase: str,
                     demo: bool = False) -> tuple[dict, str]:
    """Return (headers, raw_body). raw_body 必须原样上线,别二次序列化导致签名不符。"""
    ts = _iso_millis()
    # 只序列化一次,签的串和发的串字节相同
    raw_body = "" if body_obj is None else json.dumps(body_obj, separators=(",", ":"))
    prehash = ts + method.upper() + request_path + raw_body
    sign = base64.b64encode(
        hmac.new(secret.encode(), prehash.encode(), hashlib.sha256).digest()
    ).decode()
    headers = {
        "OK-ACCESS-KEY": api_key,
        "OK-ACCESS-SIGN": sign,
        "OK-ACCESS-TIMESTAMP": ts,
        "OK-ACCESS-PASSPHRASE": passphrase,    # 明文,务必 TLS + 不 log
        "Content-Type": "application/json",
    }
    if demo:                                   # 模拟盘:加 demo header(见 3.4)
        headers["x-simulated-trading"] = "1"
    return headers, raw_body
```

**陷阱**:① body 序列化两次(签一次、发一次)只要空格/顺序差一点就签名不符;② GET 忘了把 query 串算进 requestPath;③ 用本地时区而非 UTC(`50102` 头号原因);④ 时间漂移超 30s 被拒(`50102`),需用 `GET /api/v5/public/time` 校 offset。

### 3.4 demo header(模拟盘隔离)

```python
# 模拟盘:① 单独申请 Demo API key(与实盘 key 不通用)② REST 加头 x-simulated-trading: 1
#         ③ WS 连 demo host wspap.okx.com(不只是加头,host 也不同)
# 实盘:不加该头(或填 0)。签名/下单流程与实盘逐字节相同。

def demo_header(is_demo: bool) -> dict:
    return {"x-simulated-trading": "1"} if is_demo else {}
# 陷阱:实盘 key 配 demo header(或 demo key 打实盘 host)→ 鉴权/权限错;
#       提币/划转/申赎类端点在 demo 不可用。
```

### 3.5 下单示例(永续,含 posSide / reduceOnly)

```python
import uuid

# POST /api/v5/trade/order
#   instId=BTC-USDT-SWAP / tdMode='cross'|'isolated'(永续绝不能 'cash')
#   side='buy'|'sell' / ordType='market'|'limit'|'post_only'|'fok'|'ioc'
#   sz = 【合约张数】,不是币数/USD!  px = 限价单价格
#   clOrdId 做幂等(a-zA-Z0-9,<=32 字符,账户内唯一)

def build_okx_order(inst_id: str, side: str, sz: str,
                    ord_type: str = "market", px: str | None = None,
                    pos_mode: str = "net_mode", reduce_only: bool = False) -> dict:
    order = {
        "instId": inst_id,
        "tdMode": "cross",                     # 或 isolated,绝不 cash
        "side": side,
        "ordType": ord_type,
        "sz": sz,                              # 张数,已按 lotSz 取整
        "clOrdId": f"lh{uuid.uuid4().hex[:24]}",  # 幂等键
    }
    if px is not None:
        order["px"] = px                       # 已按 tickSz 取整
    # 仓位模式:必须先 GET /api/v5/account/config 查 posMode,再决定 posSide
    if pos_mode == "long_short_mode":          # 双向持仓:posSide 必填
        # 开多 buy+long / 开空 sell+short / 平多 sell+long(+reduceOnly)
        if side == "buy":
            order["posSide"] = "long" if not reduce_only else "short"
        else:
            order["posSide"] = "short" if not reduce_only else "long"
    else:                                      # 单向 net_mode:用 reduceOnly 平仓
        if reduce_only:
            order["reduceOnly"] = True
    return order
# 名义:线性 USDT 永续 notional = sz*ctVal*markPx;反向 USD = sz*ctVal
#   ctVal/ctType/lotSz/minSz/tickSz 来自 GET /api/v5/public/instruments,sz 须按 lotSz 取整
```

**陷阱**:① `sz` 当成币数/USD → 量级错几个数量级;② net 模式硬塞 hedge 的 `posSide`(或反之)→ `51000`;③ `tdMode='cash'` 用在 SWAP 上;④ 漏 `clOrdId` 超时重发 → 双倍成交。

### 3.6 OKX 响应封套与错误分级

```python
# OKX 用 HTTP 200 + 业务 code 返回;批量下单必须逐条看 data[i].sCode,
# 顶层 code 可能是 '1'(整体失败)而单腿各有自己的 sCode。
def okx_classify(code: str) -> str:
    if code == "0":
        return "OK"
    if code in {"50011", "50026", "63999"}:    # 限频/系统/内部 → 退避重试
        return "RETRYABLE"
    if code == "50102":                        # 时间戳过期 → 重新校时再签
        return "RESYNC_TIME"
    if code in {"50104", "50105", "50113"}:    # passphrase 缺失/错 / 签名错 → 凭证问题
        return "CREDENTIAL"
    return "FATAL"                             # 51xxx 参数类(如 posSide/余额)→ 修了再说,别重试
# 关键码:50061 子账户限频 / 51000 参数 / 51008 余额不足 / 51119 风控拒单 /
#         60009 WS登录失败 / 60018 频道不存在(常是连错 URL)
```

---

## ④ 安全模板(密钥/脱敏/环境切换)

```python
import os, logging

# --- 1. 密钥一律从 env / KMS 注入,缺失即 fail-fast,绝不硬编码进源码/配置/Git ---
def load_credentials(prefix: str) -> dict:
    """prefix 如 'BINANCE_TESTNET' / 'OKX_LIVE',按环境取不同变量名物理隔离。"""
    try:
        creds = {
            "api_key": os.environ[f"{prefix}_API_KEY"],
            "api_secret": os.environ[f"{prefix}_API_SECRET"],
        }
        pp = os.environ.get(f"{prefix}_PASSPHRASE")   # OKX 第三凭证
        if pp:
            creds["passphrase"] = pp
    except KeyError as e:
        raise SystemExit(f"FATAL: missing credential env {e}, refuse to start")
    return creds

# --- 2. 日志脱敏:key/secret/sign/token/passphrase 全程打码,只留后 4 位 ---
def mask(s: str | None) -> str:
    if not s or len(s) <= 8:
        return "****"
    return s[:4] + "****" + s[-4:]

class RedactFilter(logging.Filter):
    """挂到 root logger,兜底过滤误打进日志的敏感串(secret/passphrase/sign)。"""
    SENSITIVE = ("secret", "passphrase", "signature", "OK-ACCESS-SIGN")
    def filter(self, record: logging.LogRecord) -> bool:
        msg = record.getMessage().lower()
        if any(k in msg for k in self.SENSITIVE):
            record.msg = "[REDACTED sensitive log line]"
            record.args = ()
        return True
# 绝不打:完整 query(含 timestamp 推导面)、login 帧、请求头原文

# --- 3. testnet / live 切换:默认 testnet,切 live 需显式声明 + 醒目告警 ---
ENDPOINTS = {
    "binance": {
        "testnet": {"rest": "https://testnet.binancefuture.com",
                    "ws":   "wss://fstream.binancefuture.com"},
        "live":    {"rest": "https://fapi.binance.com",
                    "ws":   "wss://fstream.binance.com"},
    },
    "okx": {
        "testnet": {"rest": "https://www.okx.com",        # demo 靠 header/host 区分
                    "ws":   "wss://wspap.okx.com:8443/ws/v5"},
        "live":    {"rest": "https://www.okx.com",
                    "ws":   "wss://ws.okx.com:8443/ws/v5"},
    },
}

def resolve_env(exchange: str) -> tuple[str, dict, dict]:
    env = os.environ.get("EXCHANGE_ENV", "testnet")   # 默认 testnet,防手滑烧钱
    cfg = ENDPOINTS[exchange][env]
    creds = load_credentials(f"{exchange.upper()}_{env.upper()}")
    if env == "live":
        logging.getLogger("connector").warning(
            "!!! LIVE TRADING — 真实资金,确认无误再继续 !!!")
    logging.getLogger("connector").info("env=%s exchange=%s key=%s",
                                        env, exchange, mask(creds["api_key"]))
    return env, cfg, creds
# 陷阱:默认值设成 live → 手滑烧真钱;testnet/live 共用 key 变量名 → 切换误用;
#       endpoint 散落多处 if env=='live' → 漏改一处跨环境串台。
```

---

## ⑤ 连接器自检清单(逐项勾选)

上线任何实盘连接器前,逐条核对。任一项「否」都不要碰真钱。

- [ ] **重连**:断线走指数退避 + **满抖动**(非固定退避/裸 while),有 cap 封顶;成功后 attempt **清零**;连续失败 N 次有**告警**(非静默死亡)。
- [ ] **心跳看门狗**:维护 `last_rx`(用 **monotonic 单调时钟**,非墙钟);超时阈值 = 心跳间隔 **2~3 倍**(避免误杀);判死后既 `close` 又**通知上层行情已中断**。Binance 靠库自动回 RFC6455 pong;OKX 发**裸文本 `ping`** 且 `pong` 不 JSON 解析。
- [ ] **重订阅**:`desired` 集合是**单一事实来源**;重连后整盘重放并**校验 ack**;重订阅突发走**令牌桶节流**(防撞限频);超频道上限时**分片**到多连接。
- [ ] **序列缺口**:带 seq/u/U 的增量逐条校验连续性;检测到 gap **丢弃本地态 + 重拉 snapshot**(非跳过续用);重快照期间到达的增量先**缓冲**;重快照本身**退避+限频**(防死循环)。订单簿用「**先订增量 → 后拉快照 → 按 lastUpdateId 衔接**」;`qty==0` **删档**;价格转**定点整数**当 key;有 checksum 就定期比对(OKX CRC32)。
- [ ] **限频**:client 侧**令牌桶**主动限速,按**权重**(非请求数)计费,设在硬限 **80%**;读响应头(`X-MBX-USED-WEIGHT` / OKX `x-ratelimit-*`)闭环;429 读 **Retry-After** 退避,418/IP ban **熔断**;**下单配额**(Binance UID / OKX 子账户 1000/2s)与请求权重是**两套独立账**;同 IP 多进程需**跨进程共享**令牌桶。
- [ ] **时钟**:启动 + 周期调交易所 server-time 算 **offset**(扣 RTT 单程),签名统一用**校准时间**;守住 recvWindow(Binance 默认 5000,**别用 60000 掩盖漂移**);NTP 常开;**内部测延迟用 monotonic,签名用校准墙钟**。
- [ ] **签名**:签的字符串与上线字节**逐字节一致**(Binance signature 拼**最后**;OKX body **只序列化一次**);Binance HMAC-hex,OKX HMAC→**base64**;**OKX WS 用 Unix 秒、REST 用 ISO 毫秒**(别混);OKX **passphrase** 第三凭证非空校验。
- [ ] **密钥安全**:key/secret 从 **env/KMS** 注入,**绝不硬编码**进源码/配置/Git(`.gitignore` + 密钥扫描);日志/异常/上报对 secret/sign/passphrase **全程脱敏**;API key 配**最小权限**(只读行情 key 不开交易/提币)+ **IP 白名单**;启动校验缺失即 **fail-fast**。
- [ ] **幂等**:每单带**唯一 clientOrderId**(Binance `newClientOrderId` / OKX `clOrdId`);超时/网络抖动**先按 ID 查单**确认是否已成,再决定是否**用同一 ID** 安全重发(**5xx = 状态未知,绝不盲目重发**,防双倍仓位);撤单/改单同样走幂等+查证;clientOrderId **不复用**。
- [ ] **错误分级**:显式映射 **可重试**(超时/5xx/429/418/维护)/ **致命不可重试**(余额不足/参数非法/签名错/权限)/ **不确定**(下单超时→查证幂等);默认分支保守设 **FATAL**(未知错误不盲目重试);重试有**次数上限**;OKX 批量单逐条看 `data[i].sCode`(顶层 code 可能掩盖单腿失败)。
- [ ] **优雅关闭**:捕获 **SIGTERM/SIGINT**;有序关停:**停接新意图 → 处理在途(撤挂单/交接对账)→ 退订关 WS → flush 异步日志 → 落盘状态(last seq / 本地簿)**;设**关闭超时**兜底,卡住强退 + 告警;信号处理里**只 set event**,重活放主循环。
- [ ] **(附)环境切换**:`EXCHANGE_ENV` 默认 **testnet**,切 live 需**显式声明 + 醒目告警**;testnet/live 密钥**不同变量名物理隔离**;endpoint 收敛到**一份配置**(非散落 if);日志打印**当前环境**。
- [ ] **(附)低延迟**:交易/行情 socket 建好立即 `TCP_NODELAY`(防 Nagle ~40ms 尾延迟);**复用长连接**(连接池 keep-alive)避免每请求重建 TLS;热路径(解析/簿更新/下单编码)避免堆分配/锁/同步日志/文本 JSON——细项过一遍 **latency-audit**。