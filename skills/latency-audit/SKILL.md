---
name: latency-audit
description: >-
  C++ 低延迟/高频交易引擎的「热路径猎手」。逐项审查会拖慢热路径、放大尾延迟(P99/P99.9)的工程坑——锁争用、false sharing、cache miss、热路径分配、syscall、序列化、分支预测失败、NUMA 跨节点、TLB miss、PyBind11 GIL——并定位到 文件:行,按严重度(致命/高危/中/低)输出一张「热路径体检报告」。
  触发词(中):热路径审查、低延迟审查、延迟猎手、锁争用、伪共享、缓存未命中、尾延迟、热路径分配、NUMA 亲和、TLB miss、GIL 争用、热路径体检。
  触发词(英):latency audit、hotpath、hot path、low-latency、false sharing、cache miss、lock contention、tail latency、NUMA、TLB miss、huge page、pybind11 GIL、latency-audit。
  适用:用户把 tick 回调/撮合/下单/序列化/无锁队列/PyBind11 混合策略等 C++ 热路径代码丢过来,问「这段会不会拖慢热路径/为什么尾延迟突刺/帮我查锁和分配/有没有 false sharing/NUMA 绑对了吗」。
  不适用:纯功能 bug(逻辑错、崩溃)、非性能问题、非 C++ 代码、要求承诺具体延迟微秒数字或替代真实 benchmark/profiling。
license: MIT
---

# latency-audit / 热路径猎手

## 核心理念:尾延迟是钱

在高频交易里,**P99.9 尾延迟不是统计噱头,是真金白银**。热路径上每一个锁、每一次堆分配、每一个 syscall、每一份多余拷贝,实盘都用**滑点、排队靠后、错失成交**还回来。均值(mean)漂亮没用——99% 的 tick 跑得飞快、偶尔一次 brk/futex/磁盘抖动/跨 NUMA 访问/TLB miss 就把那一笔订单的延迟从微秒打到毫秒,而那一笔往往就是行情剧烈、最该抢到的成交。

本 skill 是一把**热路径照妖镜**:不替你写引擎,而是逐项揪出「会拖慢热路径、增大尾延迟」的工程坑,定位到 `文件:行`,按严重度排出修复优先级。

**铁律:本工具只做静态工程审查,不替代实测。** 所有结论都应回到 `perf` / `clang-tidy` / `perf c2c` / `numastat` / 火焰图 上验证——代码看着像坑不等于它在你的负载下真是瓶颈,优化要由 profiling 数据驱动,本报告**不承诺任何具体微秒数字**。

## 何时启用

- 用户丢来 tick 回调 / 行情解析 / 撮合 / 信号 / 下单 / 序列化 / 无锁队列 / PyBind11 混合策略 等 C++ 代码,问「这段热路径有没有坑 / 为什么尾延迟突刺 / 帮我查锁和分配 / NUMA 绑对了吗」。
- 触发词:热路径审查、低延迟审查、锁争用、false sharing、cache miss、尾延迟突刺、热路径分配、NUMA 亲和、TLB miss、GIL 争用、`latency audit`、`hotpath`、`low-latency`。
- code review 阶段想专门过一遍「性能/延迟」维度(功能正确性另用通用 code review)。

**不启用**:纯功能 bug、逻辑错误、崩溃排查;非 C++ 代码;要你承诺「这能跑到 X 微秒」或拿它替代真实 benchmark。

## 审查协议

1. **先圈热路径**。明确哪些函数在关键路径上、每 tick/每消息都跑:`on_tick` / `on_md` / `on_message` / `handle_quote` / 撮合 `match` / 信号 `on_signal` / 下单 `send_order` / 序列化 `encode`/`decode` / 无锁队列 `push`/`pop` / PyBind11 导出的热回调。**只在热路径上较真**——冷路径(配置加载、日志落盘、对账、启动初始化)用 mutex、分配、格式化都允许,别误报。
2. **逐类过三大陷阱**。对圈出的热路径,依次走「并发/锁/伪共享/NUMA/GIL」「内存/缓存/分配/TLB」「syscall/IO/序列化/网络/分支」三类清单(详见 `references/latency-pitfalls.md`)。
3. **定位到 文件:行,给可验证的检测手段**。每条命中都要落到具体 `文件:行`,并给出「怎么用工具坐实」——`grep` 模式、`clang-tidy` check、`perf`/`perf c2c`/`perf lock`/`numastat`/`perf stat -e dTLB-load-misses`/火焰图 的观测点。**基于真实代码定位,不臆测**;吃不准是不是热路径就标注「需确认是否在热路径」。
4. **按严重度汇总**,输出「热路径体检报告」。
5. **收尾强调实测**。报告末尾固定提示:用 `perf` 实测 P99/P99.9、用 `perf c2c` 验 false sharing、用 `numastat`/`numactl --hardware` 验 NUMA 落点、用 `perf stat` 看 TLB miss、用火焰图定位真实热点,**代码审查只负责找嫌疑、不下延迟判决**。

## 三大类陷阱

完整清单(每条含 why / detectHint / fix)见 `references/latency-pitfalls.md`,下面是每类最致命的速查。

### ① 并发 / 锁 / 伪共享 / NUMA / GIL(concurrency)

- **【致命】热路径上 `std::mutex` 锁争用** — 争用即陷内核(futex),上下文切换 1-5μs,被抢占可等整个调度周期(ms 级),P99.9 直接炸。热路径单生产单消费改 SPSC ring,跨线程状态用 atomic acquire/release。
- **【致命】持锁执行 I/O / 耗时操作** — 临界区内 `send`/`redisCommand`/`new`/格式化,把锁持有时间从 ns 拉到 ms,拖死所有等锁线程。锁内只做「换指针/拷快照」,I/O 甩给旁路线程。
- **【致命】NUMA 跨节点访问 / 线程-内存-网卡中断未同节点亲和** — 多路服务器上,交易线程跑在 node0 却访问分配在 node1 的对象池/订单簿,远端内存读延迟近乎翻倍(数十 ns 量级上探),`first-touch` 默认策略常把内存分配到错误 node;网卡 DMA/中断落在非交易 node 还要跨 QPI/UPI 搬运。隐蔽且只在多 socket 机器上爆。把交易线程、其内存、网卡中断**全部绑到同一 NUMA node**:`numactl --cpunodebind=0 --membind=0`,启动期在目标线程上 first-touch 预触摸所有热内存,网卡中断亲和(`/proc/irq/*/smp_affinity`)对齐到本 node。验证:`numastat -p <pid>` 看 `numa_miss`/`other_node`,`perf c2c` 看 remote HITM。
- **【致命】PyBind11 / 嵌入式 Python 在热路径持有或争用 GIL** — C++/Python 混合引擎里,若 tick 回调里回调进 Python、或 C++ 热段未 `gil_scoped_release` 就跑长逻辑,等于全程握着一把**进程级全局锁**,所有线程被 Python 解释器串行化,彻底毁掉多核与确定性。热路径纯 C++ 段必须 `py::gil_scoped_release`;绝不在 tick/撮合回调里同步调 Python;跨语言只传 POD/句柄,Python 只在冷路径(参数下发、收盘统计)介入。验证:`grep -n gil_scoped_acquire|PyEval_|Py_BEGIN`,火焰图看热路径下挂 `take_gil`/`PyEval_EvalFrame`。
- **【高危】`std::queue`/`std::deque` + mutex 当跨线程队列** — deque 每次 push 可能堆分配,叠加锁=双重杀手。SPSC 通道换定长 2 的幂数组 ring buffer。
- **【高危】false sharing:多线程高频写挤同一 cache line** — ring 的 head/tail、各线程计数器若同 line,MESI ping-pong 几十~上百 ns。字段加 `alignas(64)` 隔到独立 cache line,`perf c2c` 看 HITM 坐实。
- **【高危】热路径 `condition_variable` 阻塞唤醒** — wait/notify 走 futex,唤醒延迟微秒级且受调度影响。超低延迟通道改 busy-poll,或 spin-then-block 混合。
- **【高危】跨线程传 `shared_ptr` 引用计数原子争用** — 控制块原子 inc/dec = 隐藏 false sharing,热路径按值传更触发拷贝抖动。改 `const&`/裸指针/对象池索引。
- **【高危】关键线程不绑核 / 频繁创建销毁 `std::thread`** — 不绑核被调度器核间迁移,迁移即丢 L1/L2/TLB,造缓存尖刺。关键线程启动期常驻 + `pthread_setaffinity_np` 绑独占核,配合 `isolcpus`/`nohz_full`/`rcu_nocbs`,必要时 `SCHED_FIFO`。
- **【中】自旋忙等退避策略不当** — 裸空转饿死 SMT 兄弟核 + 烧功耗,过度 `yield` 引入调度抖动。分级退避:短自旋带 `_mm_pause()` → 更多自旋 → 必要时 yield;独占核可纯 busy-poll(注意是延迟换 CPU 功耗的取舍)。
- **【中】`std::atomic` 默认 seq_cst** — x86 上 seq_cst store 插 mfence(几十 ns),多数场景只需 acquire/release,纯计数用 relaxed。显式写 memory_order。

### ② 内存 / 缓存 / 分配 / TLB(memory)

- **【致命】热路径动态分配 `new`/`malloc`** — 走 ptmalloc arena 锁、free-list,甚至触发 mmap/brk + 缺页,数十~数百 ns 且不可预测,P99.9 突刺头号杀手。零分配热路径:启动期对象池预分配,运行期 RSS 不增长。
- **【高危】`std::string`/`std::vector` 热路径反复扩容拷贝** — 容量不足即 realloc + memcpy,SSO 超 ~15/22 字节落堆。事前 `reserve(maxSize)`,定长用 `std::array`,复用 buffer(`clear()` 不释放容量)。
- **【高危】热路径虚函数 / vtable 指针追逐** — vptr 加载可能 cache miss,间接跳转破坏分支预测、阻止内联。改 CRTP 静态多态 / `std::variant`+visit,虚函数留冷路径。
- **【高危】`std::map`/`std::unordered_map` 热路径查找** — 红黑树/链式哈希指针追逐,节点散落堆上,几乎每跳一次 node 一次 cache miss。换 flat hash(`absl::flat_hash_map`/`robin_hood`),整型 key 直接数组索引。
- **【高危】按值传大对象 / 不必要拷贝** — 按值传 `vector`/大 struct = 深拷贝 + 分配。只读用 `const&`/`string_view`/`span`,转移用 `move`,range-for 用 `const auto&`。小 POD(<=16B)按值反而更好,别一刀切。
- **【高危】指针追逐数据结构(链表/树)** — `std::list`/侵入链表每跳一节点一次依赖性 load,CPU 无法预取,N 节点 = N 次串行 miss。连续存储优先,必要时 `__builtin_prefetch`。
- **【高危】AoS 布局 / 冷热数据混排** — 热循环只读少数字段却撑满 cache line。批量遍历改 SoA,冷热字段分离,`pahole` 查布局。随机访问单条记录时 AoS 更好,按访问模式选。
- **【高危】大热表 TLB miss / 未用 huge page** — 几十 MB~GB 级的对象池、订单簿、行情索引用默认 4K 页时,工作集页数远超 TLB 表项(L1 dTLB 仅数十~上百项),每次跨页访问触发 TLB miss + page walk(数十 ns),大表随机访问尤甚。热大表用显式 huge page(2M/1G:`mmap(MAP_HUGETLB)` 或 hugetlbfs;别依赖透明大页 THP,其 khugepaged 合并会带来抖动,延迟敏感场景常显式关 THP 改显式 hugetlb),配 `mlockall(MCL_CURRENT|MCL_FUTURE)` 锁页防换出 + 启动期预触摸。验证:`perf stat -e dTLB-load-misses,dtlb_load_misses.walk_active` 看 page walk 占比。
- **【中】未对齐 / 跨 cache line 的热结构** — 起始地址未按 64B 对齐或大小跨 line,一次访问横跨两条 line。热 struct 加 `alignas(64)`,ring slot padding 到 line 整数倍,`pahole` 查空洞。
- **【中】`noexcept` 缺失 / 热路径异常控制流** — throw 栈展开微秒级毁尾延迟;缺 noexcept 阻止 vector move 优化。热路径用返回码/`std::expected`,关键函数标 noexcept。

### ③ syscall / IO / 序列化 / 网络 / 分支(io)

- **【致命】热路径同步日志落盘** — 每条日志一次 `write`/`fsync`,磁盘抖动打到 ms 级,P99 突刺最常见元凶。改无锁异步日志(spdlog async / NanoLog),热路径只把二进制参数压环,后台线程格式化。
- **【致命】未设 `TCP_NODELAY` / Nagle 生效** — 小包被缓冲攒 MSS 或等 ACK,叠加 delayed ACK 最高 ~40ms 停顿,极隐蔽。socket 建好立即 `setsockopt(TCP_NODELAY)`,ZMQ 设 `ZMQ_TCP_NODELAY`。
- **【致命】同步阻塞 I/O 在热路径** — 阻塞 `recv`/同步 `redisCommand`/同步 DB,交易线程陷内核睡眠,恢复 cache 全冷。全异步非阻塞:`O_NONBLOCK` + epoll(ET)/io_uring,Redis 异步 API 或移出热路径。
- **【高危】JSON 文本序列化/反序列化在热路径** — DOM 库大量小对象分配 + `strtod`,`nlohmann::json` 尤慢。优先二进制协议(SBE/FlatBuffers/Cap'n Proto),必须 JSON 用 `simdjson`+`fast_float`。
- **【高危】热路径字符串格式化拼接** — `to_string`/`sprintf`/`fmt::format` 触发分配 + locale,数百 ns。订单/symbol 用整型 ID,FIX 用预分配 buffer + 手写 itoa/ftoa。
- **【高危】高频时间戳 syscall** — 每 tick `clock_gettime`/`system_clock::now()`,clocksource 退化成非 tsc 时变真 syscall。热路径用 RDTSC/RDTSCP(启动校准),确认 `clocksource=tsc` 且 CPU 有 `constant_tsc`/`nonstop_tsc`。
- **【高危】WebSocket/ZMQ 每条消息多次内存拷贝** — recv→string→message→queue 4-5 次 memcpy,吃满内存带宽污染 cache。全链路 zero-copy:`string_view`/`span` 建视图,ZMQ `zmq_msg_init_data`,`recvmmsg` 批量。
- **【高危】缺批量处理 / 每消息一次 syscall** — syscall 次数 = 消息数,每次 100-300ns 切换。`recvmmsg`/`sendmmsg` 批量,epoll 一次处理多个 fd,应用层攒批。
- **【中】分支预测失败 / 缺 likely/unlikely** — 罕见路径(风控拒单、异常)平铺成普通 if,误预测清流水线 ~15-20 cycle。冷路径标 `[[unlikely]]`+抽成 `noinline`,优先上 PGO/AutoFDO。
- **【中】关键小函数未内联 / 跨 TU 调用** — 小函数定义在 `.cpp` 无 LTO 不内联,付调用开销 + 阻断跨调用优化。热路径小函数 header inline,开 `-flto`。别盲目 `always_inline` 大函数(撑爆 I-cache 反更慢)。
- **【中】`std::endl` 强制 flush / iostream 在热路径** — `std::endl` 每次一次 `write` syscall + iostream 慢(locale/锁)。换行用 `'\n'`,诊断输出走异步日志。
- **【中】未用 busy-poll / 内核旁路** — 标准内核栈固有几微秒 + 唤醒抖动限制延迟下限。超低延迟用 DPDK/Onload/AF_XDP 用户态轮询(注意是延迟换 CPU 功耗,非全栈适用)。

## 热路径体检报告(固定输出模板)

> 可直接截图发群。严格按此结构,逐条带 `文件:行`,不堆术语。

```
═══════════════════════════════════════════
        热路径体检报告 / latency-audit
═══════════════════════════════════════════
审查范围:<文件/模块,圈出的热路径函数清单>
总评:🔴 危险 / 🟡 需整改 / 🟢 基本健康
致命 ✕N   高危 ✕N   中 ✕N   低 ✕N
一句判语:<例:撮合热路径上有一把 std::mutex + 一处 new,P99.9 极可能突刺,先拔锁去分配。>
───────────────────────────────────────────

【致命】热路径 std::mutex 锁争用
  📍 order_engine.cpp:142  on_tick() 临界区
  问题:每 tick 进 lock_guard 抢全局 g_book_mutex,争用陷 futex,
        被抢占可等整个调度周期,直接炸 P99.9。
  修复:该通道改 SPSC ring buffer;跨线程状态用 atomic
        acquire/release;锁挪到冷路径。
  验证:perf lock / perf record -e sched:sched_switch 看 futex_wait;
        火焰图出现 __lll_lock_wait 即坐实。

【致命】NUMA 跨节点访问:交易线程在 node0,对象池在 node1
  📍 engine_init.cpp:58  order_pool_ 在主线程构造,未 membind
  问题:对象池 first-touch 落在 node0,但交易线程绑到 node1 核,
        每次取单跨 socket 读远端内存,远端延迟近乎翻倍,尾延迟抬高。
  修复:numactl --cpunodebind=1 --membind=1 启动,或在交易线程内
        预触摸对象池;网卡中断亲和对齐到同一 node。
  验证:numastat -p <pid> 看 numa_miss/other_node;perf c2c 看 remote HITM。

【高危】false sharing:ring head/tail 同 cache line
  📍 spsc_queue.hpp:30-32  head_ / tail_ 紧邻无 padding
  问题:生产者写 tail_、消费者写 head_,落同一 64B line,
        MESI ping-pong 抬高尾延迟方差。
  修复:head_ / tail_ 各 alignas(64) 隔到独立 cache line。
  验证:perf c2c record/report 看 HITM(remote hit modified)。

【中】std::atomic 默认 seq_cst
  📍 spsc_queue.hpp:45  tail_.store(next)  未指定 memory_order
  问题:seq_cst store 在 x86 插 mfence,每 tick 累积开销。
  修复:发布用 release store,消费用 acquire load。
  验证:反汇编看热路径是否有多余 mfence。

───────────────────────────────────────────
⚠ 本报告为静态工程审查,只标嫌疑、不下延迟判决。
  请用 perf 实测 P99/P99.9、perf c2c 验 false sharing、
  numastat 验 NUMA 落点、perf stat 看 TLB miss、
  火焰图定位真实热点后再优化——优化由 profiling 数据驱动,
  本报告不承诺任何具体微秒数字。
═══════════════════════════════════════════
```

无命中时如实写「本轮未发现热路径级延迟坑」,并提示仍应实测验证,**不硬凑条目**。

## 严重度定义

| 严重度 | 含义 | 处置 |
|--------|------|------|
| 🔴 致命(fatal) | 热路径上必然引入不可预测的 μs~ms 级尾延迟尖刺(锁争用、热路径分配、同步落盘、阻塞 I/O、Nagle、NUMA 跨节点、持 GIL),实盘直接吃滑点/丢成交 | **上线前必须修** |
| 🟠 高危(high) | 显著抬高均值或尾延迟方差(false sharing、cache miss 结构、TLB miss、热路径拷贝/格式化、cv 阻塞、缺批量、不绑核) | **强烈建议修**,实测确认后优先处理 |
| 🟡 中(medium) | 累积可观但单次有限,或取舍型(seq_cst、未内联、分支预测、对齐、碎片、忙等退避) | 结合 profiling 评估收益再改 |
| ⚪ 低(low) | 风格/微优化,收益需实测佐证 | 可选,别为它牺牲可读性 |

> 严重度是**先验排序**,不是实测结论。某条「致命」若 profiling 证明根本不在瓶颈上,就该降级——**永远以 perf 数据为准**。

## 重要说明

- **这是工程审查工具,不是 profiler。** 它基于代码模式找「延迟嫌疑」,真正的瓶颈必须用 `perf` / `perf c2c` / `perf lock` / `numastat` / `perf stat -e dTLB-load-misses` / 火焰图 / `cachegrind` 在真实负载下实测确认。代码看着像坑 ≠ 它就是你的瓶颈。
- **不替代 benchmark。** 任何「修复」都要 A/B 实测(改前改后对比 P50/P99/P99.9),用数据说话,不靠直觉。
- **不承诺具体延迟数字。** 本 skill 永远不会说「这能跑到 X 微秒」「修完省 Y ns」——延迟取决于硬件、内核参数、NUMA 拓扑、网卡、负载,只有你的实测说了算。
- **只在热路径较真,冷路径放行。** 配置加载、日志落盘、对账、启动初始化里的 mutex / 分配 / 格式化都正常,别误报成致命。
- **修复有取舍成本。** busy-poll 烧 CPU 功耗、绑核占用整核、NUMA 亲和限制部署灵活性、huge page 需预留与运维配置、内核旁路抬高运维复杂度、零分配增加代码复杂度、`gil_scoped_release` 需保证不碰 Python 对象——本报告给方向,落地与否由你按 ROI 和实测收益定夺。
- **优先成熟轮子。** SPSC/MPMC 队列(moodycamel、Disruptor 思路)、flat hash(abseil、robin_hood)、异步日志(spdlog async、Quill、NanoLog)、JSON(simdjson)、分配器(jemalloc/tcmalloc/mimalloc)、NUMA 工具(libnuma/numactl)都有久经实盘的实现,优先复用而非手搓。

完整 40+ 条陷阱明细(每条含原理 / 检测手段 / 修复方案,含 NUMA 亲和、TLB/huge page、PyBind11 GIL 专条)见 `references/latency-pitfalls.md`。
