# 热路径延迟陷阱百科 (Hot-Path Latency Pitfalls)

> SKILL.md 指向的完整热路径陷阱参考。覆盖 HFT / 低延迟交易引擎在 **tick → signal → order** 关键路径上最常见的工程陷阱。
>
> 按三大类组织:**并发 (Concurrency)** · **内存 (Memory)** · **I/O (网络/磁盘/序列化)**。
>
> 每条陷阱结构:
> - **名称【严重度】** —— `fatal` 致命 / `high` 高危 / `medium` 中 / `low` 低
> - **why** —— 为什么慢(底层原理)
> - **codeSmell** —— 代码长啥样(C++ 反例片段)
> - **detectHint** —— 怎么检测(grep / clang-tidy / perf)
> - **fix** —— 怎么修(正确写法片段)
>
> 总原则:**热路径零分配、零锁、零 syscall、零拷贝、零文本格式化**。所有重活在开盘前预分配,运行期只做纳秒级操作。

---

## 目录

- [一、并发陷阱 (Concurrency)](#一并发陷阱-concurrency)
- [二、内存陷阱 (Memory)](#二内存陷阱-memory)
- [三、I/O 陷阱 (I/O · 网络 · 序列化)](#三io-陷阱-io--网络--序列化)
- [附录:严重度速查表](#附录严重度速查表)

---

## 一、并发陷阱 (Concurrency)

### 1. 热路径上使用 std::mutex 锁争用【fatal】

**why** —— `std::mutex` 在争用时会陷入内核(futex syscall),一次上下文切换 1-5μs,被抢占后可能等待整个调度周期(毫秒级)。在 tick→signal→order 热路径上,锁争用会让 P99.9 尾延迟从微秒级炸到毫秒级,是 HFT 引擎最致命的延迟来源。即使无争用,lock/unlock 也有原子操作 + 内存屏障开销。

**codeSmell**
```cpp
// 反例:每个 tick 都抢一把全局锁
std::mutex g_book_mutex;
void on_tick(const Tick& t) {
    std::lock_guard<std::mutex> lk(g_book_mutex);   // 热路径阻塞点
    order_book_.update(t);
    signal_ = strategy_.compute(order_book_);       // 还持锁做计算
}
```

**detectHint**
- `grep -nE 'std::(mutex|recursive_mutex|timed_mutex|shared_mutex|lock_guard|unique_lock|scoped_lock)'` 限定在 `on_tick/on_message/on_md/process/handle` 等热路径函数体内。
- clang-tidy 暂无专门 check,可用 AST matcher 找热路径函数内的 `lock_guard` 构造。
- `perf record -e sched:sched_switch` + `perf lock` 看 futex 争用与上下文切换。
- 火焰图里出现 `__lll_lock_wait` / `futex_wait` 即坐实。

**fix**
```cpp
// 单生产单消费数据通道:无锁 SPSC ring buffer
SpscRing<Tick, 1u << 16> md_ring_;          // 定长 2 的幂,位掩码取模

// 跨线程状态:atomic + acquire/release,而非互斥
std::atomic<Signal> latest_signal_;
latest_signal_.store(sig, std::memory_order_release);   // 生产者
Signal s = latest_signal_.load(std::memory_order_acquire); // 消费者

// 极短临界区(非热路径)才允许自旋
std::atomic_flag spin = ATOMIC_FLAG_INIT;
while (spin.test_and_set(std::memory_order_acquire)) { _mm_pause(); }
```
把锁挪出热路径:配置加载、日志落盘、对账等**慢路径**才允许用 mutex。

---

### 2. 持锁执行 I/O 或耗时操作【fatal】

**why** —— 在临界区内做网络发送(`send`/`zmq_send`)、Redis 调用(hiredis)、日志 `fwrite`、内存分配(`new`/`malloc` 可能触发 brk/mmap)、字符串格式化等,会把锁持有时间从纳秒级拉到微秒甚至毫秒级,所有等锁线程被这条慢操作"拖死",直接放大全链路尾延迟,且不可预测。

**codeSmell**
```cpp
// 反例:临界区内做网络发送 + 日志 + 分配
{
    std::lock_guard<std::mutex> lk(mtx_);
    auto* msg = new OrderMsg(...);            // 锁内分配
    zmq_send(sock_, msg, sizeof(*msg), 0);    // 锁内 I/O,可能阻塞数毫秒
    logger_.info("sent order {}", order_id);  // 锁内格式化 + 落盘
}
```

**detectHint**
- AST / 人工审查:`lock_guard` 作用域内出现 `send/recv/zmq_/redisCommand/sprintf` / `std::string` 拼接 / `new`/`malloc` / `std::cout` / `log` 调用。
- `strace -T` 看持锁期间的 syscall 耗时;perf 火焰图里锁持有栈下挂着 I/O 栈。
- 用花括号可视化临界区,行数 >5 行就要警惕。

**fix**
```cpp
// 缩小临界区:锁内只做"交换指针/拷贝快照"这类纳秒级操作
OrderMsg snapshot;
{
    std::lock_guard<std::mutex> lk(mtx_);
    snapshot = pending_;        // 仅拷贝 POD 快照
}
// I/O、分配、格式化全部移到锁外
io_thread_queue_.push(snapshot);   // 甩给专门的 I/O 线程
```
读侧无锁可用 RCU / 双缓冲(double buffering)。

---

### 3. std::queue/std::deque + mutex 做跨线程队列而非无锁 ring【high】

**why** —— `std::queue` 底层 `deque` 每次 push 可能触发堆分配(malloc 加锁、可能缺页),配合 mutex 形成"分配 + 锁"双重热路径杀手。高频场景下分配抖动 + 锁争用让入队/出队延迟方差巨大,P99 不可控。对单生产单消费的行情/订单通道,这是典型的"用错数据结构"。

**codeSmell**
```cpp
// 反例:有锁动态分配队列
std::queue<Tick> q_;
std::mutex q_mtx_;
std::condition_variable q_cv_;
void producer(const Tick& t) {
    { std::lock_guard<std::mutex> lk(q_mtx_); q_.push(t); }  // push 可能 malloc
    q_cv_.notify_one();
}
```

**detectHint**
- `grep -nE 'std::(queue|deque|list)<'` 配合同作用域的 `mutex`/`condition_variable`。
- 搜 `push_back`/`pop_front` + `lock` 组合。
- 看队列是否预分配固定容量(无锁 ring 一般是 `alignas` 的定长数组 + 2 的幂掩码)。若队列类型不是基于固定 capacity 的数组实现,基本就是有锁动态分配队列。

**fix**
```cpp
// SPSC ring:定长 2 的幂数组 + 原子 head/tail + 位掩码
template <class T, size_t N>   // N 必须是 2 的幂
class SpscRing {
    static constexpr size_t kMask = N - 1;
    alignas(64) std::atomic<size_t> head_{0};   // 消费者独占 cache line
    alignas(64) std::atomic<size_t> tail_{0};   // 生产者独占 cache line
    alignas(64) std::array<T, N> buf_;          // 预分配全部内存
public:
    bool push(const T& v) {
        size_t t = tail_.load(std::memory_order_relaxed);
        if (t - head_.load(std::memory_order_acquire) >= N) return false;
        buf_[t & kMask] = v;
        tail_.store(t + 1, std::memory_order_release);
        return true;
    }
};
```
多生产多消费用 MPMC(`moodycamel::ConcurrentQueue`)或 LMAX Disruptor 思路。运行期零分配。

---

### 4. 伪共享:多线程频繁写的变量挤在同一 cache line【high】

**why** —— x86 cache line 64 字节。两个线程分别在不同核上高频写同一 cache line 内的不同变量时,MESI 协议反复触发缓存行 invalidate / ping-pong,每次跨核同步几十到上百纳秒。在 ring buffer 的 head/tail 索引、每线程统计计数器、原子标志等高频写字段上,伪共享可让吞吐暴跌数倍、尾延迟显著抬高。

**codeSmell**
```cpp
// 反例:两个被不同线程写的原子挤在一条 cache line
struct Counters {
    std::atomic<uint64_t> producer_idx;   // 生产者线程写
    std::atomic<uint64_t> consumer_idx;   // 消费者线程写 —— 同一 line,ping-pong
};
```

**detectHint**
- 审查相邻声明的多个原子/计数器是否缺 `alignas(64)` 或显式 padding。
- `grep -nE 'std::atomic'` 看结构体内多个原子是否紧挨着。
- `perf c2c record/report` 是检测伪共享的金标准(看 **HITM** —— remote hit modified 高即坐实)。
- `perf stat` 看 `cache-misses` / `L1-dcache` 异常。

**fix**
```cpp
struct Counters {
    alignas(64) std::atomic<uint64_t> producer_idx;   // 独占一条 line
    alignas(64) std::atomic<uint64_t> consumer_idx;   // 独占一条 line
    // 或 C++17:alignas(std::hardware_destructive_interference_size)
};
```
每线程统计计数器按线程分槽并对齐;只读共享数据与可变数据分离(冷热分离)。

---

### 5. std::atomic 使用过强内存序(默认 seq_cst)【medium】

**why** —— atomic 的 load/store/RMW 默认 `memory_order_seq_cst`,在 x86 上 seq_cst store 会插入 `mfence`(全屏障,几十 ns),且 seq_cst 要求全局单一修改顺序,限制编译器/CPU 重排,牺牲性能。绝大多数生产者-消费者场景只需 acquire/release,纯计数/标志位甚至 relaxed 即可。在每 tick 都触发的原子操作上,seq_cst 的累积开销可观且抬高方差。

**codeSmell**
```cpp
// 反例:隐式运算符重载 = seq_cst,每次 store 一个 mfence
std::atomic<uint64_t> seq_;
seq_ = next;            // 等价 store(next, seq_cst)
uint64_t x = seq_;      // 等价 load(seq_cst)
counter_.fetch_add(1);  // seq_cst RMW
```

**detectHint**
- `grep -nE '\.(load|store|fetch_add|fetch_sub|exchange|compare_exchange)\('` 检查是否显式传了 memory_order——没传即默认 seq_cst。
- `grep 'std::atomic'` 看是否用了重载运算符(`x = atomic` / `atomic = x` 也是 seq_cst)。
- 反汇编看热路径是否有多余 `mfence`/`lock` 前缀。

**fix**
```cpp
// 生产者发布用 release,消费者读取用 acquire,配对建立 happens-before
seq_.store(next, std::memory_order_release);
uint64_t x = seq_.load(std::memory_order_acquire);

// 纯统计计数器用 relaxed
counter_.fetch_add(1, std::memory_order_relaxed);
```
一律**显式**写 memory_order,让意图清晰;避免隐式运算符重载。只有真正需要全局顺序的极少数场景才保留 seq_cst。

---

### 6. 热路径上用 condition_variable 阻塞唤醒【high】

**why** —— `condition_variable` 的 wait/notify 走 futex,涉及内核态切换:消费者被唤醒需经过调度器重新上 CPU,从 notify 到实际运行有微秒级且不可控的唤醒延迟;还存在伪唤醒(spurious wakeup)必须 `while` 谓词复查。在"每来一个 tick 就 notify 一次"的极低延迟场景,这套阻塞/唤醒开销远高于自旋轮询,且尾延迟受调度影响严重。

**codeSmell**
```cpp
// 反例:每条消息 notify,消费者阻塞等待
void on_tick(const Tick& t) {
    { std::lock_guard<std::mutex> lk(m_); q_.push(t); }
    cv_.notify_one();   // 每 tick 一次 futex 唤醒,μs 级且抖动
}
void consumer() {
    std::unique_lock<std::mutex> lk(m_);
    cv_.wait(lk, [&]{ return !q_.empty(); });
}
```

**detectHint**
- `grep -nE 'std::condition_variable|\.wait\(|\.notify_one\(|\.notify_all\('` 出现在行情/信号/下单线程。
- 看 wait 是否在每条消息粒度被调用。
- `perf sched` 看唤醒延迟(`sched:sched_wakeup` 到 `sched_switch` 间隔)。
- 火焰图出现 `pthread_cond_wait`/`futex` 即命中。

**fix**
```cpp
// 超低延迟通道:消费者忙轮询无锁 ring,彻底去掉 cv
void consumer() {
    for (;;) {
        Tick t;
        if (md_ring_.pop(t)) { process(t); }
        else { _mm_pause(); }     // SMT 友好
    }
}
// 若需降功耗,用 spin-then-block 混合策略
```
必须用 cv 时确保带谓词(`while(!pred) cv.wait`,防伪唤醒);notify 尽量在锁外发。

---

### 7. 跨线程传递 shared_ptr 导致引用计数原子争用【high】

**why** —— `shared_ptr` 的控制块引用计数是原子的,跨核拷贝/析构 `shared_ptr` 会触发原子 RMW(`fetch_add`/`sub`),多个核高频操作同一控制块计数 = 缓存行 ping-pong + 原子争用,这是隐藏的伪共享。在热路径上按值传递 `shared_ptr`(每次拷贝一次 inc、析构一次 dec)累积开销显著,且 dec 到 0 时还可能在热路径触发析构 + 释放内存。

**codeSmell**
```cpp
// 反例:热路径按值传 shared_ptr,每次调用 inc/dec 原子计数
void process(std::shared_ptr<OrderBook> book) {   // 按值 = 拷贝 + 原子++
    book->update(...);
}   // 析构 = 原子--,可能触发 delete
```

**detectHint**
- `grep -nE 'std::shared_ptr'` 看热路径函数签名是否按值传(应按 `const&` 或裸指针/索引)。
- 看是否跨线程共享同一 `shared_ptr` 对象并频繁拷贝。
- `perf c2c` 看控制块所在 cache line 的 HITM;火焰图里出现 `__shared_count` 原子操作。

**fix**
```cpp
// 热路径传 const&、裸指针或对象池句柄
void process(const std::shared_ptr<OrderBook>& book) { book->update(...); }
void process(OrderBook* book) { book->update(...); }   // 生命周期外层保证

// 跨线程交接所有权用一次性 move,不反复拷贝
worker.submit(std::move(order_ptr));

// 对象池 + 32 位索引代替指针
uint32_t handle = pool_.acquire();
```
只读共享大对象用 RCU/双缓冲发布裸指针,生命周期由发布侧统一管理。

---

### 8. 热路径频繁创建销毁 std::thread 而非线程池+绑核【high】

**why** —— 每次 `std::thread` 创建涉及 clone syscall、栈分配、TLS 初始化,几十微秒级且抖动大;析构/join 同样昂贵。更关键的是:不绑核(CPU affinity)的工作线程会被 OS 调度器在核间迁移,迁移即丢失 L1/L2 缓存与 TLB,造成突发的缓存未命中尖刺,直接抬高尾延迟。交易引擎的关键线程必须长期常驻并绑定独占核。

**codeSmell**
```cpp
// 反例:每个请求起一个线程,且不绑核
void on_order(const Order& o) {
    std::thread([o]{ submit(o); }).detach();   // 每单一次 clone,核间迁移
}
```

**detectHint**
- `grep -nE 'std::thread|pthread_create|\.detach\(|\.join\('` 看是否在请求/消息处理粒度创建线程。
- 反查是否存在 `pthread_setaffinity_np`/`sched_setaffinity`/`CPU_SET` 绑核调用——关键线程完全没有绑核代码则可疑。
- `perf stat` 看 `context-switches`、`cpu-migrations` 偏高;`cat /proc/<pid>/task` 看线程数是否随负载剧烈波动。

**fix**
```cpp
// 启动期创建常驻工作线程,运行期不再创建销毁,并绑定独占核
std::thread worker([]{ strategy_loop(); });
cpu_set_t set; CPU_ZERO(&set); CPU_SET(/*isolated core*/ 3, &set);
pthread_setaffinity_np(worker.native_handle(), sizeof(set), &set);

// 实时优先级减少被抢占
sched_param sp{}; sp.sched_priority = 80;
pthread_setschedparam(worker.native_handle(), SCHED_FIFO, &sp);
```
配合内核 `isolcpus`/`nohz_full`/`rcu_nocbs`;NUMA 场景把线程、内存、网卡中断绑到同一 NUMA 节点。

---

### 9. 自旋忙等缺退避/让出策略不当【medium】

**why** —— 自旋等待若是裸 `while` 空转,在 SMT(超线程)同核兄弟逻辑核上会饿死对方、浪费流水线资源,并让 CPU 无法降功耗;但若过早 `yield`/`sched_yield` 让出又会引入调度延迟,破坏低延迟。两个极端都伤尾延迟:纯空转浪费能效 + 饿邻居,过度让出引入 μs 级唤醒抖动。关键是自旋循环里缺少 CPU `pause` 提示。

**codeSmell**
```cpp
// 反例 A:裸空转,饿死 SMT 兄弟核
while (!ready_.load(std::memory_order_acquire)) { }   // 无 pause

// 反例 B:每圈都 yield,引入调度抖动
while (!ready_.load()) { std::this_thread::yield(); }
```

**detectHint**
- `grep -nE 'while\s*\(.*\)\s*\{?\s*\}?'` 找空自旋循环。
- 检查自旋体内是否有 `_mm_pause()`/`__builtin_ia32_pause()`/`asm volatile("pause")`/`std::this_thread::yield`。
- 自旋热路径里出现 `sched_yield()`/`usleep()`/`sleep_for` 说明让出可能过激。
- `perf stat` 看 IPC 异常低 + 高 CPU 占用但低吞吐 = 无效自旋。

**fix**
```cpp
// 分级退避:短自旋(带 pause)→ 更多自旋 → 必要时 yield/阻塞
int spins = 0;
while (!ready_.load(std::memory_order_acquire)) {
    if (spins < 64)       { _mm_pause(); }                 // SMT 友好,缩短退出延迟
    else if (spins < 256) { for (int i=0;i<16;++i) _mm_pause(); }
    else                  { std::this_thread::yield(); }   // 长时空闲才让出
    ++spins;
}
```
延迟敏感的关键线程独占核时可纯 busy-poll 不让出;权衡功耗与延迟选择策略,而非一刀切。

---

### 10. 读多写少共享数据用互斥锁而非 RCU/双缓冲/seqlock【medium】

**why** —— 配置参数、风控阈值、合约表、订单簿快照这类"读极频繁、写极少"的数据,若用 `std::mutex`(甚至 `shared_mutex`)保护,读侧每次都要付出加锁的原子操作与潜在争用成本。在每 tick 都要读配置/阈值的热路径上,纯读锁开销与写者偶发独占会无谓抬高读延迟方差。

**codeSmell**
```cpp
// 反例:每 tick 加读锁读一次几乎不变的风控阈值
std::shared_mutex cfg_mtx_;
double max_pos_;
double get_max_pos() {
    std::shared_lock<std::shared_mutex> lk(cfg_mtx_);   // 读侧加锁
    return max_pos_;
}
```

**detectHint**
- 审查被 `mutex`/`shared_mutex` 保护的数据的读写比——读远多于写却用了互斥锁即可疑。
- `grep 'shared_mutex'/'shared_lock'` 在热路径读场景。
- 看是否有更新极少的全局配置/参数表被加锁读取。

**fix**
```cpp
// 双缓冲:写者准备新副本后原子换指针,读侧完全无锁
std::atomic<const Config*> cfg_{&buf_a_};
const Config* cfg() { return cfg_.load(std::memory_order_acquire); }  // 读侧零锁
void publish(Config&& next) {            // 写者(极少)
    buf_b_ = std::move(next);
    cfg_.store(&buf_b_, std::memory_order_release);
    std::swap_ranges_later(...);          // 双缓冲轮换
}
```
小结构体可用 seqlock(读者乐观读 + 版本号校验);更新极少的只读数据直接做不可变发布(immutable snapshot),换指针而非改原对象。

---

### 11. 锁粒度过粗:一把大锁保护无关数据【medium】

**why** —— 用单一全局 mutex 保护多个互不相关的资源(如订单表 + 持仓 + 统计同用一把锁),会人为制造争用:本可并行的操作被串行化,等锁线程增多放大尾延迟。粗粒度锁还容易诱发持锁做多件事、临界区变长的连锁问题。

**codeSmell**
```cpp
// 反例:一把 g_mutex 保护所有不相关数据
std::mutex g_mutex;
void update_orders()    { std::lock_guard lk(g_mutex); orders_.update(...); }
void update_positions() { std::lock_guard lk(g_mutex); positions_.update(...); } // 被串行化
void bump_stats()       { std::lock_guard lk(g_mutex); ++stat_; }
```

**detectHint**
- `grep` 同一个 mutex 变量被多少个不同函数/数据访问点引用——引用面广、保护对象语义不相关即粒度过粗。
- 看是否存在一个 `g_mutex`/`globalLock` 贯穿整个引擎。
- `perf lock` 看该锁的等待次数/等待时长是否畸高。

**fix**
```cpp
// 按数据/分片各用独立锁,降低争用面
std::mutex orders_mtx_;
std::mutex positions_mtx_;
std::atomic<uint64_t> stat_{0};   // 纯计数直接无锁

// 或按 symbol 分片(sharding)
std::array<std::mutex, kShards> shard_mtx_;
auto& lock_for(uint32_t sym) { return shard_mtx_[sym % kShards]; }
```
能用无锁结构的子模块换掉锁;拆锁后固定加锁顺序防死锁;评估能否用 per-thread 数据 + 最终汇总彻底消除共享。

---

## 二、内存陷阱 (Memory)

### 1. 热路径动态内存分配(new/delete/malloc 在 tick→signal→order 路径上)【fatal】

**why** —— `malloc`/`new` 会走 glibc ptmalloc 的 arena 锁、free-list 查找,甚至触发 `mmap`/`brk` 系统调用与缺页中断。单次延迟从几十 ns 到数百 ns 不等,且高度不可预测——正是它造成 P99.9 尾延迟尖刺。多线程下还有 arena 锁争用。热路径上任何一次堆分配都是低延迟引擎的头号杀手。

**codeSmell**
```cpp
// 反例:每条消息 new 一个对象
void on_message(const Msg& m) {
    auto* order = new Order(m);            // 堆分配,可能触发 brk/mmap
    std::vector<Leg> legs;                 // 未 reserve,push 时再分配
    legs.push_back(...);
    send(order);
    delete order;                          // 析构 + free
}
```

**detectHint**
- `grep -nE '\b(new|delete|malloc|calloc|realloc|free|make_shared|make_unique)\b'` 限定在 `on_tick/on_message/handle_quote/process_order` 等回调体内。
- clang-tidy 启用 `cppcoreguidelines-no-malloc`。
- `perf record -e syscalls:sys_enter_mmap` 或对 `malloc` 打 uprobe,看热路径线程是否有分配调用栈。
- `heaptrack` / `valgrind --tool=massif` 定位热点分配点。测试环境可用自定义 `operator new` 在开盘后 abort 强制暴露。

**fix**
```cpp
// 启动期预分配对象池,热路径只 acquire/release
ObjectPool<Order, 1u << 14> order_pool_;   // 开盘前全部构造好
void on_message(const Msg& m) {
    Order* o = order_pool_.acquire();       // 纳秒级,无 syscall
    o->reset(m);
    send(o);
    order_pool_.release(o);                  // 归还,不析构不 free
}
```
原则:**所有分配在开盘前完成,运行期 RSS 不增长**。容器初始化期一次性 `reserve` 到最大容量。

---

### 2. std::string / std::vector 在热路径反复扩容拷贝【high】

**why** —— vector/string 容量不足时按倍增策略 `realloc` + 拷贝全部元素,这是隐藏的堆分配 + memcpy。在热路径里拼接订单字符串、累积成交、`push_back` 到未 reserve 的 vector,每次跨越容量阈值就是一次分配尖刺;string 的 SSO 一旦超过 ~15/22 字节也立刻落堆。看似无 `new`,实则每条消息都可能触发分配。

**codeSmell**
```cpp
// 反例:未 reserve 的 vector + string 拼接
void on_fill(const Fill& f) {
    std::vector<Fill> fills;                       // 容量 0,push 即分配
    fills.push_back(f);
    std::string id = "ORD-" + std::to_string(f.id) // 多次堆分配 + SSO 溢出
                   + "-" + std::to_string(f.seq);
}
```

**detectHint**
- `grep -nE '\.(push_back|emplace_back|append|insert|operator\+=|\+)'` 在热路径函数中,反查同一容器外层是否有 `.reserve()`。
- 出现在循环/回调里的 `std::string` 拼接(`+`/`+=`)。
- clang-tidy `performance-inefficient-string-concatenation`。
- `heaptrack` 看分配次数随消息数线性增长即为信号;`ltrace -e 'malloc'` 计数。

**fix**
```cpp
// 预分配 + 复用 buffer + 手写/快速格式化
std::vector<Fill> fills_;                  // 成员变量
fills_.reserve(kMaxFills);                  // 初始化期一次

void on_fill(const Fill& f) {
    fills_.clear();                          // clear 不释放容量
    fills_.push_back(f);
    char buf[32];
    char* p = fmt::format_to(buf, "ORD-{}-{}", f.id, f.seq);  // 写预分配 buffer
}
```
只读传递用 `std::string_view`;定长场景用 `std::array` + 手写格式化(避免 `std::to_string`)。

---

### 3. 热路径虚函数 / 多态间接跳转(vtable 指针追逐)【high】

**why** —— 虚调用需要先加载对象的 vptr(可能 cache miss),再加载 vtable entry,再间接跳转——这条间接跳转破坏 CPU 分支预测与指令预取,且阻止编译器内联。在每条 tick 都调用的策略/handler 接口上,虚函数会显著抬高均值并加剧尾延迟(预测失败时流水线清空数十周期)。

**codeSmell**
```cpp
// 反例:每 tick 通过基类指针虚调用
struct IStrategy { virtual void onTick(const Tick&) = 0; };
IStrategy* strat_;
void dispatch(const Tick& t) { strat_->onTick(t); }   // 间接跳转,无法内联
```

**detectHint**
- `grep -nE '\bvirtual\b'` 后看这些类是否出现在热路径回调里;查找通过基类指针/引用调用方法的模式(`IStrategy*->onTick()`)。
- `perf stat -e branch-misses,branches` 看分支预测失败率;`perf annotate` 看热点函数是否有 `callq *%reg`(间接调用)。
- `objdump` 反汇编确认。

**fix**
```cpp
// CRTP 静态多态:编译期解析,可内联
template <class Derived>
struct StrategyBase {
    void onTick(const Tick& t) { static_cast<Derived*>(this)->onTickImpl(t); }
};
struct MyStrategy : StrategyBase<MyStrategy> {
    void onTickImpl(const Tick& t) { /* ... */ }   // 直接内联,无 vtable
};

// 或类型集合固定时用 std::variant + std::visit
std::variant<StratA, StratB> strat_;
std::visit([&](auto& s){ s.onTick(t); }, strat_);
```
仅在配置/初始化等冷路径保留虚函数。

---

### 4. std::map / std::unordered_map 在热路径做缓存不友好查找【high】

**why** —— `std::map` 是红黑树,每次查找 O(log n) 次指针追逐,节点散落堆上,几乎每跳一次 node 就一次 cache miss。`std::unordered_map` 是链式哈希,bucket 数组 + 节点链表同样指针追逐且节点独立堆分配。在按 symbol/order_id 高频查找的热路径上(订单簿、持仓表、行情索引),这种指针追逐式数据结构是 cache miss 重灾区,延迟随表增大而恶化。

**codeSmell**
```cpp
// 反例:每 tick 在红黑树/链式哈希里查
std::map<uint64_t, Order> orders_;
std::unordered_map<std::string, Position> positions_;
Order& find(uint64_t id) { return orders_.at(id); }   // O(log n) 指针追逐
```

**detectHint**
- `grep -nE 'std::(map|unordered_map|set|unordered_multimap)'` 看是否用于热路径查找;查热路径中的 `.find()`/`.at()`/`operator[]`。
- `perf stat -e LLC-load-misses,L1-dcache-load-misses,cache-misses` 对比热点函数。
- `valgrind --tool=cachegrind` 量化 miss 率。

**fix**
```cpp
// flat hash:开放寻址 + 连续存储,cache 友好
absl::flat_hash_map<uint32_t, Order> orders_;        // 或 robin_hood / ankerl::unordered_dense

// key 是小整数:直接数组 O(1) 索引
std::array<Order, kMaxOrders> order_by_id_;          // symbol/id 映射成连续 int
Order& find(uint32_t id) { return order_by_id_[id]; }

// 订单簿用价格档位数组,而非树
std::array<PriceLevel, kLevels> bids_, asks_;
```
需要顺序 + 查找用 `boost::container::flat_map`(排序 vector)。

---

### 5. 按值传递大对象 / 不必要的拷贝【high】

**why** —— 按值传 `std::string`、`std::vector`、大 struct(如完整 `MarketDataSnapshot`/`OrderBook`)会触发深拷贝 + 堆分配。返回容器时若未触发 RVO/move 也会拷贝。热路径上每条消息一次大对象拷贝,既是分配又是 memcpy,直接抬高延迟。函数边界、回调、lambda 捕获是高发区。

**codeSmell**
```cpp
// 反例:按值传大对象 + range-for 拷贝元素
void evaluate(OrderBook book) {                  // 整本订单簿深拷贝
    for (auto level : book.levels()) { ... }     // auto 拷贝每个 level
}
```

**detectHint**
- `grep` 函数签名中按值传容器/大类型的参数(非 `const&` 非 `&&`)。
- clang-tidy `performance-unnecessary-value-param`、`performance-unnecessary-copy-initialization`、`performance-for-range-copy`。
- `grep 'for *( *auto +\w+ *:'` 漏掉 `&` 的循环。
- perf 看拷贝构造/memcpy 在热点。

**fix**
```cpp
void evaluate(const OrderBook& book) {           // 只读 const&
    for (const auto& level : book.levels()) { ... }  // const auto&,零拷贝
}
order_queue_.push(std::move(order));             // 转移所有权用 move
// 小 POD (<=16B) 按值反而更好,别一刀切
```
只读入参用 `const T&` / `std::string_view` / `std::span`;确保返回值依赖 RVO/NRVO。

---

### 6. AoS 布局导致热遍历 cache miss / 冷热数据混排【high】

**why** —— 把所有字段塞进一个大 struct 再放进数组(Array of Structs),当热路径只遍历访问其中少数字段(如只读 price/qty)时,每次加载整个 struct 撑满 cache line,有效数据占比低,cache 利用率差,带宽浪费,尾延迟上升。同理把高频访问的热字段和很少碰的冷字段(字符串名、时间戳、日志元数据)放同一对象,冷字段挤占 cache line。

**codeSmell**
```cpp
// 反例:大 struct 的数组,热循环只用 price 却拉满整行
struct Quote {                          // sizeof ~ 200B
    double price, qty;                  // 热字段
    char symbol[32]; uint64_t ts;       // 冷字段,挤占 cache line
    char venue[16]; LogMeta meta;
};
std::vector<Quote> quotes_;
double vwap() { double s=0; for (auto& q : quotes_) s += q.price * q.qty; return s; } // 每条拉 200B
```

**detectHint**
- 审查热路径遍历的数据结构:大 struct(字段多)+ 数组,热循环只访问 1-2 字段。
- `perf stat cache-misses` 高 + `perf mem` 看访问模式。
- `pahole` 看 struct 布局、cache line 跨越和 padding 空洞;`cachegrind` 量化 D1 miss。

**fix**
```cpp
// SoA:每个字段一条连续数组,遍历只触碰需要的
struct QuoteBook {
    std::vector<double> price, qty;     // 紧凑连续,利于 SIMD/向量化
    std::vector<uint64_t> ts;           // 冷字段单独存
};
double vwap(const QuoteBook& b) {
    double s = 0;
    for (size_t i = 0; i < b.price.size(); ++i) s += b.price[i] * b.qty[i];
    return s;
}
```
冷热分离:热字段集中紧凑结构,冷字段拆旁路结构用 id 关联。注意:随机访问单条记录时 AoS 更好,按访问模式选。

---

### 7. 热点共享数据 false sharing(多核同 cache line 写竞争)【high】

**why** —— 两个被不同线程频繁写的变量(如 SPSC ring buffer 的 head/tail 索引、各线程的计数器)落在同一 64B cache line 时,一个核写就让另一个核的副本失效(MESI 协议 cache line ping-pong),引发跨核同步的隐性延迟。这是无锁队列性能塌方的经典原因,均值看不出、尾延迟剧烈抖动。

**codeSmell**
```cpp
// 反例:producer/consumer 索引同 line
struct Ring {
    std::atomic<size_t> write_pos;   // 生产者线程写
    std::atomic<size_t> read_pos;    // 消费者线程写 —— 同 line,ping-pong
    std::array<T, N> buf;
};
```

**detectHint**
- 审查无锁队列/多线程共享结构里相邻的原子变量、计数器是否被不同线程写。
- `perf c2c record/report` 专门定位 false sharing 的 cache line 和冲突地址。
- `pahole` 看两个热字段是否同 line;`grep alignas` 缺失;查 `std::atomic` 成员紧挨排列。

**fix**
```cpp
struct Ring {
    alignas(64) std::atomic<size_t> write_pos;   // 独占 cache line
    alignas(64) std::atomic<size_t> read_pos;    // 独占 cache line
    alignas(64) std::array<T, N> buf;
    // C++17: alignas(std::hardware_destructive_interference_size)
};
```
只读共享的热常量可放一起(constructive interference)。验证用 `perf c2c` 复测。

---

### 8. 未对齐 / 跨 cache line 的热数据结构【medium】

**why** —— 热点 struct 起始地址未按 cache line(64B)对齐,或大小跨越 line 边界,导致一次逻辑访问横跨两条 cache line,触发两次 cache line 加载;原子操作跨 line 在部分平台甚至非原子或极慢。环形缓冲槽位未对齐也会让相邻槽位共享 line 加剧 false sharing。

**codeSmell**
```cpp
// 反例:热 struct 无对齐,ring slot 大小非 line 整数倍
struct HotState { uint64_t a, b, c; };   // 24B,跨核访问可能横跨两 line
struct Slot { char data[40]; };           // 40B 非 64 整数倍,相邻 slot 共享 line
Slot ring_[N];
```

**detectHint**
- `pahole` 查看 struct 大小、对齐和是否跨 line。
- `grep` 热点 struct/数组定义缺少 `alignas`;检查 ring buffer slot 大小是否为 cache line 整数倍。
- 查 `reinterpret_cast` / placement new 到未对齐 buffer;UBSan(`-fsanitize=alignment`)捕获未对齐访问。

**fix**
```cpp
struct alignas(64) HotState { uint64_t a, b, c; };      // 整体对齐
struct alignas(64) Slot { char data[64]; };             // padding 到 line 整数倍

// 对齐内存分配
void* p = std::aligned_alloc(64, N * sizeof(Slot));     // 或 posix_memalign
```
大热表按 page(4K / 2M huge page)对齐减少 TLB miss。SIMD load/store 用对齐版本前确保数据已对齐。

---

### 9. 未 reserve 导致容器运行期增量分配【medium】

**why** —— vector/unordered_map 等容量增长容器在热路径不断 push/insert 而事前未 reserve,首次跨越容量阈值即触发 `realloc` + 全量 rehash/拷贝。哪怕只在前几百条消息发生,这些早期分配尖刺也会污染冷启动期的尾延迟,且若容器随交易日增长(如订单表)会持续撞分配。

**codeSmell**
```cpp
// 反例:声明即用,无 reserve
std::vector<Order> active_;                 // 容量 0
std::unordered_map<uint64_t, Order> book_;  // 默认 bucket 数
void on_new(const Order& o) {
    active_.push_back(o);                    // 跨阈值即 realloc 全量拷贝
    book_[o.id] = o;                         // 跨 load_factor 即 rehash
}
```

**detectHint**
- `grep .reserve` / `.rehash` 是否在容器声明附近出现;反查热路径插入的容器是否有对应 reserve。
- `heaptrack` 看分配次数曲线是否随消息累积出现台阶式跳变。
- 静态审查所有热路径容器的初始化代码。

**fix**
```cpp
// 初始化期一次性 reserve 到上限
active_.reserve(kMaxActiveOrders);
book_.max_load_factor(0.5f);
book_.reserve(kMaxOrders);                   // 预留 bucket,避免 rehash
```
大小有硬上限时直接用定长 `std::array` / 自研 ring buffer 彻底消除增长;持续增长的表(历史订单)用分代/归档把热表大小钉死。

---

### 10. 异常出现在热路径或热路径函数未标 noexcept【medium】

**why** —— C++ 零成本异常模型(Itanium ABI)在不抛异常时几乎无运行期开销,但:① 真正 throw 时栈展开极慢(微秒级,彻底毁掉尾延迟);② 热路径用异常做正常控制流(如解析失败、订单拒绝走 throw)是灾难;③ 函数缺 `noexcept` 会阻止某些优化(如 vector 增长时 move vs copy 的选择),并保留异常表/landing pad 影响代码局部性。

**codeSmell**
```cpp
// 反例:用异常做常规分支 + move 构造没标 noexcept
void parse(const Msg& m) {
    if (m.bad()) throw ParseError{};         // 常规失败走 throw = 栈展开
}
struct Order { Order(Order&&); };            // move ctor 未 noexcept → vector 增长走 copy
```

**detectHint**
- `grep -nE '\bthrow\b'` 在热路径函数;查热路径用 try/catch 做常规分支。
- `grep` 热路径函数签名缺 `noexcept`(尤其 move 构造/赋值)。
- `-fno-exceptions` 编译看热路径模块能否通过;`perf` 抓 `__cxa_throw` / `_Unwind_Resume` 调用栈。

**fix**
```cpp
// 可恢复错误用返回码 / std::expected / std::optional,不用异常
std::optional<Order> parse(const Msg& m) noexcept {
    if (m.bad()) return std::nullopt;
    return Order{m};
}
struct Order { Order(Order&&) noexcept; };   // noexcept 保证 vector 增长走 move
```
异常仅用于真正异常的不可恢复路径(启动配置错误等);可考虑对热路径模块 `-fno-exceptions`。

---

### 11. 长生命周期对象池/缓冲导致内存碎片(运行期分配抖动)【medium】

**why** —— 若没有统一的预分配策略,混用不同大小的频繁分配/释放会让 ptmalloc 产生外部碎片,后续分配命中慢路径甚至触发 mmap,RSS 缓慢膨胀,长时间运行(整个交易日)后尾延迟逐渐恶化。碎片是隐蔽的、累积性的尾延迟来源,短测发现不了。

**codeSmell**
```cpp
// 反例:各处自由 new/delete 变长对象,无统一管理
auto* a = new char[37];    // 各种奇怪大小
auto* b = new char[1024];
delete a; delete b;        // 长跑后 ptmalloc 外部碎片累积,RSS 膨胀
```

**detectHint**
- 长跑压测观察 RSS / VmRSS 是否随时间单调增长(`cat /proc/<pid>/status` 的 VmRSS、`malloc_stats()`)。
- `heaptrack`/`massif` 看 allocation 与碎片。
- 对比 glibc malloc vs jemalloc/tcmalloc 的碎片表现;`grep` 是否缺统一内存管理。

**fix**
```cpp
// 统一固定大小块的 slab/对象池,消除变长碎片
SlabAllocator<64>  small_pool_;    // 固定块大小
SlabAllocator<256> mid_pool_;
// 预分配一个大 arena,运行期不向系统申请
```
换 jemalloc / tcmalloc / mimalloc(碎片控制与多线程扩展性优于 glibc ptmalloc);超大连续缓冲用 huge page;监控 RSS 设告警,长跑验证无增长。

---

### 12. 热路径指针追逐式数据结构(链表 / 多级指针 / 树)【high】

**why** —— `std::list`、侵入式链表、嵌套指针对象图、树结构在遍历时每跳一个节点就是一次依赖性内存加载,CPU 无法预取(地址直到上一次加载完成才知道),pointer chasing 把延迟变成串行 cache miss 链。热路径上遍历/查找这类结构,每个 miss ~100ns,N 个节点就是 N 次串行 miss,尾延迟随规模线性恶化。

**codeSmell**
```cpp
// 反例:链表遍历,每个 next 一次串行 cache miss
std::list<Order> active_;
for (auto it = active_.begin(); it != active_.end(); ++it) { eval(*it); }

// 侵入式链表对象图
for (Node* n = head_; n; n = n->next) { ... }   // 节点散落堆上
```

**detectHint**
- `grep -nE 'std::list|std::forward_list'` 在热路径;审查遍历的是否为通过 next/parent/child 指针串联的对象图。
- `perf stat cache-misses` + 高 `stalled-cycles-backend`(内存等待);`perf mem` 看 load latency 长尾。

**fix**
```cpp
// 连续存储优先
std::vector<Order> active_;   // 物理连续,可预取
for (const auto& o : active_) { eval(o); }

// 无法避免追逐时软件预取下一节点
for (Node* n = head_; n; n = n->next) {
    __builtin_prefetch(n->next);   // 提前拉,仅在下一地址可预测时
    process(n);
}
```
侵入式/池化节点用对象池让节点物理连续;树改成隐式数组表示(订单簿用价格档数组、堆用数组)。

---

### 13. 热路径 RAII 析构隐藏的释放 / 容器作用域内重建【medium】

**why** —— 在热路径函数内创建局部容器/string/智能指针,作用域结束时析构触发 `delete`/`free`——分配在构造时、释放在析构时,两端都是堆操作,且这种成本被 RAII 语法藏起来不易察觉。每条消息进出一次热路径函数,就是一对 alloc/free,典型如"每次回调里 new 一个临时 vector 装结果"。

**codeSmell**
```cpp
// 反例:每次回调重建临时容器
std::vector<Signal> on_tick(const Tick& t) {
    std::vector<Signal> result;        // 构造时分配
    result.push_back(...);
    return result;                      // 析构链 + 可能拷贝
}   // ~vector → free
```

**detectHint**
- 审查热路径函数内声明的局部 `std::vector`/`std::string`/`std::unordered_map`/`unique_ptr` 是否每次调用重建。
- `heaptrack` 看分配点是否就是热路径函数;`perf` 看 `~vector`/`~string` 析构里的 free 在热点。

**fix**
```cpp
// 复用成员/线程局部 scratch buffer,clear() 保留 capacity
thread_local std::vector<Signal> scratch_;
void on_tick(const Tick& t, std::vector<Signal>& out) {   // 写调用方预分配缓冲
    out.clear();                        // 不释放容量
    out.push_back(...);
}
```
小定长结果用栈上 `std::array`;核心是把构造/析构成本移出热路径,运行期只复用。

---

## 三、I/O 陷阱 (I/O · 网络 · 序列化)

### 1. 热路径同步日志落盘(synchronous logging on hot path)【fatal】

**why** —— 在 tick→signal→order 热路径里直接调用日志库做格式化 + `write()`/`fsync()`,每条日志都触发一次 write syscall(甚至磁盘 I/O),单次几微秒到几十微秒不等,磁盘抖动时直接打到毫秒级。这是 P99/P99.9 尾延迟最常见的元凶:99% 时间日志很快,偶尔一次落盘卡顿就把尾延迟拉爆。同步日志还会在交易线程内做字符串格式化(见下条),双重伤害。

**codeSmell**
```cpp
// 反例:热路径同步日志 + endl 强刷
void on_tick(const Tick& t) {
    std::ofstream log("trade.log", std::ios::app);
    log << "tick " << t.price << std::endl;     // 每条 write + flush
    spdlog::info("recv tick {}", t.price);       // 默认同步 sink 落盘
}
```

**detectHint**
- `grep` 热路径函数内:`LOG(` / `spdlog::` / `logger->info` / `std::ofstream` / `fprintf(` / `fputs(` / `syslog(` / `<< std::endl`。重点看是否用了同步 logger。
- `perf top` / `strace -f -e write,fsync` 压测时看交易线程是否有 write/fsync;火焰图里热路径下挂 write/format 即命中。
- AST/grep 定位 `tick handler` / `on_message` / `on_tick` 函数体内的日志语句。

**fix**
```cpp
// 无锁异步日志:热路径只把"原始二进制参数"压入 SPSC ring,后台线程格式化落盘
async_log_ring_.push(LogRecord{t.ts, t.price});  // 延迟格式化(deferred formatting)

// 或用 spdlog async / Quill / NanoLog
auto logger = spdlog::create_async<sink_t>("trade", ...);  // 专用 sink 线程
```
绝不在交易线程 `fsync`;日志线程绑定到非关键 CPU 核;分级:致命错误才允许同步,常规埋点全异步。

---

### 2. 热路径字符串格式化/拼接(string formatting on hot path)【high】

**why** —— `std::to_string`、`sprintf`/`snprintf`、`fmt::format`、`std::stringstream`、`operator+` 拼接 `std::string`——这些都会触发堆分配(malloc,可能 brk/mmap)、locale 查询、整数/浮点格式化,单次可达数百纳秒到几微秒,且分配抖动直接推高尾延迟。常见于日志消息、订单 ID 生成、symbol 拼接、错误信息构造。即便日志最终被异步丢弃,格式化本身已在热路径发生。

**codeSmell**
```cpp
// 反例:热路径生成订单字符串
void send_order(const Order& o) {
    std::string oid = "ORD" + std::to_string(o.id);      // 堆分配 + 格式化
    std::stringstream ss;
    ss << o.symbol << "|" << o.price << "|" << o.qty;     // stringstream 极慢
    transmit(ss.str());
}
```

**detectHint**
- `grep`:`std::to_string` / `sprintf` / `snprintf` / `std::stringstream` / `std::ostringstream` / `fmt::format` / `std::string +` / `.append(` / `+= std::string` 在 `on_tick/on_message/send_order` 等。
- 火焰图:热路径下挂 `__vfprintf_internal`、`basic_string` 构造、`operator new`。
- clang-tidy `performance-inefficient-string-concatenation`、`performance-unnecessary-copy-initialization`。

**fix**
```cpp
// 整型 ID / 定长 char 数组,symbol 预映射成 uint32 instrument_id
struct Order { uint64_t id; uint32_t instrument_id; int64_t price; uint32_t qty; };

// 必须发字符串的协议(FIX):预分配缓冲 + 手写 itoa/ftoa,零分配
char buf[64];
char* p = jeaiii::to_text(buf, o.id);     // 或 ryu/dtoa 处理浮点
```
日志走延迟格式化(参数二进制压环,后台格式化);所有缓冲预分配,热路径不 `new`。

---

### 3. JSON 文本序列化/反序列化在热路径(text JSON parse on hot path)【high】

**why** —— 交易所行情/订单走 JSON 文本(WebSocket 推送常见),用 `nlohmann::json`、RapidJSON DOM、jsoncpp 在热路径解析/生成,单条消息几微秒到几十微秒:DOM 模型大量小对象堆分配、字符串拷贝、浮点 parse(`strtod` 慢且不精确)。`nlohmann::json` 尤其慢(每个字段一次 map 查找 + 堆分配)。这是软件层吃掉 tick-to-trade 预算的大头。

**codeSmell**
```cpp
// 反例:nlohmann DOM 解析每条行情
void on_message(const std::string& payload) {
    auto j = nlohmann::json::parse(payload);     // 海量小对象分配
    double price = j["p"].get<double>();         // map 查找 + strtod
    double qty   = j["q"].get<double>();
}
```

**detectHint**
- `grep`:`nlohmann` / `json::parse` / `Json::Value` / `rapidjson::Document` / `.GetObject()` / `jsoncpp` / `std::stod` / `strtod` / `atof` 在 `on_message/parse_tick/decode`。
- 火焰图:热路径下挂 json parser、`strtod`、`operator new` 密集。
- `perf record` 采样符号。

**fix**
```cpp
// 优先二进制协议:SBE / protobuf(arena) / FlatBuffers / Cap'n Proto(zero-copy 直接读字段)
auto tick = flatbuffers::GetRoot<Tick>(buf);   // 无解析,直接视图

// 必须 JSON:simdjson on-demand,只 parse 用到的字段,零拷贝
simdjson::ondemand::parser parser;
auto doc = parser.iterate(payload);
double price = double(doc["p"]);               // fast_float 替代 strtod
```
解析与热路径解耦:解析线程产出 POD 结构体喂给策略线程。

---

### 4. TCP_NODELAY 未设置 / Nagle 算法生效【fatal】

**why** —— 未对交易/行情 socket 设置 `TCP_NODELAY`,Nagle 算法会缓冲小包等待 ACK 或攒满 MSS 才发送,叠加对端 delayed ACK 时可造成最高 ~40ms 的发送停顿。对下单链路这是灾难级尾延迟,且极隐蔽(吞吐正常、只有偶发请求被 hold)。同样适用于内部 ZMQ/服务间 TCP。

**codeSmell**
```cpp
// 反例:connect 后没设 TCP_NODELAY,Nagle 默认生效
int fd = socket(AF_INET, SOCK_STREAM, 0);
connect(fd, ...);
send(fd, order_buf, len, 0);     // 小包被 Nagle 缓冲,最高卡 40ms
```

**detectHint**
- `grep` socket 配置代码是否有 `TCP_NODELAY` / `setsockopt`。`connect()`/`accept()` 后没紧跟 `setsockopt(..., TCP_NODELAY, ...)` 即命中。
- ZMQ:是否设 `ZMQ_TCP_NODELAY`(旧版默认关)。
- `ss -i` / `tcpdump` 看是否有等待 ACK 的小包合并、发送被延迟;`strace` 看 setsockopt 列表。

**fix**
```cpp
// 所有低延迟 socket 创建后立即关 Nagle
int one = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &one, sizeof(one));
setsockopt(fd, IPPROTO_TCP, TCP_QUICKACK, &one, sizeof(one));  // 禁 delayed ACK

// ZMQ 显式开
zmq_setsockopt(sock, ZMQ_TCP_NODELAY, &one, sizeof(one));
```
封装一个 `set_low_latency_sockopts()`,所有 socket 强制走它,避免漏设。

---

### 5. 热路径高频时间戳 syscall(gettimeofday/clock_gettime spam)【high】

**why** —— 每个 tick/每条消息都调 `clock_gettime(CLOCK_REALTIME)`、`gettimeofday`、`std::chrono::system_clock::now()` 做埋点。虽然 vDSO 让多数调用走用户态(几十纳秒),但仍有开销,且若内核 clocksource 退化成 hpet/acpi_pm(非 tsc)则每次变成真 syscall(数百纳秒到微秒),尾延迟剧增。频繁调用累加吃掉延迟预算,还可能引入跨核同步。

**codeSmell**
```cpp
// 反例:每条消息取两次墙钟时间做埋点
void on_message(const Msg& m) {
    auto t0 = std::chrono::system_clock::now();   // CLOCK_REALTIME
    process(m);
    auto t1 = std::chrono::high_resolution_clock::now();
    record(t1 - t0);
}
```

**detectHint**
- `grep`:`clock_gettime` / `gettimeofday` / `system_clock::now` / `high_resolution_clock::now` / `std::time(` / `CLOCK_REALTIME`,统计热路径出现次数。
- 检查 clocksource:`cat /sys/devices/system/clocksource/clocksource0/current_clocksource` 是否为 `tsc`。
- `strace -c` 看 `clock_gettime` syscall 计数异常;`perf` 看 `__vdso_clock_gettime` 占比。

**fix**
```cpp
// 用 RDTSC/RDTSCP 读 TSC,启动时校准频率换算纳秒,单次约 10-20ns
static inline uint64_t rdtsc_now() {
    unsigned aux;
    return __rdtscp(&aux);     // 序列化版本,避免乱序
}
// 换算:ns = (tsc - base) * 1e9 / tsc_freq
```
确保 BIOS/内核启用 invariant TSC(`constant_tsc`、`nonstop_tsc`,`cat /proc/cpuinfo` 确认),`clocksource=tsc`;减少埋点频率(批量/采样);墙钟时间只在慢路径取。

---

### 6. 热路径动态内存分配触发 brk/mmap(I/O 缓冲视角)【fatal】

**why** —— 收发 I/O 时在热路径 `new`/`malloc` 接收缓冲、`std::vector` push_back 扩容、`std::string` 装 payload、`std::function` 捕获回调——这些命中 free list 时几十纳秒,但一旦触发 `brk`/`mmap` 向内核要内存就是 syscall(数微秒)+ 可能缺页中断,且 glibc malloc 有锁(多线程争用)。最坏叠加 page fault、TLB miss,直接打出 P99.9 长尾。分配是 HFT 热路径头号大忌。

**codeSmell**
```cpp
// 反例:每次收包 new 缓冲 + string 装 payload
void on_readable(int fd) {
    auto* buf = new char[4096];                 // 每包分配
    ssize_t n = recv(fd, buf, 4096, 0);
    std::string payload(buf, n);                // 再拷一份 + 可能落堆
    enqueue(std::move(payload));
    delete[] buf;
}
```

**detectHint**
- `grep` 热路径:`new ` / `make_shared` / `malloc` / `std::vector`(无 reserve) / `.push_back` / `std::function` / `std::string` 装 payload。
- `strace -f -e brk,mmap` 压测看交易线程是否有调用;`perf record` 火焰图下挂 `operator new`、`_int_malloc`、`page_fault`。
- `heaptrack` 看热路径分配点。

**fix**
```cpp
// 预分配固定接收缓冲池 / ring slot,recv 直接进预分配内存,零分配
alignas(64) char recv_buf_[kMaxFrame];          // 复用
void on_readable(int fd) {
    ssize_t n = recv(fd, recv_buf_, sizeof(recv_buf_), 0);
    md_ring_.emit(std::string_view(recv_buf_, n));   // 视图,不拷贝
}
```
启动时 `mlockall(MCL_CURRENT|MCL_FUTURE)` 锁页防换出,预触摸所有页避免运行期缺页;避免 `std::function`(用模板/函数指针);换 jemalloc/tcmalloc 减少锁争用(根治还是不分配)。

---

### 7. WebSocket/ZMQ 每条消息多次内存拷贝(no zero-copy)【high】

**why** —— 从内核 recv buffer 到应用、再到解析、再到内部队列,每一跳都 memcpy 一份(recv 到临时 vector → 拷进 `std::string` → 拷进 message 结构 → 拷进队列)。单条消息 4-5 次拷贝,行情高吞吐下吃满内存带宽 + 污染 cache,推高每条处理延迟。ZMQ 用 `zmq_msg` 默认拷贝、WebSocket 库也常多拷。

**codeSmell**
```cpp
// 反例:recv → string → struct → queue,4 次拷贝
std::vector<char> tmp(n);
recv(fd, tmp.data(), n, 0);
std::string s(tmp.begin(), tmp.end());          // 拷贝 1
Message msg; msg.body = s;                        // 拷贝 2
queue_.push(msg);                                 // 拷贝 3
```

**detectHint**
- `grep`:`memcpy` / `std::copy` / `std::string(`(从 buffer 构造) / `.assign(` / `std::vector<char>(` / `msg.data()` 拷出。看 recv→parse→enqueue 链路。
- ZMQ:是否用 `zmq_msg_init_data`(zero-copy)还是 `zmq_msg_init_size`+copy;是否 `ZMQ_RCVMORE` 多帧。
- 火焰图看 `__memcpy_avx_unaligned` 占比;`perf stat` 看 `LLC-load-misses`、内存带宽。

**fix**
```cpp
// 全链路 zero-copy:recv 进预分配 ring slot,解析器在原 buffer 建视图,下游传索引
Slot& slot = ring_.next_slot();
ssize_t n = recv(fd, slot.data, sizeof(slot.data), 0);
auto tick = parse_view(std::span<const char>(slot.data, n));   // 不拷贝
downstream_.push(slot.index);                                   // 传句柄/索引

// ZMQ zero-copy
zmq_msg_t m; zmq_msg_init_data(&m, ptr, len, free_cb, nullptr);
```
多帧用 scatter-gather(`iovec`/`sendmmsg`/`recvmmsg`);内核层考虑 `MSG_ZEROCOPY`、io_uring;内部队列传 POD/句柄而非 `std::string`。

---

### 8. 同步阻塞 I/O / 阻塞 socket 在热路径(blocking I/O on hot path)【fatal】

**why** —— 交易/行情线程用阻塞 `recv`/`send`/`read`/`write`,或在热路径做同步 Redis 命令(hiredis `redisCommand` 同步等回包)、同步 DB 写、阻塞文件 I/O。一次阻塞 syscall 让交易线程进入内核态睡眠、被调度出去,恢复要等唤醒 + 重新上 CPU(cache 全冷),延迟从纳秒级跳到微秒甚至毫秒级,彻底毁掉确定性。

**codeSmell**
```cpp
// 反例:热路径同步访问 Redis + 阻塞 recv
void on_signal(const Signal& s) {
    redisReply* r = (redisReply*)redisCommand(ctx_, "SET pos %f", s.pos); // 同步等回包
    char buf[256];
    recv(fd_, buf, sizeof(buf), 0);   // 阻塞 socket,睡眠等数据
}
```

**detectHint**
- `grep`:`redisCommand(`(同步 hiredis) / `recv(` / `send(` / `read(` / `write(` / `fread` / `fwrite`(热路径) / `std::future::get` / `.wait()` / `mysql_query` / `pqexec`。
- 检查 socket 是否设 `O_NONBLOCK`(`fcntl`)。
- `strace` 看交易线程是否阻塞在 `recv`/`read`/`futex`;`perf sched` 看上下文切换/睡眠。

**fix**
```cpp
// 全异步非阻塞:O_NONBLOCK + epoll(EPOLLET) / io_uring 事件驱动
fcntl(fd, F_SETFL, O_NONBLOCK);
epoll_ctl(ep_, EPOLL_CTL_ADD, fd, &ev);   // 边沿触发

// Redis 移出热路径:异步落库由旁路线程做
redisAsyncCommand(async_ctx_, cb, nullptr, "SET pos %f", s.pos);
```
绝不在交易线程同步等任何 I/O 或锁;跨线程通信用无锁 SPSC ring;慢路径(持久化/审计)交给独立线程。

---

### 9. 缺少批量处理 / 频繁小包 syscall(no batching)【high】

**why** —— 每条消息一次 `recv`/`send` syscall(无 `recvmmsg`/`sendmmsg`),或每个 tick 一次 `epoll_wait` → 处理一条 → 再 `epoll_wait`。高吞吐下 syscall 次数 = 消息数,每次 syscall 用户/内核态切换约 100-300ns,累加占满 CPU 并放大尾延迟。频繁小包还触发更多中断、更差的网卡聚合。

**codeSmell**
```cpp
// 反例:一次 recv 一条,逐条 epoll
for (;;) {
    epoll_wait(ep_, &ev, 1, -1);     // 一次只取一个
    char buf[256];
    recv(ev.data.fd, buf, sizeof(buf), 0);   // 一条一个 syscall
    process_one(buf);
}
```

**detectHint**
- `grep`:单条 `recv(` / `send(` 而非 `recvmmsg` / `sendmmsg` / `readv` / `writev`;epoll 循环里是否一次只处理一个 fd/一条消息。看是否有 batch_size / 攒批逻辑。
- `strace -c` 压测看 `recv`/`send`/`epoll_wait` syscall 计数 vs 消息数(1:1 即未批量)。
- `perf` 看 syscall enter/exit 开销、软中断频率。

**fix**
```cpp
// recvmmsg 一次收多条;epoll_wait 一次返回多个 fd 全处理
struct mmsghdr msgs[kBatch];
int got = recvmmsg(fd, msgs, kBatch, MSG_DONTWAIT, nullptr);  // 一个 syscall 多条
for (int i = 0; i < got; ++i) process(msgs[i]);

int n = epoll_wait(ep_, events_, kMaxEvents, 0);              // 一次取多个
for (int i = 0; i < n; ++i) handle(events_[i]);
```
应用层攒批:ring 攒 N 条或 T 微秒(取小者)再批量处理;发送端 `writev`/`sendmmsg` 合并多帧。

---

### 10. 未用 busy-poll / 内核旁路(no kernel bypass for ultra-low latency)【medium】

**why** —— 对超低延迟(sub-microsecond tick-to-trade)场景仍走标准内核网络栈 + epoll 阻塞等待,包从网卡到应用要经过中断、软中断、协议栈、socket buffer、唤醒调度,固有几微秒 + 抖动。epoll 阻塞唤醒还有调度延迟。这不是 bug 但限制延迟下限,且唤醒抖动推高尾延迟。注意:busy-poll 烧 CPU,是延迟换功耗的取舍,非所有场景适用。

**codeSmell**
```cpp
// 反例:纯内核栈 + 阻塞 epoll 等待,无 busy-poll
epoll_wait(ep_, events_, kMax, -1);   // 阻塞唤醒,μs 级且抖动,内核栈固有延迟
```

**detectHint**
- 看网络栈选型:是否用 DPDK / Solarflare Onload(ef_vi) / Mellanox VMA / io_uring SQPOLL / AF_XDP;还是纯 epoll + 内核栈。
- `grep`:`epoll_wait` 配长 timeout/阻塞、无 busy-poll;是否设 `SO_BUSY_POLL`;是否线程绑核 + 忙等 ring。
- 是否有 `isolcpus`、网卡中断亲和配置;`perf` 看是否大量时间花在 `ksoftirqd`/`schedule`/`futex`。

**fix**
```cpp
// 超低延迟路径用户态轮询网卡 ring,绕开协议栈与中断(DPDK / ef_vi / AF_XDP)
while (running_) {
    int n = rte_eth_rx_burst(port, q, bufs, kBurst);   // DPDK 用户态轮询
    for (int i = 0; i < n; ++i) handle(bufs[i]);
}
// 或 epoll 配 NAPI busy-poll 减少唤醒延迟
int us = 50; setsockopt(fd, SOL_SOCKET, SO_BUSY_POLL, &us, sizeof(us));
```
配套:`isolcpus` 隔离核、CPU 亲和、关 C-states、网卡中断亲和到非交易核、RT 调度;只对真正延迟敏感路径上重武器,别全栈 busy-poll 烧机器。

---

### 11. 热路径分支预测失败 / 缺 likely-unlikely 与分支重排【medium】

**why** —— 热路径里把罕见情况(错误处理、风控拒单、断连重连、异常 symbol)和常见情况平铺成普通 if,CPU 分支预测器对偶发分支命中率低;一次 misprediction 清空流水线约 15-20 cycle(现代核约 5-8ns),高频累加 + 偶发触发推高尾延迟。错误/异常路径内联进热代码还污染 I-cache、撑大热函数。

**codeSmell**
```cpp
// 反例:罕见错误路径平铺,无冷热提示
void on_tick(const Tick& t) {
    if (t.invalid()) { handle_error(t); return; }   // 罕见,却没标 unlikely
    if (risk_reject(t)) { reject(t); return; }       // 罕见
    place_order(t);                                  // 常见路径排在最后
}
```

**detectHint**
- `grep` 热路径 if/switch:错误处理、风控、边界检查是否用了 `[[likely]]`/`[[unlikely]]`(C++20)或 `__builtin_expect`。
- `perf stat -e branch-misses,branches` 看 misprediction 率;`perf record` 热点函数看 branch-misses 集中点。
- PGO 编译报告或 `perf annotate` 看具体分支。

**fix**
```cpp
void on_tick(const Tick& t) {
    if (t.invalid()) [[unlikely]]   { handle_error(t); return; }
    if (risk_reject(t)) [[unlikely]] { reject(t); return; }
    place_order(t);                  // 常见路径主干
}
// 冷路径抽成 noinline,让热函数小、I-cache 友好
[[gnu::noinline]] void handle_error(const Tick&);
```
优先上 **PGO/AutoFDO**(用真实负载 profile 让编译器自动布局冷热代码),通常比手工标注更可靠;真正热的判断可用 branchless(cmov/位运算)。

---

### 12. 关键小函数未内联 / 跨翻译单元调用开销【medium】

**why** —— 热路径上频繁调用的极小函数(ring buffer push/pop、价格转换、订单校验、时间换算)若定义在 `.cpp` 跨翻译单元,无 LTO 时无法内联,每次调用付函数调用开销(栈帧、寄存器保存、间接跳转)外加阻止编译器跨调用优化(无法常量传播/向量化)。虚函数更糟:vtable 间接跳转 + 阻断内联 + 可能 I-cache miss。累加吃掉延迟预算。

**codeSmell**
```cpp
// 反例:热路径小函数定义在 .cpp,无 LTO 不内联
// price.cpp
int64_t to_ticks(double px);     // 跨 TU,每 tick 一次真 call

// hot.cpp
void on_tick(const Tick& t) { int64_t p = to_ticks(t.price); }  // 无法内联
```

**detectHint**
- 看热路径小函数定义位置:`.cpp`(跨 TU、无 inline/LTO 不内联)还是 header inline。
- `grep` 热路径:`virtual`(热路径虚函数)、热函数是否标 `inline`/`[[gnu::always_inline]]`。
- 检查构建是否开 `-flto`;`perf annotate` 看是否有 call 指令到小函数、vtable 间接 call。

**fix**
```cpp
// 关键小函数放 header 并 inline,开启 LTO 让跨 TU 内联
// price.hpp
inline int64_t to_ticks(double px) noexcept { return llround(px * kTickInv); }

// 构建:-flto -O3
```
热路径避免虚函数(CRTP / 模板 / `std::variant`+visit / 函数指针表);配合 PGO 让编译器知道哪些调用热;别盲目 `always_inline` 大函数(撑爆 I-cache 反而更慢)。

---

### 13. std::endl 强制刷缓冲 / iostream 在热路径【medium】

**why** —— `std::cout << x << std::endl` —— `std::endl` 不只是换行,它强制 flush 输出缓冲,每次触发一次 write syscall。在热路径或高频日志里等于每行一次 syscall(数微秒)+ 磁盘抖动风险。叠加 iostream 本身慢(locale、虚函数 sync、内部锁、格式化开销),是经典隐蔽性能坑。`printf` 在热路径同理(格式化 + 行缓冲 flush + 内部锁)。

**codeSmell**
```cpp
// 反例:热路径 iostream + endl
void on_tick(const Tick& t) {
    std::cout << "tick " << t.price << std::endl;   // 每条强制 flush + write
    printf("recv %f\n", t.price);                    // 行缓冲 flush + 内部锁
}
```

**detectHint**
- `grep`:`std::endl`(尤其循环内)、`std::cout` / `std::cerr` / `printf(` / `fflush`(热路径) / `<< std::flush`。
- `strace` 看 write syscall 是否与日志行 1:1。
- 火焰图看 `std::ostream::flush`、`std::__basic_file::xsputn`、`write`。

**fix**
```cpp
// 热路径禁用 iostream 输出;换行用 '\n' 不用 endl;诊断走异步日志
// 若临时调试:
std::ios_base::sync_with_stdio(false);
out << "tick " << t.price << '\n';    // '\n' 不强制 flush,批量后手动 flush
```
生产热路径理想是**零 stdout/stderr 输出**,所有埋点走无锁异步 logger 的二进制延迟格式化(见 I/O 第 1 条)。

---

### 14. 热路径浮点价格运算 / 不精确比较(隐性延迟+正确性)【medium】

**why** —— 用 `double` 表示价格在热路径反复运算,除了浮点精度问题(`0.1 + 0.2 != 0.3` 导致档位/撮合错误),`double` 比较、除法、`fmod` 在某些路径比整数慢,且依赖链长的浮点运算阻碍流水线。更隐蔽的是把价格 `double` 当 map key、或每 tick 做 `double` 转字符串再比较,叠加格式化分配。整数 tick 表示既快又精确。

**codeSmell**
```cpp
// 反例:double 价格直接比较 + 当 key
std::map<double, Level> book_;
if (px == best_bid_) { ... }              // 浮点相等比较不可靠
book_[px] = lvl;                           // double 当 key,精度/cache 双坑
```

**detectHint**
- `grep` 热路径:`double`/`float` 价格的 `==` 直接比较、`std::map<double,` / `unordered_map<double,`、`std::abs(a-b) < eps` 散落各处。
- 查价格是否全程 `double` 而非整数 tick / 定点。
- `perf` 看浮点除法/`fmod` 在热点;正确性侧看撮合/档位偶发错配。

**fix**
```cpp
// 价格用整数 tick(定点),全程整数运算,精确且快
using Ticks = int64_t;
Ticks px_ticks = llround(px / kTickSize);   // 入口转一次
if (px_ticks == best_bid_ticks_) { ... }     // 整数精确比较
std::array<Level, kLevels> book_;            // 档位数组,O(1) 且 cache 友好
```
需要小数运算时用定点(scaled int)或经过校验的 decimal 库;只在展示/落库边界转回 `double`/字符串。

---

## 附录:严重度速查表

| # | 类别 | 陷阱 | 严重度 |
|---|------|------|--------|
| C1 | 并发 | 热路径 std::mutex 锁争用 | fatal |
| C2 | 并发 | 持锁执行 I/O 或耗时操作 | fatal |
| C3 | 并发 | std::queue+mutex 而非无锁 ring | high |
| C4 | 并发 | 伪共享(高频写挤同 cache line) | high |
| C5 | 并发 | atomic 过强内存序(默认 seq_cst) | medium |
| C6 | 并发 | 热路径 condition_variable 阻塞唤醒 | high |
| C7 | 并发 | 跨线程传 shared_ptr 引用计数争用 | high |
| C8 | 并发 | 频繁创建 std::thread 而非池+绑核 | high |
| C9 | 并发 | 自旋忙等退避策略不当 | medium |
| C10 | 并发 | 读多写少用互斥锁而非 RCU/双缓冲 | medium |
| C11 | 并发 | 锁粒度过粗(一把大锁) | medium |
| M1 | 内存 | 热路径动态分配 new/malloc | fatal |
| M2 | 内存 | string/vector 反复扩容拷贝 | high |
| M3 | 内存 | 热路径虚函数 vtable 指针追逐 | high |
| M4 | 内存 | std::map/unordered_map 缓存不友好查找 | high |
| M5 | 内存 | 按值传大对象 / 不必要拷贝 | high |
| M6 | 内存 | AoS 布局 / 冷热混排 cache miss | high |
| M7 | 内存 | 热点共享数据 false sharing | high |
| M8 | 内存 | 未对齐 / 跨 cache line 热结构 | medium |
| M9 | 内存 | 未 reserve 运行期增量分配 | medium |
| M10 | 内存 | 异常在热路径 / 缺 noexcept | medium |
| M11 | 内存 | 长跑内存碎片(分配抖动) | medium |
| M12 | 内存 | 指针追逐式结构(链表/树) | high |
| M13 | 内存 | RAII 析构隐藏释放 / 容器重建 | medium |
| I1 | I/O | 热路径同步日志落盘 | fatal |
| I2 | I/O | 热路径字符串格式化/拼接 | high |
| I3 | I/O | JSON 文本序列化/反序列化 | high |
| I4 | I/O | TCP_NODELAY 未设 / Nagle 生效 | fatal |
| I5 | I/O | 高频时间戳 syscall | high |
| I6 | I/O | I/O 缓冲热路径动态分配 brk/mmap | fatal |
| I7 | I/O | 每条消息多次内存拷贝(no zero-copy) | high |
| I8 | I/O | 同步阻塞 I/O 在热路径 | fatal |
| I9 | I/O | 缺批量处理 / 频繁小包 syscall | high |
| I10 | I/O | 未用 busy-poll / 内核旁路 | medium |
| I11 | I/O | 分支预测失败 / 缺 likely-unlikely | medium |
| I12 | I/O | 关键小函数未内联 / 跨 TU 调用 | medium |
| I13 | I/O | std::endl 强制刷缓冲 / iostream | medium |
| I14 | I/O | 浮点价格运算 / 不精确比较 | medium |

**审查优先级**:先全量扫 `fatal`(锁争用、持锁 I/O、同步日志、Nagle、阻塞 I/O、热路径分配)——这 6 类是 P99.9 尾延迟塌方的直接元凶;再扫 `high`,最后处理 `medium`。

**黄金法则**:tick→signal→order 热路径上,**零分配、零锁、零 syscall、零拷贝、零文本格式化**;所有重活前移到开盘前,运行期只做确定性的纳秒级操作。

