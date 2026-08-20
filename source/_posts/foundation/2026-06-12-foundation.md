---
title: "每日基础技术总结 · 2026-06-12 · 负载均衡最少连接与加权轮询算法"
date: 2026-06-12 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-12 · 负载均衡最少连接与加权轮询算法

## 📚 今日主题

> **负载均衡最少连接与加权轮询算法**（网络基础）

### 1. 核心概念速览
负载均衡最少连接与加权轮询是分布式系统中反向代理层调度请求的两类核心算法，本质是解决『多副本服务实例间如何分配请求』的决策问题。最少连接（Least Connections）以服务端当前活跃连接数为唯一调度依据，每次选择连接数最少的实例；加权轮询（Weighted Round Robin）则依据管理员预设的权重比例，以循环方式按权重分配请求，强调请求数量的确定性比例而非实时状态。二者位于OSI第七层（应用层）或L4传输层，是水平扩展、容量规划、故障转移的基础机制。专业工程师必须掌握其数学本质（权重归一化、计数器状态机）与分布式约束（无全局时钟、连接生命周期），因为任何微服务网关、消息队列消费者分配、数据库连接池都隐式或显式依赖此类决策逻辑，且其变体（平滑加权轮询、最少连接+权重）是生产环境的默认选择。

### 2. 底层原理剖析
最少连接的底层机制：维护每个后端实例的活跃连接计数器C_i，每次新请求到达时遍历所有实例，选择argmin(C_i)的实例，然后C_i+1；连接关闭时C_i-1。核心状态是连接计数的瞬时快照，无需预知请求处理时长，但隐含假设『连接数能近似反映负载』，适用于长连接（如WebSocket、数据库会话）。其伪代码：
```
onRequest(req):
  best = min_by(instances, key=active_conns)
  active_conns[best]++
  forward(best, req)
onClose(instance):
  active_conns[instance]--
```
加权轮询的底层机制：为每个实例i分配权重W_i（正整数），维护当前轮询位置pos和当前权重current_weight。经典平滑算法（Nginx所用）使用最大公约数（GCD）和递推式：
```
gcd = gcd(W_1,...,W_n)
current_weight = 0
onRequest():
  repeat:
    current_weight += gcd
    if current_weight > total_weight: current_weight = min(W_i)  // 重置
    pos = (pos + 1) % n
  until W[pos] >= current_weight
  return pos
```
更常见的平滑加权轮询（Smooth WRR）维护每个实例的current_weight，每次选择当前权重最大的实例，然后将其current_weight减去总权重，其余实例加上自己的权重。本质是让高权重实例在时间轴上分布更均匀，避免突发连续请求打向同一实例。
与前端已有概念的对比：这类似于前端『事件循环』中的任务调度——轮询类似公平调度，最少连接类似动态优先级调度。但更贴切的对比是浏览器对资源请求的并发控制（如HTTP/1.1的每个域名最多6个连接），那是『最少连接』的客户端视角；而JavaScript的宏任务/微任务队列是固定优先级的轮询，没有动态权重。另一个对比：TypeScript的接口是编译期静态契约，而负载均衡算法的『接口』是运行时动态策略——前者关注类型形状，后者关注状态分布。工程师常误以为加权轮询是『按权重概率随机』，实则它是确定性序列，而最少连接是贪心算法，两者在数学上分别属于公平调度和在线负载近似。

### 3. 基础代码与实战验证
以下为极简Python实现，无框架依赖，展示最少连接与平滑加权轮询的核心状态机。
```python
class LeastConnectionsBalancer:
    def __init__(self, instances):
        # instances: 实例标识列表，如 ['s1','s2','s3']
        self.conns = {i: 0 for i in instances}

    def on_request(self, req):
        # 使用min()按连接数升序取第一个，即当前活跃连接最少的实例
        # 注意：min返回第一个最小值，符合贪心策略
        target = min(self.conns, key=lambda x: self.conns[x])
        self.conns[target] += 1  # 分配后递增连接计数
        # 实际转发逻辑省略，仅返回目标实例
        return target

    def on_connection_close(self, instance):
        # 连接关闭时必须递减，否则计数器会无限增长，导致算法失效
        self.conns[instance] -= 1


class SmoothWeightedRoundRobin:
    def __init__(self, weights):
        # weights: 字典 {实例名: 权重}，如 {'a':5,'b':1,'c':1}
        self.weights = weights
        # 当前权重表，初始为0，这是平滑算法的关键状态
        self.current = {i: 0 for i in weights}
        # 总权重，用于减法操作
        self.total = sum(weights.values())

    def next(self):
        # 1. 所有实例的当前权重加上各自的原始权重
        for i in self.weights:
            self.current[i] += self.weights[i]
        # 2. 选择当前权重最大的实例
        best = max(self.current, key=self.current.get)
        # 3. 被选中实例的当前权重减去总权重，为下一次选择让位
        self.current[best] -= self.total
        return best

# 验证：使用权重 {'a':5,'b':1,'c':1} 调用next() 7次，输出顺序应为 a,a,b,a,c,a,a（平滑分布）
```
注释已说明底层运作：最少连接依赖外部事件（连接关闭）来修正计数器；平滑加权轮询仅依赖数学运算，无需任何反馈。

### 4. 常见误区与进阶思考
误区一：认为加权轮询的权重是『请求处理能力的比例』，但实际权重只是分配请求数量的比例，与实例CPU、内存等真实容量无关。若不结合健康检查与动态权重，高权重实例可能因过载而雪崩。最典型的错误是只配权重不配超时，导致连接堆积在慢实例上。
误区二：认为最少连接算法能精确反映负载，忽略了连接持续时间与资源消耗的差异。例如两个实例连接数相同，但一个实例处理的是CPU密集的请求，另一个是I/O密集的请求，实际负载截然不同。最少连接只适合连接生命周期能代表工作量的场景。
进阶思考题：在一个无状态HTTP短连接场景下（每个请求一个连接），最少连接算法退化成什么？它与纯轮询有何区别？请从计数器变化频率和并发窗口的角度分析，说明为何在这种场景下最少连接几乎等价于轮询，以及什么情况下会真正产生差异。
