---
title: "每日基础技术总结 · 2026-10-31 · 微服务熔断器Hystrix/Sentinel的滑动窗口统计原理"
date: 2026-10-31 07:01:34
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-31 · 微服务熔断器Hystrix/Sentinel的滑动窗口统计原理

## 📚 今日主题

> **微服务熔断器Hystrix/Sentinel的滑动窗口统计原理**（后端基础）

### 1. 核心概念速览
**定义**：滑动窗口统计是一种数据结构，其核心是「把时间轴切成等长的时间片（桶）+ 用环形数组复用这些桶 + 用世代号惰性重置每个桶」。它的作用是在恒定的内存开销（O(N)）、写入开销 O(1)、读取开销 O(N) 下，维护「最近一个时间区间内」按事件类型（成功 / 失败 / 超时 / 拒绝 / RT）聚合的**近似**计数与均值，误差上界为一个桶的时长。Hystrix 的 HystrixRollingNumber / HystrixRollingPercentile 与 Sentinel 的 LeapArray / WindowWrap / MetricBucket 都是它的具体实现。

**解决什么问题**：熔断器的开断判定依赖「当前健康度」，而不是「历史总量」。
- 全局计数器单调递增、没有时间衰减，故障恢复后失败率仍长期保持高位，无法反映当前状态；
- 固定窗口（fixed window）每 T 清零一次，在窗口切换的边界处存在计数突变：t=T-1 与 t=T+1 的两次突发会被切到两个窗口，各自都不超阈值，但现实中 2ms 内已经产生了 2 倍的错误量，导致判定滞后或抖动。
滑动窗口提供一种随时间衰减的滚动聚合，把边界效应压缩进一个桶内。

**机制本质**：三条不变式。
1) 时间定位：`idx = floor(t / w) mod N`，任意时刻只有唯一一个「活跃世代」的桶可写；
2) 世代校验：桶内保存 `windowStart`（或 `windowNumber`），读写前比对当前世代，不匹配即判定为「上一轮残留」，写前惰性重置、读时直接跳过；
3) 聚合读取：遍历全部 N 个桶，只累加世代号落在 `[curStart-(N-1)w, curStart]` 内的桶。

**体系位置**：位于服务治理 → 稳定性防护这条链路的**数据面最底层**。向上支撑熔断状态机（CLOSED / OPEN / HALF_OPEN）、Sentinel 的 FlowSlot 限流 QPS 统计、系统自适应保护；横向与后端 Metrics（Micrometer / Prometheus 的 counter 快照、histogram）、时序库的 rollup 降采样、前端埋点采样缓冲同源，都是「用有损近似换恒定开销」这一权衡的具体实例。

**为什么必须掌握**：它回答了「高并发热点路径上如何做度量」这一基础问题。不理解桶化滑动的误差结构，就无法正确设置窗口长度 / 桶数量 / 最小请求数，也无法解释线上两类典型现象：失败率已经很高却没有熔断、刚恢复就立刻再次熔断。集群流控、自适应限流、混沌实验的探针判定全部建立在这一层之上。

### 2. 底层原理剖析
**1. 问题形状：判定「最近 T 内的失败率」**
- 全局累计：`ratio = fail / (pass + fail)`，永不衰减，故障恢复后长期高位，无法用于自动闭合。
- 固定窗口：每 T 清零。缺陷有二：边界突变（跨边界的两段突发各自不超阈值）；清零瞬间计数归零，可能让刚 OPEN 的熔断器立刻抖动回 CLOSED。
- 桶化滑动窗口：把 T 切成 N 份（w = T/N），窗口以 w 为步长**跳跃**滑动，任意时刻统计的是「最近 N 个槽位中世代有效的桶」，边界效应被限制在 w 之内。

**2. 数据结构**
    Bucket[N]                         // 环形数组，N = sampleCount
    每个 Bucket:
      gen      : 世代号          // Hystrix 用 windowNumber = floor(t/w)（单调绝对编号）
                                 // Sentinel 用 windowStart = floor(t/w)*w（桶起点毫秒）
      counters : LongAdder[K]    // 按事件类型索引；Sentinel 的 MetricBucket 另有 minRt（volatile long）
写路径完全不加锁（LongAdder 分段 CAS），仅在世代切换时进入临界区。

**3. 写路径（惰性重置 / lazy reset）**
    write(event, t):
      i = floor(t / w) mod N
      s = floor(t / w) * w
      b = arr[i]
      if b.gen != s:                  // 快路径：世代匹配则零同步开销
          lock(i)                     // Hystrix: 对 windowNumber 做 CAS；Sentinel: ReentrantLock + 二次校验
              if b.gen != s:          // 必须二次校验，否则会清掉并发线程刚写入的计数
                  b.gen = s; b.counters.reset()
      b.counters[event].increment()
不存在任何后台清理线程。桶在被「下一轮的同一槽位」选中之前，一直是历史数据；一旦被选中，靠世代号识别并原地复用。空闲（无请求）期间系统零开销，这是把清理方式从「推送模型（定时扫描）」改成「拉取模型（访问时校验）」的直接收益。

**4. 读路径**
    count(event, t):
      cur = floor(t / w) * w
      low = cur - (N - 1) * w          // 等价于 cur - T + w
      sum = 0
      for b in arr:
          if low <= b.gen <= cur:      // 只累加世代有效的桶
              sum += b.counters[event].sum()
      return sum
这个 `low` 过滤是正确性的必要条件：环形数组里的 N 个槽位可能全部是「N 个窗口之前残留的桶」，如果跳过世代校验，就会把整轮的错误历史累加进来，在故障恢复后造成误熔断。

**5. 一个必须内化的推论：窗口长度不是常数**
任意时刻 `count()` 实际覆盖的区间是 `[cur - (N-1)w, t]`，长度 = `(N-1)w + (t mod w) + 1`，在 `[(N-1)w, Nw]` 之间浮动。N=2、w=500ms 时，「1 秒窗口」实际在 500ms~1000ms 之间摆动；N=10、w=100ms 时在 900~1000ms 之间。因此「增大 N」的直接收益不是让窗口变精确，而是**缩小窗口长度的波动与边界误差**；代价是读 O(N)、内存 N×事件类型数、以及更频繁的世代切换（临界区争用上升）。

**6. 熔断状态机如何消费这些统计**
- Hystrix：HealthCounts 由 SUCCESS / FAILURE / TIMEOUT / THREAD_POOL_REJECTED / SEMAPHORE_REJECTED 聚合；仅当 totalRequests ≥ circuitBreakerRequestVolumeThreshold（默认 20）才计算 errorPercentage，超过 circuitBreakerErrorThresholdPercentage（默认 50%）进入 OPEN；睡眠 sleepWindowInMilliseconds（默认 5s）后进入 HALF_OPEN，放行单请求探测，成功则 CLOSED 并重置窗口。
- Sentinel：三种策略（慢调用比例 SLOW_REQUEST_RATIO、异常比例 ERROR_RATIO、异常数 ERROR_COUNT）都从同一个 ArrayMetric 的滑动窗口取值。注意 RT 是在 entry exit 时写入的，所以窗口内的 RT 统计天然滞后一个完整的请求生命周期。

**7. 与前端已有概念的映射**
- `throttle(fn, T)` 是 N=1 的固定窗口，状态只有一个 lastTime 变量；滑动窗口是它的 N 桶推广，用 O(N) 空间换取对「最近 T」的连续刻画。
- 埋点的「批量缓冲 + 定时 flush」是定时器驱动的**主动清理**；滑动窗口是世代号驱动的**惰性淘汰**，与 LRU、WeakMap、分代 GC 同属「用访问时校验替代周期性扫描」的族。
- rAF 中统计最近 60 帧平均帧时长是**样本窗口**（按条数）；HystrixRollingPercentile 的桶内 int[] 采样与之一致，而 RollingNumber 是**时间窗口**（按时长）。混用两者会造成语义错位。
- 类型系统对照：TS 的 interface 是编译期结构化契约、运行期被完全擦除，不占内存、不参与分派；Java 的 interface 是运行期存在的名义类型，方法分派经 vtable/itable，但同样不承载状态。这里的 Bucket / WindowWrap 是运行期真实对象，其字段 `gen` 本身就是正确性依据——典型的「编译期约束 vs 运行期状态机」之别。

**8. Hystrix 与 Sentinel 的实现差异**
- 默认窗口 / 桶：Hystrix 10000ms / 10 桶（w=1s）；Sentinel 1000ms / 2 桶（w=500ms，并按资源维度另有分钟级 LeapArray）。
- 世代标识：Hystrix 用 floor(t/w)；Sentinel 用 floor(t/w)*w（保留时间语义，便于日志与调试）。
- 重置并发控制：Hystrix 对 Bucket.windowNumber 做 CAS，失败即让位；Sentinel 用 per-window 的 ReentrantLock + 双重检查。
- 计数器：均为 LongAdder 数组；Sentinel 的 MetricBucket 额外维护 minRt，用于慢调用判定。
- 分位数：Hystrix 另有 HystrixRollingPercentile，桶内存 int[] 采样（默认每桶 100 条），读时合并全部桶样本排序取分位，读放大为 O(N·K·log(N·K))，且是近似分位；Sentinel 不提供窗口分位数，只有平均 RT 与 minRt。

### 3. 基础代码与实战验证
```text
以下实现是 Sentinel LeapArray 的最小骨架（1s 窗口 / 2 桶），去掉了全部框架依赖，可以单文件编译运行；时钟可注入，便于确定性验证。

    import java.util.concurrent.atomic.LongAdder;
    import java.util.function.LongSupplier;

    public class MiniSlidingWindow {
        static final int  N        = 2;             // 桶数量
        static final long INTERVAL = 1000L;         // 总窗口
        static final long W        = INTERVAL / N;  // 单桶时长 500ms

        static final class Bucket {
            volatile long gen = Long.MIN_VALUE;     // 世代号 = 桶起点时间；惰性重置的版本标识
            final LongAdder pass  = new LongAdder();
            final LongAdder block = new LongAdder();
            void reset(long s) { gen = s; pass.reset(); block.reset(); }  // 只能在临界区内调用
        }

        private final Bucket[] arr   = new Bucket[N];
        private final Object[] locks = new Object[N];
        LongSupplier clock = System::currentTimeMillis;   // 生产环境应使用单调时钟

        public MiniSlidingWindow() {
            for (int i = 0; i < N; i++) { arr[i] = new Bucket(); locks[i] = new Object(); }
        }

        Bucket current(long now) {
            int  i = (int) ((now / W) % N);   // 环形取模：时间轴 -> 槽位，O(1) 定位
            long s = now - now % W;           // 当前桶的左边界，即本代的世代号
            Bucket b = arr[i];
            if (b.gen != s) {                 // 快路径命中则完全不加锁
                synchronized (locks[i]) {     // 判世代 + 清零 + 后续写入 必须构成原子段
                    if (b.gen != s) b.reset(s);   // 二次校验：否则会清掉并发线程刚写入的计数
                }
            }
            return b;
        }

        void pass()  { current(clock.getAsLong()).pass.increment();  }
        void block() { current(clock.getAsLong()).block.increment(); }

        long count(boolean isPass, long now) {
            long cur = now - now % W;
            long low = cur - (N - 1) * W;     // 有效世代下界，等价于 cur - INTERVAL + W
            long sum = 0;
            for (Bucket b : arr) {
                long g = b.gen;               // volatile 读：可能读到正在切换的世代，统计场景可容忍
                if (g >= low && g <= cur)     // 关键：过滤上一轮残留的桶，否则读到脏值造成误熔断
                    sum += isPass ? b.pass.sum() : b.block.sum();  // LongAdder.sum() 是弱一致快照
            }
            return sum;
        }

        public static void main(String[] args) {
            long[] t = {0};
            MiniSlidingWindow w = new MiniSlidingWindow();
            w.clock = () -> t[0];

            t[0] = 0;   for (int i = 0; i < 3; i++) w.pass();  // 落入桶 [0,500)
            t[0] = 400; for (int i = 0; i < 4; i++) w.pass();  // 仍落桶 [0,500)，累计 7
            t[0] = 600; for (int i = 0; i < 5; i++) w.pass();  // 落入桶 [500,1000)

            t[0] = 999;  System.out.println(w.count(true, 999));   // 12：低界=0，两桶全计入
            t[0] = 1000; System.out.println(w.count(true, 1000));  // 5 ：低界抬到 500，桶 [0,500) 被过滤
            t[0] = 1400; System.out.println(w.count(true, 1400));  // 5 ：桶 [500,1000) 仍在窗口内
            t[0] = 1500; w.pass();                                 // 触发槽位 1 的世代切换，桶 [500,1000) 就地清零
            t[0] = 1600; System.out.println(w.count(true, 1600));  // 1 ：一次惰性重置回收整桶历史
        }
    }

执行输出：
    12
    5
    5
    1

最关键的两行：count() 中的「世代区间过滤」保证读正确性（不会把上一轮残留的桶算进来），current() 中的「DCL 二次校验」保证写正确性（不会丢掉并发写入）。删掉任意一行，输出都会错。把 N 改成 10、INTERVAL 保持不变，无需改动任何逻辑即可验证第 5 点推论：窗口长度波动从 500ms 收窄到 100ms 量级，但读循环从 2 次变成 10 次。
```

### 4. 常见误区与进阶思考
**误区一：认为滑动窗口统计的是「固定长度 T 的连续区间」**
反例：N=2、w=500ms 时，count() 覆盖的区间长度在 500ms~1000ms 之间浮动（= (N-1)w + t mod w + 1）。你以为在判断「最近 1 秒失败率」，实际可能是在判断「最近 0.5 秒」。由此推出两条实践结论：
- 调阈值必须连同 N 一起看。N 越小，窗口越短、分母越小，突发更容易被判超阈值（Sentinel 默认 N=2 是为 QPS 统计保留平滑性；Hystrix 选 10s/10 桶是为了让 5s 的 sleepWindow 有更长的观察期）；
- 绝不能把它当精确计数用于对账、计费或审计。它是控制面的近似度量，不是数据面的账本。

**误区二：认为存在后台定时清理，或认为环形数组保留的是「最近 N 个桶的历史快照」**
实际上是「单活跃桶 + N-1 个待复用槽位」：任意时刻只有当前世代对应的桶可写，其余槽位要么属于窗口内（只读历史），要么是上一轮的残留（读时按世代过滤掉）。既没有定时任务，也没有后台线程，空闲期零开销。推论：它不能回放时间序列，只能回答「当前窗口的聚合值」；要时间序列请用 Metrics 上报。
同源的次级误区：认为 `LongAdder.sum()` 是强一致快照。它是弱一致的，并发累加时可能读到中间态——这在统计场景可接受，但在「一致性校验」场景会导致偶发误判。

**思考题**
给定 Sentinel 默认配置 N=2、w=500ms：(a) 写出任意时刻 t 的 `count()` 实际覆盖的时间区间，并证明其长度落在 `[(N-1)w, Nw]` 内；(b) 若把 N 提到 10（w=100ms），窗口长度波动收窄到 900~1000ms，那么 Sentinel 为什么不默认用 10？请从读放大 O(N)、世代切换频率（临界区争用概率与 LongAdder 数组重建次数）、内存（N × 事件类型数 × LongAdder 分段 cell 数）三个维度给出定量分析，并说明在什么业务特征下你会主动把它调大；(c) 进阶：当发生 NTP 时钟回拨时，`floor(t/w) mod N` 会指向一个世代号**大于**当前计算值的槽位。Sentinel 的处理是返回该旧桶并标记为 deprecated，而不是重置它。这个选择避免了什么更严重的问题？如果改为「直接重置为当前世代」，环形数组的哪条不变式会被破坏，会引发什么连锁后果？
