<div align="center">

# 🎯 Latency Hunter Toolkit

**给一人量化的工程审查插件包 —— 无 alpha、无密钥、无模型,edge 你自己带。**

[![License: MIT](https://img.shields.io/badge/License-MIT-22c55e.svg)](LICENSE)
[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-plugin-d97757.svg)](https://docs.claude.com/en/docs/claude-code)
[![version](https://img.shields.io/badge/version-0.2.0-3b82f6.svg)](#路线图)
[![skills](https://img.shields.io/badge/skills-3-8b5cf6.svg)](#三个-skill)

**中文 | [English](README.en.md)**

</div>

---

> 回测虚高 0.x 个夏普的坑,实盘会一次性还给你。
> 热路径上多一把锁、一次 false sharing、一回堆分配,尾延迟会替你买单。
> 这个工具不帮你找 alpha,只帮你**别在工程上自欺**。

## 这是什么

**Latency Hunter Toolkit** 是一个面向**一人量化(OPC)**的 Claude Code **插件包**。作者人设 "Latency Hunter" —— 一个专盯延迟与陷阱、对"乐观假设"零容忍的猎手。

它**不是策略库、不是信号源、不是模型**。它做且只做一件事:**对你的量化工程代码做尽职调查(due diligence)**,把会让回测虚高、让实盘亏钱、让热路径变慢的工程陷阱逐项揪出来,按严重度出一张"体检报告"。

核心 ethos:

> **无 alpha · 无密钥 · 无模型 —— 只做工程审查,edge 你自己带。**

它不碰你的 API key,不预测收益,不给买卖点。它只回答三个问题:

- 这个值,**在该时点真能算出来吗**?(未来函数 / 前视偏差)
- 这笔成交,**实盘真能成吗**?(成交真实性 / 成本)
- 这段热路径,**为什么慢**?(锁争用 / false sharing / cache miss / 分配 / syscall)

当前含三个 skill:两个**审查型**(backtest-guard / latency-audit,按"严重度分级 + 定位到 `文件:行` + 不臆测"的铁律工作)+ 一个**生成型**(connector-forge,把"连接器锻造协议"固化成可运行骨架,让连接器从地基上就不漏)。

## 一键安装

### 方式一:Claude Code 插件市场(推荐)

```bash
# 1. 添加本仓库为插件市场
/plugin marketplace add paidaxing1234/latency-hunter-toolkit

# 2. 安装插件(包含三个 skill + 三个 slash command)
/plugin install latency-hunter
```

装完后,三个 skill 会在你聊到回测/热路径/连接器相关话题时**自动触发**,也可以用 slash command 显式调用:

```bash
/backtest-guard   ./strategy        # 审一份回测/策略目录
/latency-audit    ./engine/src      # 审一段 C++ 热路径
/connector-forge  ./my-bot          # 生成/修复一个交易所连接器骨架
```

### 方式二:手动拷贝到 skills 目录(备选)

如果你不走插件市场,只想要 skill 本体:

```bash
# 克隆仓库
git clone https://github.com/paidaxing1234/latency-hunter-toolkit.git

# 拷贝三个 skill 到你的用户级 skills 目录
mkdir -p ~/.claude/skills
cp -r latency-hunter-toolkit/skills/backtest-guard  ~/.claude/skills/
cp -r latency-hunter-toolkit/skills/latency-audit   ~/.claude/skills/
cp -r latency-hunter-toolkit/skills/connector-forge ~/.claude/skills/
```

重启 Claude Code 后,直接说"帮我审一下这个回测有没有未来函数""看看这段热路径为什么慢"或"帮我生成一个 Binance 连接器 / WS 老掉线帮我加重连"即可触发。

## 三个 skill

### 1. `backtest-guard` · 回测照妖镜

> 把一份回测/策略代码当"嫌疑犯"过堂,逐项揪出会让回测虚高、实盘还回去的陷阱。

**是什么** —— 量化回测代码的"照妖镜"。回测的默认假设是**乐观**的(零成本、零延迟、能看到未来、永远成交);实盘的默认假设是**敌意**的。这个 skill 的工作就是把每一处"乐观假设"暴露出来,标注严重度,而**不是替你优化收益**。

覆盖四大类、共 **48 条**检测点(完整清单见 `skills/backtest-guard/references/backtest-pitfalls.md`):

| 类别 | 抓什么 |
|------|--------|
| ① 未来函数 / 数据泄漏 | 负数 `shift`、同 bar 信号即成交偷价、全样本 `fit` 预处理、`bfill` 灌未来值、`center=True` 滚动、repaint 指标、point-in-time 缺失、时序 shuffle 切分… |
| ② 过拟合 / 数据窥探 / 幸存者偏差 | 无样本外、网格只报最优参、p-hacking、用"当前"成分股回测历史、自由参数过多、样本量不足、无多重检验校正… |
| ③ 成交真实性 / 成本执行 | 零手续费、零滑点、用 bar 内极值当成交价、假设无限流动性、零延迟限价即成、忽略爆仓/资金费率/做空成本… |
| ④ 收益统计 / 口径错误 | 年化因子用错(crypto 套 252)、算术累加冒充复利、夏普分母口径错、权益曲线 realized/MTM 混用… |

**怎么触发**

- 自动:把策略/回测/信号/数据加载/参数寻优代码丢过来,问"这个回测能信吗 / 为什么实盘对不上回测 / 帮我查未来函数 / 有没有过拟合"。
- 触发词(中):回测审查、回测照妖镜、回测体检、未来函数、前视偏差、数据泄漏、过拟合、数据窥探、幸存者偏差、回测陷阱、回测可信度、实盘对不上回测。
- 触发词(英):backtest audit、look-ahead bias、data leakage、overfitting、survivorship bias、curve fitting。
- 显式:`/backtest-guard ./your-strategy`

**示例体检报告(片段)**

```
══════════════════════════════════════════
   回测体检报告 · backtest-guard
══════════════════════════════════════════
受检对象: my-strategy/     扫描文件: 12 个     代码行: ~3,400

总评: 🔴 致命 2 项 · 🟠 高危 2 项 · 🟡 中 1 项 · 🔵 低 0 项
判语: 「这更像一次对 2021 牛市的精细过拟合,而不是一个策略。」

[🔴 致命] 负数 shift 引入未来值
  位置: src/features.py:84
  问题: df['mom'] = df['close'].shift(-5) 把 t+5 收盘移到 t,
        该列进入了 X 特征矩阵(features.py:131)。
  修复: 进特征的列改 shift(>=1);未来收益只留 y/label,绝不进 X。

[🟠 高危] 零手续费 + 零滑点
  位置: backtest/engine.py:––(全文件未发现 commission/slippage)
  问题: 成交按 close 原值记账,日内策略年换手 ~900 次,成本被完全忽略。
  修复: 按 taker 费率建模 + 半个 spread 滑点,做成本翻倍敏感性。
══════════════════════════════════════════
```

> 报告只给**方向性判断**("剔除偏差后表现往哪边走"),**绝不承诺任何具体收益数字**。每条都定位到 `文件:行`,找不到证据就标"未发现 / 需人工确认"。

### 2. `latency-audit` · 热路径猎手

> 把 C++ 低延迟交易引擎的热路径放上手术台,逐项揪出偷走微秒、抖动尾延迟的工程缺陷。

**是什么** —— C++ 低延迟/高频交易引擎热路径的审查器。它先定位真正的热路径(tick 回调 → 信号 → 下单 → 撮合 → 序列化 → 无锁队列),再逐项检查那些在均值上看不出、却在 **p99/p99.9 尾延迟**上要命的工程问题,出一张"热路径体检报告"。

覆盖三大类:

| 类别 | 抓什么 |
|------|--------|
| ① 并发 / 锁 / 伪共享 | 热路径上的 `mutex`/锁争用、false sharing(未对齐到 cache line 的共享计数器)、虚假唤醒、原子操作内存序过严… |
| ② 内存 / 缓存 / 分配 | 热路径堆分配(`new`/`malloc`/隐式 `std::string`/`std::vector` 扩容)、cache miss、指针追逐、未预留容量、拷贝而非 zero-copy… |
| ③ syscall / IO / 序列化 / 网络 / 分支 | 热路径 syscall、未批处理的网络 IO、序列化拷贝、不可预测分支、缺 `__builtin_expect`/`[[likely]]`、日志阻塞热路径… |

**怎么触发**

- 自动:把 C++ 低延迟/HFT 引擎热路径代码丢过来,问"这段热路径为什么慢 / 帮我查锁争用 / 有没有 false sharing / 哪里在偷偷堆分配"。
- 触发词(中):热路径审查、热路径猎手、低延迟审查、锁争用、伪共享、缓存未命中、堆分配、零拷贝、尾延迟。
- 触发词(英):latency audit、hot path、lock contention、false sharing、cache miss、zero-copy、tail latency。
- 显式:`/latency-audit ./engine/src`

**示例体检报告(片段)**

```
══════════════════════════════════════════
   热路径体检报告 · latency-audit
══════════════════════════════════════════
受检对象: engine/src/   热路径: on_tick → signal → send_order

总评: 🔴 致命 1 项 · 🟠 高危 2 项 · 🟡 中 1 项
判语: 「均值看着漂亮,p99.9 在替这把锁和这次堆分配还债。」

[🔴 致命] 热路径上持锁
  位置: engine/src/book.cpp:212
  问题: on_tick() 内 std::lock_guard 锁住整个 order book 更新,
        与撮合线程争用,tick 风暴下尾延迟抖动。
  修复: 改 SPSC 无锁队列 / 序列号发布,把锁移出热路径。

[🟠 高危] false sharing(共享计数器未对齐 cache line)
  位置: engine/src/stats.hpp:30
  问题: 多线程频繁写的 counters[] 同处一条 cache line,引发缓存行乒乓。
  修复: alignas(64) 每计数器独占一行,或线程本地累加后汇总。
══════════════════════════════════════════
```

> 报告建议用 `perf` / `clang-tidy` 实测验证,**只指方向、绝不承诺具体微秒数**。是不是真热路径、改完快多少,以你机器上的实测为准。

### 3. `connector-forge` · 连接器锻造

> 把"怎么从地基上锻造一个不漏的连接器"固化成可执行协议,一键生成能跑起来的 Binance / OKX 行情+交易连接器骨架(或审已有连接器的可靠性缺口)。

**是什么** —— 前两个 skill 是"审查型"(挑刺),这一个是**"生成型"脚手架**。连接器是量化系统的**地基**:它不直接赚钱,但**一漏就全盘崩**——90% 的"实盘对不上回测""莫名其妙断线""下单偶发失败"都出在这一层,而且都是**静默故障**(连接看似健康,数据却早已是死的或错的)。这个 skill 把一条铁律固化下来:**宁可显式判死重建,绝不带病运行**。

它生成 / 修复的东西,逐项是工程而非 alpha:

| 维度 | 锻造内容 |
|------|----------|
| ① 连不上 / 老掉线 | 指数退避 + 满抖动重连、心跳**看门狗**(单调时钟判死,防 TCP 半开假死)、Binance 24h 强断提前重连、OKX `'ping'/'pong'` 应用层心跳 |
| ② 数据对不上 | 重连后按"期望订阅集合"**整集重订并校验 ack**、带序列号增量**逐条校验连续性**、缺口即丢弃→重拉 snapshot 重放、本地订单簿 `qty==0` 删档 + checksum 兜底 |
| ③ 签名 / 时间报错 | Binance HMAC-SHA256(query+body)/ OKX prehash→Base64 + passphrase、基于**校准后交易所时间**的时间戳(防 -1021 / 50102)、`listenKey` 每 30min 续期 / OKX `login` 续期 |
| ④ 限频 / 幂等 / 安全 | 按 **weight 非请求数**的令牌桶卡 80% 上限、每单 `clientOrderId` 幂等 + 超时"先查证后重发"、密钥一律走 **env** 注入 + fail-fast + 日志脱敏(绝不写死) |

**怎么触发**

- 自动:说"帮我生成 / 写一个 Binance/OKX 连接器""搭个行情订阅骨架""WS 老掉线帮我加重连",或报故障"签名报错 -1022/50113""时间戳报错 -1021/50102""listenKey 过期收不到成交""重连后收不到数据""本地订单簿和交易所对不上"。
- 触发词(中):连接器、连接器锻造、交易所对接、Binance/OKX 连接器、行情订阅、断线重连、老掉线、签名报错、时间戳报错、listenKey 续期、重订阅、序列缺口、本地订单簿、实盘对不上。
- 触发词(英):connector、connector-forge、exchange connector、market data feed、websocket reconnect、heartbeat、listenKey、ws login、orderbook snapshot、sequence gap、rate limit token bucket、clock drift。
- 显式:`/connector-forge ./my-bot`

**生成的连接器骨架(目录结构)**

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

> 交付时附一份逐条勾选的**连接器自检清单**(已实现 / TODO / 不适用),让你或 code-reviewer 一眼看清地基哪里还漏。密钥安全是最高级:**绝不写死**,只给 env 注入位 + fail-fast;骨架**默认 testnet**,切 live 须显式确认。两大交易所的端点路由、签名、限频、续期等**完整模板见 `skills/connector-forge/references/connector-blueprint.md`**。

## 目录结构

```
latency-hunter-toolkit/
├── .claude-plugin/
│   ├── plugin.json           # 插件清单(name / version / 描述)
│   └── marketplace.json      # 插件市场清单(供 /plugin marketplace add 发现)
├── commands/
│   ├── backtest-guard.md     # /backtest-guard slash command
│   ├── latency-audit.md      # /latency-audit slash command
│   └── connector-forge.md    # /connector-forge slash command
├── skills/
│   ├── backtest-guard/
│   │   ├── SKILL.md                          # 回测照妖镜:协议 + 四大类陷阱 + 报告模板
│   │   └── references/
│   │       └── backtest-pitfalls.md          # 48 条陷阱百科(why → smell → detect → fix)
│   ├── latency-audit/
│   │   ├── SKILL.md                          # 热路径猎手:协议 + 三大类 + 报告模板
│   │   └── references/                       # 热路径陷阱参考
│   └── connector-forge/
│       ├── SKILL.md                          # 连接器锻造:六步协议 + 两大所要点 + 可靠性清单
│       └── references/
│           └── connector-blueprint.md        # Binance/OKX 连接器完整模板(端点/签名/限频/续期)
├── LICENSE                   # MIT
├── README.md                 # 中文(本文)
└── README.en.md              # English
```

## 路线图

`v0.2.0` 在地基上又夯了一层:两个**审查型** skill(backtest-guard / latency-audit)+ 一个**生成型** skill(connector-forge)。后续按"一人量化全链路工程审查"的思路继续补,**仍然坚持无 alpha、无密钥、无模型**:

- [x] **`connector-forge`(连接器锻造)** —— ✅ 已交付(v0.2.0):一键生成/修复生产级 Binance·OKX 行情+交易连接器骨架,WebSocket 重连与心跳、限频令牌桶、签名与时间同步、listenKey/login 续期、重订阅、序列缺口重快照、密钥走 env;也能审已有连接器的可靠性缺口。
- [ ] **`risk-killswitch-guard`** —— 实盘风控/熔断逻辑审查:仓位上限、最大回撤熔断、单笔/单日亏损阈值、kill switch 可达性。
- [ ] **`order-state-machine-audit`** —— 订单全生命周期状态机审查:挂单/部成/撤单/超时的竞态与漏态。
- [ ] **`orderbook-sanity`** —— 本地订单簿一致性审查:snapshot/增量衔接、序列缺口、checksum 校验、定点价格 key、删档逻辑。
- [ ] **`risk-config-lint`** —— 风控配置静态检查:杠杆/仓位/止损阈值是否缺项、互相矛盾或被写死成危险默认值。
- [ ] **`data-pipeline-integrity`** —— 行情/因子数据管线完整性:缺口、时区/夏令时、复权口径、point-in-time 落库。

有想看的 skill,欢迎提 issue。猎手永远在找下一个延迟。

## 合规与免责

**Latency Hunter Toolkit 是一套工程审查工具,不是投资工具。**

- **非投资建议**:本工具**不预测收益、不评估策略盈利能力、不荐股、不给买卖点或仓位建议**。"通过审查" ≠ "策略能赚钱",只代表未发现本清单覆盖的工程陷阱。
- **不承诺任何数字**:报告中的"偏差影响""尾延迟"等表述仅为**方向性判断**(通常使样本外表现低于回测 / 指出潜在延迟来源),**绝不承诺、不量化任何具体收益、亏损或微秒数**。延迟数字一律以你机器上的 `perf`/基准实测为准。
- **审查有边界**:结论依赖所提供的代码与可见上下文;未提供的数据、外部依赖、市场结构变化中的风险本工具无法发现。
- **风险自负**:一切实盘交易决策与资金风险由使用者自行承担。投入实盘资金、加杠杆等高危操作前,务必小资金隔离验证。

## License

[MIT](LICENSE) © 2026 paidaxing (Latency Hunter)

---

<div align="center">

*No alpha. No keys. No models. Just the engineering review that keeps your edge honest.*

</div>
