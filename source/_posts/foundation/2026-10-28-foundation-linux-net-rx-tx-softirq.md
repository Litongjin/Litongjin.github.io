---
title: "每日基础技术总结 · 2026-10-28 · Linux网络栈中Softirq处理Net_RX与Net_TX的流程差异"
date: 2026-10-28 07:03:24
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-28 · Linux网络栈中Softirq处理Net_RX与Net_TX的流程差异

## 📚 今日主题

> **Linux网络栈中Softirq处理Net_RX与Net_TX的流程差异**（后端基础）

### 1. 核心概念速览
softirq 是 Linux 中断下半部中在编译期静态注册（enum NR_SOFTIRQS）的一类延迟执行上下文：不可睡眠、同一 CPU 上同类串行、不同 CPU 上并行，通过 per-CPU pending 位图置位，由 irq_exit 尾部的 invoke_softirq、进程上下文的 do_softirq/local_bh_enable，或 ksoftirqd 内核线程执行。NET_RX_SOFTIRQ 与 NET_TX_SOFTIRQ 是网络数据面仅有的两个软中断向量，二者在触发源、数据结构、执行上下文的来源、预算机制上完全不同：

- NET_RX_SOFTIRQ（net_rx_action）：由硬件收包中断触发。驱动在中断处理中调用 napi_schedule，把 napi_struct 挂入 per-CPU softnet_data.poll_list 并置位 NET_RX_SOFTIRQ；net_rx_action 以 netdev_budget（默认 300 包）与 netdev_budget_usecs（默认 2000us）为上限，轮询调用 napi->poll（weight 默认 64），逐包上送 netif_receive_skb 进入协议栈。它是 RX 的驱动引擎，是收包路径的必经环节。
- NET_TX_SOFTIRQ（net_tx_action）：正常路径不会触发。它承担两个补偿职责：其一，回收 softnet_data.completion_queue——驱动在 TX 完成后用 dev_kfree_skb_irq/napi_consume_skb 把 skb 挂入队列，延迟到软中断中批量 kfree_skb，避免在硬中断上下文做耗时释放；其二，执行 softnet_data.output_queue——当 __dev_xmit_skb 抢占 qdisc 的 __QDISC_STATE_RUNNING 失败，或 __qdisc_run 的 quota（dev_tx_weight，默认 64）耗尽时，通过 __netif_schedule 把 Qdisc 挂入本 CPU output_queue 并置位 NET_TX_SOFTIRQ，由 net_tx_action 择机运行 qdisc_run。

本质区别一句话：RX 软中断是接收方向的同步主路径（事件驱动 + 预算轮询），TX 软中断是发送方向的异步化补偿机制（避锁、批量化、延迟回收）。

它位于内核数据面最热路径，直接决定 pps、尾延迟、CPU 分布与缓存局部性。netdev_budget、netdev_max_backlog、NAPI weight、RPS/RFS、XPS、TC_BQL、GRO、XDP、eBPF tc hook 全部围绕这条路径工作；不理解 softirq 的两个向量，就无法解释软中断 CPU 打满、ksoftirqd 抢占业务、time_squeeze 增长、TX 抖动等生产问题，也无法正确评估 DPDK/XDP 旁路的价值边界。

### 2. 底层原理剖析
一、per-CPU 数据结构（struct softnet_data，DECLARE_PER_CPU）：
  poll_list        — napi_struct 链表，待 poll 的设备实例（RX）
  input_pkt_queue  — 非 NAPI 驱动的入口队列与 RPS backlog（RX）
  completion_queue — 待释放的 sk_buff 链表（TX 回收）
  output_queue     — 待运行的 Qdisc 链表（TX 补偿）

二、RX 路径（事件驱动 + 预算轮询）：
  硬中断 handler → napi_schedule(napi) → ____napi_schedule：
      list_add_tail(&napi->poll_list, &sd->poll_list)
      if (!test_and_set_bit(NET_RX_SOFTIRQ, pending)) __raise_softirq_irqoff(NET_RX_SOFTIRQ)
  → irq_exit() → !in_interrupt && local_softirq_pending() → invoke_softirq() → do_softirq()
  → net_rx_action()：
      budget = netdev_budget
      time_limit = jiffies + usecs_to_jiffies(netdev_budget_usecs)
      local_irq_disable(); 遍历 sd->poll_list
      napi_poll(n, &budget) 返回 0 表示本轮 work==weight（未清空），list_move_tail 保留在 poll_list
      budget<=0 || time_after(jiffies, time_limit) → __raise_softirq_irqoff(NET_RX_SOFTIRQ) 并 break
      （softnet_stat 第三列 time_squeeze 即计此事件；ksoftirqd 在中断风暴/预算反复耗尽时接管）
  驱动 poll 内部：取 RX ring → napi_gro_receive → netif_receive_skb → ip_rcv → …（若配置 RPS，则在 netif_receive_skb 前 enqueue_to_backlog 并 ____napi_schedule 到目标 CPU，制造跨核 RX softirq）

三、TX 路径（进程上下文为主，软中断为溢出补偿）：
  sendmsg → dev_queue_xmit → __dev_queue_xmit → 选 txq（XPS）→ __dev_xmit_skb：
    qdisc_run_begin(q) 成功（test_and_set __QDISC_STATE_RUNNING）：
        qdisc_enqueue → __qdisc_run（sch_direct_xmit → ndo_start_xmit）→ 若 quota 耗尽且 qdisc 非空 → __netif_schedule(q) → qdisc_run_end
    qdisc_run_begin 失败（另一执行流正持有 RUNNING）：
        qdisc_enqueue → __netif_schedule(q)：test_and_set __QDISC_STATE_SCHED，挂 sd->output_queue，raise NET_TX_SOFTIRQ，立即返回用户态路径（不自旋、不阻塞）
  net_tx_action()：
    阶段 1：local_irq_disable 取走 completion_queue 整链；local_irq_enable 后逐 skb __kfree_skb（批量释放，摊薄中断延迟）
    阶段 2：local_irq_disable 取走 output_queue 整链；逐个 qdisc_run(q) + qdisc_run_end(q)（重新获得 RUNNING 位，恢复队列）

四、与前端已有体系的映射：
  - net_rx_action 的 budget + time_limit + 未完成项 list_move_tail 留待下轮 ≈ libuv 事件循环的 poll 阶段：一轮处理有限个就绪 fd，处理不完顺延到下一 tick；time_squeeze ≈ 一轮未清空队列被迫让出；ksoftirqd ≈ 把溢出工作转交给后台线程/下一 tick 的慢路径。
  - completion_queue 的批量延迟释放 ≈ V8 的延迟清理/惰性析构或 Node 的批次回调：不在硬中断（等价于同步、不可阻塞的中断栈）里执行 kfree_skb（等价于执行析构/GC），而是入队后批量处理，压缩临界区。
  - __QDISC_STATE_RUNNING ≈ JS 单线程的单一执行栈：同一时刻只有一条执行流能跑 qdisc；抢不到锁时不忙等，而是把工作变成异步任务推进 output_queue —— 与 Promise.then 把后续推到微任务队列而非阻塞当前栈同构。
  - 同 CPU 同类 softirq 串行、跨 CPU 并行 ≈ Worker 模型：不同 CPU 可并行处理各自队列，但单 CPU 上对同一 poll_list 严格串行；softirq 不可被抢占，但可被硬中断打断（相当于在同步栈中插入更高优先级的中断处理）。

五、差异归纳：RX 由设备事件发起、上下文承接自硬中断、有全局预算、语义是生产并上送；TX 由系统调用发起、上下文承接自进程或 ksoftirqd、预算由 qdisc quota 与 RUNNING 位控制、语义是提交后的异步补偿与资源回收。

### 3. 基础代码与实战验证
```text
内核关键路径伪代码（对应 net/core/dev.c，去掉错误处理）：

struct softnet_data {
    struct list_head    poll_list;         // RX: 待轮询 NAPI 实例
    struct sk_buff_head completion_queue;  // TX: 待释放 skb（硬中断里只入队，不 free）
    struct Qdisc       *output_queue;      // TX: 抢锁失败的 qdisc
};
DECLARE_PER_CPU(struct softnet_data, softnet_data);

// RX 侧：中断里仅挂链 + 置位，绝不做重活
void ____napi_schedule(struct softnet_data *sd, struct napi_struct *napi) {
    list_add_tail(&napi->poll_list, &sd->poll_list);         // O(1) 入链
    if (!test_and_set_bit(NET_RX_SOFTIRQ, &pending))         // per-CPU 位图去重
        __raise_softirq_irqoff(NET_RX_SOFTIRQ);              // 只置位，真正执行在 irq_exit
}

// RX 侧：带预算的轮询循环，这是 time_squeeze 的唯一来源
void net_rx_action(struct softirq_action *h) {
    int budget = netdev_budget;                              // 默认 300
    unsigned long time_limit = jiffies + usecs_to_jiffies(netdev_budget_usecs); // 默认 2000us
    local_irq_disable();
    while (!list_empty(&sd->poll_list)) {
        struct napi_struct *n = list_first_entry(&sd->poll_list, ...);
        if (napi_poll(n, &budget) == 0)                      // 0 = 已达 weight 上限，未清空 ring
            list_move_tail(&n->poll_list, &sd->poll_list);   // 本轮放弃，下轮继续
        if (budget <= 0 || time_after(jiffies, time_limit)) {
            __raise_softirq_irqoff(NET_RX_SOFTIRQ);          // 自我重挂，让出 CPU；time_squeeze++
            break;                                           // 剩余工作由下一次 do_softirq/ksoftirqd 承担
        }
    }
    local_irq_enable();
}

// TX 侧：两件事，批量回收 + 兜底跑 qdisc
void net_tx_action(struct softirq_action *h) {
    struct sk_buff *clist;
    local_irq_disable();
    clist = sd->completion_queue.next;                       // 整链摘走
    sd->completion_queue.next = NULL;
    local_irq_enable();
    while (clist) { struct sk_buff *skb = clist; clist = skb->next; __kfree_skb(skb); } // 批量析构

    struct Qdisc *q = sd->output_queue;
    sd->output_queue = NULL;                                 // 摘链后即可开中断
    while (q) { struct Qdisc *next = q->next_sched;
                qdisc_run(q);                                // 重新抢 RUNNING 位并发送
                qdisc_run_end(q); q = next; }
}

// TX 侧：发送路径的分叉点，决定是否 raise NET_TX_SOFTIRQ
int __dev_xmit_skb(struct sk_buff *skb, struct Qdisc *q, ...) {
    if (qdisc_run_begin(q)) {                                // test_and_set __QDISC_STATE_RUNNING
        qdisc_enqueue(skb, q, &to_free);
        __qdisc_run(q);                                      // quota(dev_tx_weight=64) 内直接下发驱动
        qdisc_run_end(q);                                    // quota 耗尽时内部已 __netif_schedule
    } else {
        qdisc_enqueue(skb, q, &to_free);
        __netif_schedule(q);                                 // 挂 output_queue + raise NET_TX_SOFTIRQ，立即返回
    }
}

用户态验证（无需框架，纯内核观测）：

# 1) 确认预算参数
sysctl net.core.netdev_budget net.core.netdev_budget_usecs net.core.dev_weight

# 2) 基线：NET_RX 应随收包量线性增长，NET_TX 在单流无竞争场景下增长远低于 RX
cat /proc/softirqs | grep -E 'NET_RX|NET_TX'

# 3) 制造 RX 预算耗尽：小 ring + 大流量，观察 softnet_stat 第 3 列 time_squeeze
for i in 1 2 3 4; do ping -f -c 200000 <gw> >/dev/null & done
awk '{print "cpu", NR-1, "processed", $1, "dropped", $2, "time_squeeze", $3}' /proc/net/softnet_stat

# 4) 制造 TX 侧 qdisc 竞争：单 qdisc 多进程并发送，NET_TX 软中断计数显著上升
for i in $(seq 32); do (while :; do dd if=/dev/zero bs=1400 count=1 2>/dev/null | nc -u -q0 <peer> 9000; done) & done
cat /proc/softirqs | grep NET_TX

# 5) 追踪两个软中断的真实执行上下文（区分 irq_exit / ksoftirqd / 进程上下文）
perf record -e irq:softirq_entry -e irq:softirq_exit -ag -- sleep 5
perf script | grep -E 'net_rx_action|net_tx_action' | head
```

### 4. 常见误区与进阶思考
误区一：把 NET_TX_SOFTIRQ 当成发送主路径，用 /proc/softirqs 的 NET_TX 计数等同于发包量。事实是：无 qdisc 竞争的发送完全在进程上下文（sendmsg 调用栈）内完成 enqueue 与 sch_direct_xmit，根本不 raise NET_TX_SOFTIRQ；只有在 __QDISC_STATE_RUNNING 竞争失败、__qdisc_run 的 quota（dev_tx_weight）耗尽，或驱动走 completion_queue 延迟回收时才会触发。因此单流、无竞争场景下 NET_TX 软中断时间接近噪声；一旦多进程共用单 qdisc 或加了限速/tbf/netem 类 qdisc，NET_TX 才会暴涨，此时 sendto 的抖动来自软中断抢锁与批处理延迟，而不是驱动慢。

误区二：把 net_rx_action 等同于「硬中断退出时执行的代码」。它可能来自三个上下文：irq_exit 的 invoke_softirq、进程上下文的 do_softirq/local_bh_enable（典型如 RPS enqueue_to_backlog 后的处理、loopback 在发送栈中的回环收包、socket 系统调用返回前的下半部执行）、以及 ksoftirqd（中断风暴或预算反复耗尽时接管）。所以 perf 抓到的 net_rx_action 调用栈可能挂在 ksoftirqd 而非中断栈上；同样，softirq 不可被抢占但可被硬中断打断，这意味着「软中断里 CPU 占用高」既可能是流量大，也可能是 backlog 不足导致的重复轮次空转，必须结合 softnet_stat 的 dropped（backlog 溢出）与 time_squeeze（预算耗尽）区分，前者调 netdev_max_backlog，后者调 netdev_budget/weight 或上 RPS。

思考题：一台 4 核机器跑单个 UDP 发送进程，观测到 NET_TX 软中断的 CPU 占用长期高于 NET_RX，且 sendto 的 p99 抖动明显放大。请从 __dev_xmit_skb 对 __QDISC_STATE_RUNNING 的 test_and_set 竞争、__qdisc_run 的 dev_tx_weight quota 轮转、__netif_schedule 把 Qdisc 挂入 output_queue 后由 net_tx_action 重新 qdisc_run 的完整回路出发，解释这个「TX 软中断 > RX 软中断」倒挂的成因；并说明启用 XPS 做 per-CPU txq 绑定、或让 TC_BQL 收紧字节配额后，哪些环节的竞争与重排会被消除、哪些只是被移位到别处。
