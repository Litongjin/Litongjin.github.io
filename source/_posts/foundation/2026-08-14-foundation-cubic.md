---
title: "每日基础技术总结 · 2026-08-14 · 拥塞控制中的 CUBIC 算法"
date: 2026-08-14 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-14 · 拥塞控制中的 CUBIC 算法

## 📚 今日主题

> **拥塞控制中的 CUBIC 算法**（网络基础）

### 1. 核心概念速览
CUBIC 是 TCP 拥塞控制算法，属于丢包检测型（Loss-Based）算法族，是 Linux 内核自 2.6.19 起默认的拥塞控制实现。其本质是：在拥塞窗口（cwnd）的更新过程中，不再采用传统的线性增长（如 Reno 的 AIMD），而是将 cwnd 增长函数定义为以丢包发生时刻的窗口值为拐点的三次多项式。该算法解决的核心问题是：在高带宽高延迟（BDP 大）网络中，传统算法在丢包后窗口减半再缓慢线性恢复，导致带宽利用率极低；CUBIC 通过使窗口增长函数独立于 RTT（基于绝对时间），实现了更激进但可控的窗口恢复，从而逼近网络容量。机制上，CUBIC 在每次丢包事件后记录窗口值 W_max，随后 cwnd 按时间 t 的三次函数增长，先快速恢复到 W_max 附近，再在 W_max 附近缓慢探测（凹增长），最后进入凸增长搜索新的带宽上限。它在整个体系中的位置：属于传输层 TCP 协议栈的关键组件，直接影响端到端吞吐量、延迟与公平性；作为专业工程师必须掌握，因为现代数据中心、云网络、视频流媒体、CDN 等场景的吞吐瓶颈往往由拥塞控制决定，且 CUBIC 是理解 BBR、Copa 等现代算法的基础，其设计思想（利用时间而非 RTT、模型化带宽探测）与前端性能优化中的『网络调度』『资源加载策略』有深刻同构性。

### 2. 底层原理剖析
CUBIC 底层机制可用以下状态机与数学函数精确描述：
1. 状态与变量：维护 cwnd（当前窗口）、W_max（丢包发生时的窗口值）、K（从当前时间到达到 W_max 所需的时间常数）、epoch_start（本轮增长起始时间）、t（从 epoch_start 起经过的绝对时间）。
2. 丢包事件（进入拥塞避免）：当检测到丢包（如 ACK 重复或 SACK），执行 cwnd = 0.8 * W_max（快速乘性减小，比 Reno 的 0.5 更保守），同时记录 W_max = 当前 cwnd 的减小前值（实际算法中 W_max = cwnd 在乘性减之前的窗口值），并重置 epoch_start = 当前时间，K = cbrt( (W_max - cwnd) / C )，其中 C 为缩放常数（默认 0.4）。
3. 窗口增长函数：在每个 ACK 到达时，若处于正常拥塞避免，计算 t = 当前时间 - epoch_start（注意：t 是绝对时间差，与 RTT 无关）。cwnd 的更新目标值为 W_target = W_max + C * (t - K)^3。实际 cwnd 是向 W_target 缓慢逼近，每收到一个 ACK 增加 (W_target - cwnd) / cwnd。
4. 增长阶段分解：
   - 快速恢复阶段（t < K）：由于 (t-K) 为负，W_target 小于 W_max，但 cwnd 是从乘性减小后的值 0.8W_max 出发，向 W_max 逼近，因此曲线凹向上，增长迅速，旨在快速利用空闲带宽。
   - 平稳探测阶段（t ≈ K）：W_target 接近 W_max，增长放缓，以避免在接近上次拥塞点时再次引发丢包，形成凹区间。
   - 凸搜索阶段（t > K）：W_target 超过 W_max，且因三次方特性，增长逐渐加速（凸区间），用于探测是否存在更大的可用带宽，直到下一次丢包。
5. 与 RTT 的关系：传统 Reno 的窗口增长速度依赖 RTT（每个 RTT 增加 1 个 MSS），导致不同 RTT 流之间的公平性极差；CUBIC 将增长函数建立在绝对时间上，所有流无论 RTT 长短，在相同时长内达到相同的窗口增长量，因此实现了 RTT 公平性（即带宽按 RTT 反比分配的现象被削弱）。
6. 伪代码（Linux 内核简化）：
   on_ack(ack):
       if (epoch_start == 0) epoch_start = now;   // 初始化
       t = now - epoch_start;
       target = W_max + C * (t - K)^3;
       cwnd = cwnd + (target - cwnd) / cwnd;      // 增量式逼近
   on_loss():
       W_max = cwnd;
       cwnd = cwnd * 0.8;                        // beta 因子
       epoch_start = now;
       K = cubic_root((W_max - cwnd) / C);
   对比前端已有概念：前端中常见『防抖与节流』基于时间控制函数执行频率，CUBIC 则是基于时间控制窗口增长速度；防抖/节流使用时间常量（如 300ms）与事件发生时刻，CUBIC 使用绝对时间差与三次方程计算目标窗口，两者本质都是『时间驱动』而非『事件驱动』。另一对比：前端状态管理（如 Redux）中的 reducer 根据 action 类型计算新状态，CUBIC 根据『丢包事件』和『时间流逝』更新 cwnd 状态，但 Redux 是确定性纯函数，CUBIC 则依赖实时网络测量，是一个闭环反馈系统。更贴切的类比是：CUBIC 的 W_max 类似前端缓存中的『上次资源加载的峰值水位』，增长函数类似『预加载策略』——先快速逼近上次峰值，再缓慢试探是否可突破，这与 HTTP 缓存中的启发式过期算法（基于上次响应时间）思想一致。

### 3. 基础代码与实战验证
以下为不依赖任何框架的 Python 模拟代码，用于验证 CUBIC 的窗口增长逻辑（忽略 ACK 细节，仅模拟时间步长下的 cwnd 演化）：

```python
import math
import matplotlib.pyplot as plt

class Cubic:
    def __init__(self, cwnd=10, w_max=0, beta=0.8, C=0.4):
        self.cwnd = cwnd
        self.w_max = w_max
        self.beta = beta
        self.C = C
        self.epoch_start = None
        self.K = 0
        self.t = 0  # 模拟绝对时间（秒）

    def on_loss(self):
        # 丢包时：记录当前窗口为 W_max，然后乘性减小窗口
        self.w_max = self.cwnd
        self.cwnd = self.cwnd * self.beta
        self.epoch_start = self.t
        # K = cbrt((W_max - cwnd) / C)，即恢复至 W_max 所需时间
        self.K = ((self.w_max - self.cwnd) / self.C) ** (1/3)

    def on_time_advance(self, dt):
        # 每过一个时间步 dt，更新绝对时间
        self.t += dt
        if self.epoch_start is None:
            return
        # 计算距 epoch_start 的绝对时间差（CUBIC 不依赖 RTT，只依赖真实时间）
        elapsed = self.t - self.epoch_start
        # 三次函数目标窗口：W_target = W_max + C*(t - K)^3
        target = self.w_max + self.C * ((elapsed - self.K) ** 3)
        # 实际 cwnd 向 target 缓慢靠近，模拟 ACK 驱动的增量（简化：按比例逼近）
        self.cwnd += (target - self.cwnd) * 0.1  # 0.1 为收敛系数，仅演示
        # 若发生丢包（cwnd 超过某容量阈值），触发 on_loss
        if self.cwnd > 200:  # 假设网络容量为 200
            self.on_loss()

# 模拟 100 秒，每 0.1 秒步进
cubic = Cubic(cwnd=10)
cubic.w_max = 100  # 初始假设上次拥塞窗口
cubic.epoch_start = 0
cubic.K = ((cubic.w_max - cubic.cwnd) / cubic.C) ** (1/3)
history = []
for i in range(1000):
    cubic.on_time_advance(0.1)
    history.append((cubic.t, cubic.cwnd))
    # 手动触发丢包演示：在 t=30 时人为丢包
    if 29.9 < cubic.t < 30.1:
        cubic.on_loss()

# 打印关键点
times = [p[0] for p in history]
cwnds = [p[1] for p in history]
print("时间: 窗口")
for t, c in zip(times[::50], cwnds[::50]):
    print(f"{t:.1f}s: {c:.2f}")
```

关键代码注释：
- `on_loss()` 中的 `self.w_max = self.cwnd` 记录拥塞点；`self.cwnd *= 0.8` 对应 beta 因子；`K` 计算恢复时间，本质是三次方程根，表示从当前减小的窗口增长回 W_max 的预期时间。
- `on_time_advance` 中的 `elapsed` 为绝对时间，体现了 CUBIC 与 RTT 解耦的核心。`target` 是三次函数，当 elapsed < K 时 target < W_max，曲线凹；当 elapsed > K 时 target > W_max，曲线凸。
- `self.cwnd += (target - self.cwnd) * 0.1` 模拟 ACK 驱动的平滑逼近，实际内核使用更精细的每 ACK 增量，但原理相同。
- 丢包阈值 `200` 模拟网络瓶颈，一旦超过即丢包，触发 `on_loss` 重置状态，形成锯齿波。
- 运行该代码可观察到：丢包后窗口先快速恢复，在接近 W_max 时增速放缓，随后增速加快，再次探测到新容量后丢包，形成周期循环。

### 4. 常见误区与进阶思考
常见误区 1：认为 CUBIC 是『以丢包为唯一信号』，因此适用于所有网络。实际上 CUBIC 仅在检测到丢包后调整窗口，在缓冲区溢出（Bufferbloat）严重、路由器队列较深时，丢包信号滞后，导致时延升高；CUBIC 的快速恢复阶段会主动填满队列，造成高排队延迟。这是它与 BBR 等基于延迟的算法的本质区别——丢包型拥塞控制无法感知延迟变化，只把丢包当作拥塞信号。
常见误区 2：混淆 CUBIC 的『三次函数增长』与『每 RTT 增长固定字节』。CUBIC 的窗口增长是绝对时间的函数，与 RTT 无关；同一时刻下，不同 RTT 的流增长速度相同（公平性），但同一流在窗口值远离 W_max 时增长快、接近时增长慢，这是曲线的凹/凸特性，不是简单地按 RTT 倍速增长。很多工程师误以为 CUBIC 就是『快速增加、慢启动』，忽略了其基于时间的三次方程本质。
进阶思考题：假设一条链路中有两条 TCP 流，RTT 分别为 10ms 和 100ms，均采用 CUBIC，且共享同一瓶颈带宽。在稳态下（周期丢包），两者的窗口平均值之比是否等于 RTT 之比的倒数？请从 CUBIC 的窗口增长与丢包事件的耦合关系分析，给出比值表达式，并说明如果改为 BBR（基于延迟），该比值将如何变化，为什么？
