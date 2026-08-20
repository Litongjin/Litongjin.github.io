---
title: "每日基础技术总结 · 2026-07-12 · CoreDNS 的 ndots 与搜索域对 Pod 域名解析的影响"
date: 2026-07-12 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-12 · CoreDNS 的 ndots 与搜索域对 Pod 域名解析的影响

## 📚 今日主题

> **CoreDNS 的 ndots 与搜索域对 Pod 域名解析的影响**（DevOps 与云原生）

### 1. 核心概念速览
ndots 是解析器（glibc/musl/Go net 包）用于判定域名是否达到 FQDN（完全限定域名）阈值的整数参数；search domain 列表是解析器在域名未达阈值时按序追加的后缀候选集。二者共同构成客户端侧的"名称展开策略"，定义于 /etc/resolv.conf，作用于 getaddrinfo() 调用之后、DNS 报文发出之前。Kubernetes 通过 kubelet 向每个 Pod 注入 ndots:5 与三层搜索域（<namespace>.svc.cluster.local、svc.cluster.local、cluster.local），使任何点数低于 5 的域名——包括仅 4 点的完整服务名 svc-a.ns-b.svc.cluster.local——都会被依次展开，产生多次串行 DNS 查询（默认 3 个搜索域 + 裸名兜底 = 4 次候选，继承节点搜索域时更多）。该机制位于 DNS 解析链路最前端（客户端 resolver 层），是"CoreDNS 延迟低但业务侧解析慢"这类现象的根源。专业工程师必须掌握，因为服务发现时延分布、DNS 超时故障定位、以及高性能网关的解析优化，全部建立在对该机制的精确理解之上。

### 2. 底层原理剖析
展开算法（以 glibc res_search 与 Go net 包 nameList 为参考实现）：

    dots = count(name, '.')
    if name 以 '.' 结尾:
        返回 [name]                    # 显式 FQDN，跳过搜索域
    elif dots >= ndots:
        返回 [name + "."]              # 达到阈值，按绝对名单次查询
    else:
        candidates = []
        for domain in search_list:
            candidates.append(name + "." + domain + ".")
        candidates.append(name + ".")   # 裸名作为 TLD 兜底
        for cand in candidates:          # 逐个发送，严格串行等待
            resp = query(cand)
            if resp.rcode != NXDOMAIN:
                return resp              # NOERROR 即终止，即使 A 记录为空
        return NXDOMAIN

关键机制：

1. 展开完全发生在客户端 resolver 内部。CoreDNS（或任何上游 DNS）收到的永远是展开后的绝对名，原始短名不会出现在 DNS 报文中，服务端无法感知或干预此过程。
2. 查询严格串行。每个候选必须等到响应（NXDOMAIN 或超时）后才尝试下一个；最坏情况总延迟 = 候选数 × (RTT + 超时重试)。glibc 默认单次超时 5 秒，可被 options timeout:n 覆盖。
3. 终止条件是"非 NXDOMAIN"而非"有 A 记录"。若候选返回 NOERROR 但 A 记录为空（如存在 CNAME 指向外部域名），resolver 立即停止并返回空结果，不会继续尝试后续候选——这是很多排障场景中被忽略的细节。
4. Kubernetes 默认 ndots:5 的语义根源：svc.cluster.local 自身含 2 点，加上 service.namespace 共 4 点，仍小于 5。因此"看似完整"的服务域名同样被展开，这就是 Kubernetes 中每次 getaddrinfo 实际产生 4~5 个查询的根本原因。
5. ndots:5 的设计意图：支持 service.namespace（1 点）这种便捷写法通过搜索域正确展开（如 api-gw.prod 追加 svc.cluster.local 后命中 api-gw.prod.svc.cluster.local）。降低 ndots 会破坏这一特性。

与前端知识体系对比：该机制与 Node.js 模块解析算法同构——require('foo') 不含 / 时，沿各级 node_modules 目录顺序查找，首个命中即返回；含 / 时直接按路径解析。ndots 相当于"specifier 是否含 /"的判定；search 列表相当于 node_modules 目录链。也等价于 webpack 的 resolve.extensions 候选后缀数组。区别在于：前端是本地文件系统 I/O（毫秒级），DNS 展开是网络 RTT（超时秒级），同一线性搜索策略在后者被放大为显著时延。

### 3. 基础代码与实战验证
```text
package main

import (
    "context"
    "fmt"
    "net"
    "os"
    "sync/atomic"
    "time"
)

func main() {
    raw, _ := os.ReadFile("/etc/resolv.conf")
    fmt.Println("--- /etc/resolv.conf ---")
    fmt.Println(string(raw)) // 确认 search 与 ndots 实际配置

    var counter int64

    r := &net.Resolver{
        PreferGo: true, // 强制使用 Go 内置 resolver，自主实现 ndots/search 展开逻辑
        Dial: func(ctx context.Context, network, address string) (net.Conn, error) {
            atomic.AddInt64(&counter, 1) // 每次真实 DNS 报文（UDP 每包一个连接）都会经过此函数
            return (&net.Dialer{Timeout: 2 * time.Second}).DialContext(ctx, "udp", "10.96.0.10:53")
        },
    }

    lookup := func(name string) {
        atomic.StoreInt64(&counter, 0)
        ips, err := r.LookupIP(context.Background(), "ip4", name)
        fmt.Printf("%-52s -> 查询次数=%d IP=%v err=%v", name, atomic.LoadInt64(&counter), ips, err)
        fmt.Println()
    }

    // 场景1: 短名，0 点 < ndots=5。Go 的 nameList 生成 4 个候选，
    // 首候选 my-service.default.svc.cluster.local. 即命中，共 1 次查询。
    lookup("my-service")

    // 场景2: 完整服务名，4 点 < ndots=5。仍被展开为 4 个候选：
    //   my-service.default.svc.cluster.local.default.svc.cluster.local.  (NXDOMAIN)
    //   my-service.default.svc.cluster.local.svc.cluster.local.          (NXDOMAIN)
    //   my-service.default.svc.cluster.local.cluster.local.              (NXDOMAIN)
    //   my-service.default.svc.cluster.local.                            (命中)
    // 共 4 次查询，前 3 次为多余开销。
    lookup("my-service.default.svc.cluster.local")

    // 场景3: 带尾点的显式 FQDN。nameList 识别尾点后直接返回单个候选，
    // 完全跳过搜索域展开，仅 1 次查询。
    lookup("my-service.default.svc.cluster.local.")
}

// 注意：此程序须在 Kubernetes Pod 内运行（或环境中 /etc/resolv.conf 包含
// search 列表与 options ndots:5），否则 Go 解析器读取的是宿主机的 DNS 配置。
```

### 4. 常见误区与进阶思考
误区一：认为"完整服务名（FQDN）不会触发搜索域展开"。实际判定只看点数与 ndots 的数值关系，与名字"看起来完整"无关。my-svc.my-ns.svc.cluster.local 仅 4 点，小于默认 5，仍会被展开为 4 个候选依次查询（前 3 个 NXDOMAIN）。只有显式带尾点（.）的绝对名，或点数 >= ndots 的名字，才会跳过搜索域。这一误区导致无数团队在 CoreDNS 层反复排查性能问题，而真正的开销发生在客户端 resolver 的串行候选尝试中。

误区二：把 ndots 改成 0 来消除多余查询。ndots=0 意味着任何名字（包括 my-service）点数都 >= 0，被直接视为 FQDN 查询 my-service.——CoreDNS 将其作为顶级域处理并返回 NXDOMAIN，同命名空间服务解析彻底失效。正确做法是：应用代码内使用带尾点的绝对服务名（service.namespace.svc.cluster.local.），同时配合 ndots:1；或保留 ndots:5 但接受短名查询的多候选开销，并利用 CoreDNS 缓存降低实际 RTT。

思考题：Pod 位于 namespace prod，resolv.conf 为 search prod.svc.cluster.local svc.cluster.local cluster.local + options ndots:5。应用调用 getaddrinfo("api-gw.prod")（1 点）。请写出 resolver 按序发送的全部 DNS 查询，并指出哪一次命中。随后将 ndots 改为 1，解释为什么 api-gw.prod 的解析会失败，以及 Kubernetes 默认选择 ndots:5 的完整代价与收益——这决定了你在真实生产环境中应如何权衡解析延迟与命名便利性。
