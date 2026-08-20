---
title: "每日基础技术总结 · 2026-07-07 · Kubernetes QoS 类与 OOM 分数计算及驱逐顺序"
date: 2026-07-07 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-07 · Kubernetes QoS 类与 OOM 分数计算及驱逐顺序

## 📚 今日主题

> **Kubernetes QoS 类与 OOM 分数计算及驱逐顺序**（DevOps 与云原生）

### 1. 核心概念速览
Kubernetes QoS（Quality of Service）类是Pod级别的资源承诺等级抽象，由kubelet根据Pod中每个容器的requests和limits计算得出，分为Guaranteed、Burstable、BestEffort三档。它解决资源竞争时的优先级与隔离问题：调度器据此决定哪些Pod可以被过量分配，kubelet和内核据此决定在内存压力下先驱逐谁、先杀死谁。本质上是节点资源管理中的分层防御策略——QoS是契约，驱逐和OOM是执行机制。在整个计算机体系中，它属于操作系统资源管理与分布式调度之间的桥接层，是云原生运行时对Linux CGroup和OOM killer的标准化封装。专业工程师必须掌握，因为QoS直接决定应用在资源不足时的存活概率，是容量规划、故障恢复和SLA保障的基础。

### 2. 底层原理剖析
1. QoS分类规则（kubelet计算）：遍历Pod所有容器，若每个容器都设置了requests和limits且对应资源request==limit（CPU和内存都相等），则Pod为Guaranteed；若所有容器都未设置requests和limits，则为BestEffort；其余情况为Burstable。注意：只要有一个容器不满足Guaranteed条件，且至少有一个容器设置了request或limit，整个Pod即为Burstable。
2. OOM分数机制：Linux内核为每个进程维护oom_score，值越高越容易被OOM killer杀死。进程的实际oom_score等于基础内存占用比例加上oom_score_adj（-1000~1000）。Kubelet在创建容器时设置进程的oom_score_adj：
   - Guaranteed：-998（接近不可杀，除非内核无选择）
   - BestEffort：1000（最高，优先被杀）
   - Burstable：若容器设置了内存limit，则 oom_score_adj = -998 + (1000 * memory_request) / memory_limit；若未设置内存limit，则为0。该公式使得Burstable中内存请求接近限制的容器adj接近-998（更不容易被杀），而请求远小于限制的容器adj在[-998, 2)之间，始终低于BestEffort。这保证了Guaranteed < Burstable < BestEffort的杀除优先级。
3. kubelet驱逐机制：kubelet的eviction manager监控节点内存压力，当超过软/硬阈值时，按以下顺序选择Pod进行驱逐（终止）：
   - 首先驱逐BestEffort且使用量超过其requests的Pod；
   - 然后驱逐Burstable且使用量超过其requests的Pod，按内存使用量/requests比例从高到低排序；
   - 最后驱逐Guaranteed且使用量超过其limits的Pod（若未超limit则不驱逐）。
   驱逐排序基于QoS类作为主键，同类中按资源使用相对于request的比例排序。
4. 与前端概念的异同：这与前端中“优先级调度”类似，比如浏览器对任务队列的宏/微任务优先级，或React的Lane调度。但本质区别在于：前端优先级是启发式的、可抢占的调度策略，服务于UI响应；而QoS是资源承诺的契约，与内核的OOM和cgroup的限流强制绑定，具有硬性语义。类似于TypeScript的接口与Java的接口——它们都描述契约，但TS的接口是结构化类型，可隐式实现，而Java的接口是显式的类型约束；QoS同样是显式的资源契约，但由系统强制实施。

### 3. 基础代码与实战验证
```text
def compute_qos(containers):
    # containers: list of dict, each has 'request_mem', 'limit_mem'
    all_guaranteed = all(c.get('request_mem') is not None and c.get('limit_mem') is not None and c['request_mem'] == c['limit_mem'] for c in containers)
    all_besteffort = all(c.get('request_mem') is None and c.get('limit_mem') is None for c in containers)
    if all_guaranteed:
        return 'Guaranteed'
    if all_besteffort:
        return 'BestEffort'
    return 'Burstable'

def oom_score_adj(qos, request_mem, limit_mem):
    if qos == 'Guaranteed':
        return -998
    if qos == 'BestEffort':
        return 1000
    # Burstable
    if limit_mem and limit_mem > 0:
        return int(-998 + (1000 * request_mem) / limit_mem)
    return 0

# 示例：一个Pod有两个容器，c1有request和limit，c2无任何限制
containers = [
    {'name': 'c1', 'request_mem': 128*1024*1024, 'limit_mem': 256*1024*1024},
    {'name': 'c2', 'request_mem': None, 'limit_mem': None},
]
qos = compute_qos(containers)
print('QoS:', qos)  # Burstable
for c in containers:
    r = c['request_mem'] or 0
    l = c['limit_mem'] or 0
    print(f"{c['name']} oom_score_adj: {oom_score_adj(qos, r, l)}")

# 驱逐顺序模拟
def eviction_order(pods):
    qos_priority = {'BestEffort':0, 'Burstable':1, 'Guaranteed':2}
    def ratio(p):
        if p['request'] == 0:
            return float('inf') if p['usage'] > 0 else 0
        return p['usage'] / p['request']
    return sorted(pods, key=lambda p: (qos_priority[p['qos']], -ratio(p)))

pods = [
    {'name': 'best', 'qos': 'BestEffort', 'usage': 100, 'request': 0},
    {'name': 'burstable', 'qos': 'Burstable', 'usage': 150, 'request': 100},
    {'name': 'guaranteed', 'qos': 'Guaranteed', 'usage': 200, 'limit': 200},
]
print('驱逐顺序:', [p['name'] for p in eviction_order(pods)])  # best, burstable, guaranteed
```

### 4. 常见误区与进阶思考
常见误区1：将kubelet驱逐与内核OOM杀死混为一谈。实际上，驱逐发生在内存达到eviction threshold之前，由kubelet主动终止整个Pod；OOM发生在内存完全耗尽时，由内核选择进程杀死。QoS影响两者的顺序，但机制和时点不同。例如，一个BestEffort Pod可能先被驱逐，但若驱逐不及时，其进程也可能被OOM killer杀掉。
常见误区2：认为Guaranteed Pod绝对安全。即使QoS为Guaranteed，如果Pod的某个容器内存使用超过其limit，内核可能因cgroup内存限制触发OOM，导致进程被杀。此外，节点上系统进程或非Pod进程导致内存耗尽时，内核可能被迫杀死任何进程，Guaranteed Pod只是oom_score_adj最低，优先级最低，但并非绝对免疫。
思考题：一个Pod包含两个容器：容器A设置requests.memory=128Mi，limits.memory=256Mi；容器B没有设置任何requests或limits。请计算该Pod的QoS类，并给出两个容器各自的oom_score_adj。如果节点内存压力上升，kubelet驱逐该Pod时，相对于另一个完全Guaranteed的Pod，谁先被驱逐？为什么？
