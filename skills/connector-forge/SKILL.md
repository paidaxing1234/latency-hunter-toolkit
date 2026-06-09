---
name: connector-forge
description: >-
  生成或修复生产级交易所行情+交易连接器骨架(自动重连、心跳看门狗、限频令牌桶、
  时间同步、签名、listenKey/login 续期、重订阅、序列缺口重快照、密钥走 env)。也能审已有连接器的可靠性缺口。
  这是个生成/脚手架型 skill,把『连接器锻造协议』固化下来,让连接器从地基上就不漏。
  触发词(中):连接器、连接器锻造、交易所对接、Binance连接器、OKX连接器、行情订阅、断线重连、
  老掉线、莫名其妙断线、签名报错、时间戳报错、listenKey 续期、重订阅、序列缺口、本地订单簿、实盘对不上。
  触发词(英):connector、connector-forge、exchange connector、market data feed、websocket reconnect、
  heartbeat、listenKey、ws login、orderbook snapshot、sequence gap、rate limit token bucket、clock drift。
  不适用:非交易所对接的通用网络代码;要你代填/生成真实 API 密钥(只给 env 注入骨架,绝不写死密钥);
  要你给出具体策略 alpha/信号逻辑/调参(那是策略层,不是连接器层);纯热路径微秒级优化体检(交给 latency-audit)。
license: MIT
---

# connector-forge / 连接器锻造

## 核心理念

连接器是量化系统的**地基**。它不直接赚钱,但**一漏就全盘崩**:90% 的「实盘对不上回测」「莫名其妙断线」「下单偶发失败」都出在这一层——而且都是**静默故障**,连接看似健康,数据却早已是死的或错的。

锻造连接器只有一条铁律:**宁可显式判死重建,绝不带病运行**。半开连接、丢失的订单簿增量、漂移的时钟、过期的 listenKey——这些都不会自愈,只会越积越错,直到你基于一个错误的盘口下单,亏的是真金白银。本 skill 把「怎么从地基上锻造一个不漏的连接器」固化成可执行协议。

## 何时启用

- 用户说「帮我生成/写一个 Binance/OKX 连接器」「搭个行情订阅骨架」「对接交易所的脚手架」。
- 用户报故障:「连接器老掉线」「莫名其妙断线」「签名报错 -1022/50113」「时间戳报错 -1021/50102」「listenKey 过期收不到成交」「重连后收不到数据」「本地订单簿和交易所对不上」。
- 用户把已有连接器丢过来要你**审可靠性缺口**(重连/心跳/限频/对账有没有漏)。

不启用:要你代填真实密钥、要具体策略 alpha/信号、纯微秒级热路径体检(→ latency-audit)。

## 锻造协议(生成一个连接器的六步)

### ① 确认边界(先问清,别瞎猜)

- **交易所**:Binance / OKX(其它先类比就近模板)。
- **市场**:现货 spot / U 本位合约 USD-M / 币本位 COIN-M / 永续 SWAP。限频、symbol 格式、签名细节按市场而非交易所分叉。
- **环境**:**默认 testnet**,切 live 必须显式声明并打醒目日志。
- **需要哪些层**:只要行情?还是行情+交易+用户数据流(成交回报)?

### ② 行情 WS 层(订阅 / ping-pong / 重连 / 缺口)

- 多 symbol 用**合并流/批量订阅**(一条连接喂多路),symbol 大小写严格按交易所要求。
- 维护「**期望订阅集合**」作为单一事实来源;每次重连后自动重订阅整集合并**校验 ack**。
- 带序列号的增量行情**逐条校验连续性**,发现缺口立即丢弃本地状态、**重拉 snapshot 重放**。

### ③ 交易层(REST/WS 签名 / 下单 / 时间同步)

- 签名严格按交易所算法(Binance HMAC-SHA256 query+body;OKX HMAC-SHA256→Base64 + passphrase)。
- 下单一律带 **clientOrderId** 幂等;超时走「先查证后重发」,绝不盲目重试。
- 时间戳基于**校准后的交易所时间**(周期拉 server time 维护 offset)。

### ④ 可靠性层(退避 / 看门狗 / 令牌桶 / 幂等)

- 重连用**指数退避 + 满抖动**,成功后重置计数;心跳**看门狗**用单调时钟判死。
- client 侧**令牌桶按 weight 主动限速**,卡在交易所上限 80%;429/418 读 Retry-After 退避。
- 见下方「通用可靠性清单」逐项落地。

### ⑤ 安全(密钥走 env / 日志脱敏)

- key/secret/passphrase 一律从**环境变量**注入,源码/配置/Git 里**绝不出现明文**;启动 fail-fast 校验。
- 日志、异常栈、上报里对 key/secret/sign/token **全程脱敏**(只留后 4 位)。
- API key 配**最小权限**(只读行情 key 不开交易/提币)+ IP 白名单。

### ⑥ 生成可运行骨架 + 标注 TODO

- 输出**结构清晰、能跑起来**的骨架,把交易所易变项(endpoint、限额、min notional)标成 `# TODO: 以 exchangeInfo/instruments 实时校验为准`,而不是写死。

## 两大交易所要点速查(完整模板见 references/connector-blueprint.md)

| 维度 | Binance(USD-M Futures 为例) | OKX(v5) |
|---|---|---|
| WS 端点路由 | 合并流 `/stream?streams=` vs 单流 `/ws/`;2026 起 `/public`(高频盘口)`/market`(markPrice/kline/ticker)`/private`(listenKey)分流,**legacy 不分流的 url 收不到 /market 类数据** | **三条独立连接**:`/public`(行情)`/private`(账户/订单)`/business`(**candle/algo 必须走这条**,走错 URL 静默无数据) |
| symbol 格式 | **全小写**(`btcusdt@aggTrade`),大写=静默空流 | 永续带 `-SWAP`(`BTC-USDT-SWAP`) |
| ping/pong | 服务器每 **3 分钟**发 WS 协议 ping,10 分钟内须回 pong;**每条连接 24h 强制断**,需提前重连 | **应用层文本** `'ping'`→`'pong'`(非协议帧!),**>30s 空闲即断**,20~25s 心跳 |
| WS 登录签名 | 用户流走 listenKey,**WS 不签名** | 私有连接先 `login`:prehash=`ts+'GET'+'/users/self/verify'`,**ts 是秒级字符串**(REST 是 ISO 毫秒,别混) |
| REST 签名 | HMAC-SHA256(secret, **queryString+body**),sig 作**最后一个**参数;`X-MBX-APIKEY` 头 | 4 头 KEY/SIGN/TIMESTAMP/**PASSPHRASE**;prehash=`ts+METHOD+path+body`,**ts 是 ISO 毫秒** |
| 时间窗 | `recvWindow` 默认 5000ms,漂移→ **-1021** | 偏差 >30s → **50102** |
| 限频 | 权重 **~2400/min/IP**(读 `X-MBX-USED-WEIGHT-1M`);下单数另算 per-UID;429 退避,418=**IP ban** | **每接口独立**限频;子账户下单 **1000/2s**(撤单不计);429 可能只是撮合繁忙,退避别封 key |
| 用户数据流续期 | `listenKey` 有效 60min,**每 30min** PUT 续期;`listenKeyExpired`/-1125 须**重建+REST 对账** | 私有 WS 推送,断线后 **REST 对账**(orders-pending / positions) |
| 仓位模式 | 单向 BOTH / 对冲须传 `positionSide`;`reduceOnly` vs `closePosition` 互斥 | `net` / `long_short`(对冲须 `posSide`);先查 `account/config` 的 `posMode` |
| 精度 | 用 `tickSize`/`stepSize`/`MIN_NOTIONAL`(**别用 pricePrecision** 取整),来自 `exchangeInfo` | `sz` 是**合约张数**(非币/U),用 `ctVal`/`lotSz`/`minSz`,来自 `public/instruments` |
| 5xx 安全 | **5xx≠确定失败**,订单状态未知;凭 clientOrderId **查证再决定**,严禁盲目重发 | HTTP 200 仍可能 `code!=0`;批量看 `data[i].sCode`;429 可能是撮合繁忙 |
| testnet | REST `testnet.binancefuture.com` / WS `fstream.binancefuture.com`,**key 独立** | 模拟盘 REST 加头 `x-simulated-trading:1` + 专用 demo key;WS 用 `wspap.okx.com` |

> ⚠️ 端点/限额/min notional 都是**移动靶**,骨架里一律设为可配置,实时以 `exchangeInfo`(Binance)/ `public/instruments`(OKX)校验,**不硬编码**。

## 通用可靠性清单(逐项落地)

- [ ] **重连退避+抖动**:`delay = rand(0, min(cap, base*2^attempt))`,base≈0.5~1s、cap≈30~60s;**成功后 attempt 清零**;连续失败 N 次告警。绝不裸 `while(true)` 立即重连(雪崩+封 IP)。
- [ ] **心跳看门狗**:维护 `last_rx_ts`(用 `monotonic()` 非墙钟),`now - last_rx_ts > 心跳间隔×2~3` 即判死强制重连;ping 发出还要确认收到 pong。防 TCP 半开「假死」。
- [ ] **重订阅**:声明式「期望集合 → 收敛」,重连后整集重订并**校验 ack**;重订阅突发也走令牌桶,别一次几百条撞限频。
- [ ] **序列缺口重快照**:`next == prev+1`(或交易所衔接规则);缺口→丢弃增量→重拉 snapshot→重放缓冲增量;**重快照本身也要退避+限频**(防死循环);支持 OKX/Kraken CRC32 checksum 主动比对。
- [ ] **本地订单簿**:先订增量入 buffer→拉 snapshot→丢弃 `lastUpdateId` 前的增量→衔接;`qty==0` 删档不置零;价格转**定点整数**当 key(防浮点精度);定期 checksum 兜底。
- [ ] **令牌桶限频**:按 **weight 非请求数**计费,卡 80% 上限;读响应头已用权重做闭环;多进程共享 IP 需跨进程共享桶(如 Redis);下单 rate 与请求 weight 是两套独立配额。
- [ ] **时钟漂移**:周期拉 server time、扣 RTT 单程估 offset、监控漂移告警;签名统一用校准时间;`recvWindow` 别设太大(放弃重放保护)。
- [ ] **幂等下单**:每单唯一 `clientOrderId`;超时**先按 cloid 查单**再决定是否重发;撤改单同样幂等+查证。
- [ ] **错误分级**:瞬时可重试(超时/5xx/429/维护)退避重试;业务致命(余额/参数/签名/权限)立即上报不重试;**下单超时=不确定**走查证路径。显式映射表,别一刀切。
- [ ] **5xx 安全**:5xx/超时**绝不当确定失败**——可能已成交,盲目重发=双倍仓位;凭 clientOrderId 查证真实状态再动作。
- [ ] **优雅关闭**:捕 SIGTERM/SIGINT,有序停:拒新意图→处理在途挂单(撤/交接)→退订关 WS→flush 日志/落盘(last seq、簿快照)→释放句柄;设超时兜底强退+告警。
- [ ] **testnet/live 切换**:endpoint+key 按环境物理隔离,单一 `ENV` 切换,**默认 testnet**,live 须显式声明+醒目告警;日志打印当前环境。

## 输出格式

### 生成的连接器骨架结构

```
exchange_connector/
├── config.py            # ENV 枚举、endpoint 映射(testnet/live)、令牌桶配额(标 TODO 以 exchangeInfo 为准)
├── credentials.py       # os.environ 读 key/secret/passphrase,fail-fast,mask() 脱敏
├── clock.py             # server time 同步、offset 维护、漂移告警
├── ratelimit.py         # 按 weight 计费的异步令牌桶
├── ws_market.py         # 行情 WS:合并流订阅 + 期望集合 + ping-pong 看门狗 + 退避重连 + 重订阅
├── orderbook.py         # snapshot+增量衔接、序列缺口检测、qty==0 删档、checksum
├── ws_user.py           # 用户数据流:listenKey 续期 / OKX login + 断线后 REST 对账
├── rest_client.py       # 签名、下单(clientOrderId 幂等)、5xx 查证、错误分级映射表
├── reconnect.py         # 指数退避+满抖动、心跳看门狗(monotonic)
├── shutdown.py          # SIGTERM/SIGINT 优雅关闭
└── README.md            # 跑起来需要哪些 env、testnet→live 切换步骤、TODO 清单
```

> 实际可按语言(Python 异步 / C++)和「只要行情」需求裁剪。每个易变项标 `# TODO`,不写死。

### 连接器自检清单(交付时附上)

一份逐条勾选的 Markdown 清单,覆盖上面「通用可靠性清单」全部条目 + 该交易所要点表里的命中项,标注**已实现 / TODO / 不适用**,让用户(或 code-reviewer)一眼看清地基哪里还漏。

## 安全与免责

- **密钥安全(最高级)**:绝不在骨架里写死任何真实 key/secret/passphrase——一律 env 注入 + fail-fast + 日志脱敏。用户要你「直接填上密钥」时**拒绝**,只给注入位。最小权限 + IP 白名单是即使泄露也限损的纵深防御。
- **testnet 先行**:任何下单/续期/错误流程先在测试网跑通;骨架默认 testnet,切 live 须显式确认。测试网深度浅、成交假,别拿它的滑点当实盘验证。
- **这是工程脚手架,不保证盈利**:connector-forge 锻造的是**可靠的地基**,不含也不负责策略 alpha、信号或调参——那是策略层的事(本 skill 明确不做)。
- **低延迟需配合 latency-audit**:连接器有 `tick→signal→order` 热路径(TCP_NODELAY、连接复用、避免热路径堆分配/锁/文本 JSON)。本 skill 只提醒「记得过一遍」,微秒级尾延迟体检和 `文件:行` 定位交给 **latency-audit** skill。
- **高危提醒**:切 live、改 IP 白名单、动实盘下单/风控逻辑前,先隔离测试 + 备份;这些都是会动真金白银的操作。