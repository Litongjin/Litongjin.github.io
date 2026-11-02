---
title: "每日基础技术总结 · 2026-11-03 · Intersection Observer API：视口检测原理与回调节流机制"
date: 2026-11-03 07:01:23
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-03 · Intersection Observer API：视口检测原理与回调节流机制

## 📚 今日主题

> **Intersection Observer API：视口检测原理与回调节流机制**（前端底层与计算机基础）

### 1. 核心概念速览
定义：Intersection Observer 是 W3C 定义的一组异步几何观察接口。它把目标元素的 layout box 与指定 root（缺省为顶层文档的布局视口）的 root intersection rectangle 求交；当相交状态跨越预设阈值时，以 IntersectionObserverEntry 的形式批量投递通知。

本质：它把『几何相交判定』从主线程的 scroll/resize 回调配合 getBoundingClientRect 轮询，下沉到浏览器渲染管线的固定阶段执行；采样策略由『按事件频率采样』改为『按状态跨越阈值边界采样』。由此同时获得两个性质：(1) 计算复杂度从 O(事件数 × 目标数) 降为 O(渲染更新次数 × 目标数)；(2) 不再在事件回调中读取几何，消除强制同步布局（layout thrashing）以及由此产生的长任务与 INP 劣化。

三要素：root（裁剪基准）、rootMargin（对 root 矩形的膨胀或收缩，用于预加载提前量与迟滞）、threshold（把连续相交比例离散化的边界数组）。目标注册进 observer 的内部 targets 列表，由文档级的 intersection observer 列表在每次渲染更新时统一驱动。

在体系中的位置：它属于浏览器『观察者原语』家族（MutationObserver、ResizeObserver、IntersectionObserver、PerformanceObserver）；四者的根本差异不在功能，而在投递时机与执行上下文（microtask、渲染更新内 layout 之后、渲染更新之后的 task）。它是 img loading=lazy、content-visibility、虚拟列表、图片墙分批渲染、广告可见性上报（IAB viewability）、曝光埋点的底层依赖。掌握它的必要性在于：这是少数几个调用时机由渲染管线而非调用方决定的 API，误用会产生跨帧时序 bug 与难以复现的漏报，并且直接决定长列表的首屏性能预算。

### 2. 底层原理剖析
一、几何定义（到底在算什么）
1. targetRect：目标元素 border box 经祖先 transform 映射到 root 坐标系后的轴对齐矩形（语义等价于 getBoundingClientRect，已包含变换）。
2. rootRect：root 为 null 时取顶层文档的 layout viewport（不是 visual viewport，双指缩放不改变它）；root 为元素时取其 content box。随后按 rootMargin 膨胀或收缩：四值顺序为 top right bottom left，百分比分别相对 root 的 height（上下）与 width（左右）。
3. 裁剪链：intersectionRect = targetRect ∩ rootRect，再依次与 target 到 root 之间每个产生 clip/overflow 剪切的包含块（含跨域 iframe 的 viewport rect）求交。因此 IO 天然感知 overflow:hidden 与 iframe 边界，但不感知由 clip-path、mask 或遮挡造成的『视觉』不可见。
4. 派生量：intersectionRatio = intersectionRect.area / targetRect.area（targetRect 面积为 0 时，相交矩形非空取 1、否则取 0）；isIntersecting 表示相交矩形非空。

二、计算时机：并入渲染更新，而非事件驱动

伪代码（一帧的渲染更新）：

渲染更新机会（每帧最多一次）
  步骤 1：运行 animation frame callbacks（你的 rAF 在此）
  步骤 2：样式计算与 layout（只有产生 layout box，几何才有意义）
  步骤 3：ResizeObserver 投递（layout 之后、paint 之前，按深度排序）
  步骤 4：paint 与 composite
  步骤 5：update intersection observations：对文档内所有 observer 的所有 target 计算一次相交；若某 target 的阈值索引或 isIntersecting 变化，则写入该 observer 的 queued entries
随后进入 notify intersection observers：queue a task（intersection observer task source），回调执行，参数为自上次投递以来的全量 entries（等价于 takeRecords 出队）

推论：
- 相交计算次数与 scroll 事件次数无关，只与渲染更新次数相关；页面不产生渲染时（后台标签、完全离屏）基本不计算。
- 回调不发生在计算的那一帧内，而在其后的 task 中；entry 是快照而非当前值，回调执行时布局可能已推进若干帧。
- 规范用一个 running 标志防重入：若回调执行期间又产生新 entry，不递归调用，而是再次排队，避免观察者自激。

三、通知条件：阈值索引状态机（所谓『节流』的真实来源）

对每个 target 逐帧执行：
  rect = computeIntersection(target, root)
  ratio = area(rect) / area(targetRect)  按上文退化规则
  idx = 第一个满足 ratio < thresholds[i] 的索引；若不存在则取 thresholds.length
  isIntersecting = rect 非空
  if idx != target.previousThresholdIndex 或 isIntersecting != target.previousIsIntersecting:
      target.previousThresholdIndex = idx
      target.previousIsIntersecting = isIntersecting
      observer.queuedEntries.push(entry)  每个 target 每帧至多 1 条

要点：
- observe() 时 previousThresholdIndex 初始化为 -1，故第一次计算必定投递一条初始 entry。这是初始化钩子，不应被当作『进入视口』事件使用。
- 每帧每 target 至多 1 条 entry；一次 callback 调用携带 N 条（N 为该 observer 本帧状态变化的 target 数）。所以节流的本质是：输出被离散化为『阈值跨越事件』，并由渲染帧率封顶（≤ 1 次/帧/目标），不是定时器。
- 阈值之间是采样空洞：一帧内 ratio 从 0 直接跳到 1（快速惯性滚动、程序化 scrollTop、布局跳变）只投递最终状态，中间阈值不补发。IO 是采样器，不是积分器。
- 回调内写入布局，会在下一次渲染更新中重新计算，可能再次跨越同一阈值，形成跨帧振荡（加载/卸载反复触发）。工程上用双阈值迟滞或状态机去抖。

四、与既有概念的对照

| 观察者 | 触发源 | 投递时机 | 典型误用 |
| scroll + getBoundingClientRect | 滚动事件 | 事件处理内，与帧不对齐 | 强制同步布局、每事件 O(N) 读取 |
| IntersectionObserver | 渲染更新阶段的几何计算 | queue a task，批量 | 当作可见性判定或连续进度 |
| ResizeObserver | 布局尺寸变化 | 同帧 layout 后、paint 前 | 回调内改尺寸造成循环 |
| MutationObserver | DOM 树变更 | microtask，当前任务结束前 | 认为能观察布局结果 |

另需与 CSS scroll-driven animations（animation-timeline: view()）区分：后者在渲染管线内由合成器驱动，不回到主线程，适合连续视觉进度；IO 适合离散状态机（懒加载、曝光、卸载）。二者互补而非替代。

### 3. 基础代码与实战验证
```text
// ============ 1. 最小可验证实现 ============
const target = document.querySelector('#probe');

const io = new IntersectionObserver((entries, observer) => {
  // 一次调用 = 一个 task 内该 observer 的 queued entries 全量出队
  // entries.length 可能 > 1：同帧内多个 target 的状态变化被合并到同一次调用
  for (const entry of entries) {
    // entry.time             : 相交被记录的时刻（performance.now 时基），不是回调执行时刻
    // entry.boundingClientRect: 记录时刻的几何快照，回调执行时布局可能已推进
    // entry.rootBounds       : 已应用 rootMargin 之后的 root 矩形
    // entry.intersectionRect : 裁剪链求交结果（targetRect ∩ rootRect ∩ 各祖先 clip）
    // entry.intersectionRatio = intersectionRect.area / boundingClientRect.area
    // entry.isIntersecting   : 相交矩形非空；与 ratio > 0 不等价（边缘相切时 ratio 为 0 但为 true）
    console.log(entry.target.id, entry.isIntersecting, entry.intersectionRatio, entry.time);
  }
}, {
  root: null,                        // null => 顶层文档的 layout viewport
  rootMargin: '100px 0px -25% 0px',  // 上扩 100px、下缩 root 高度的 25%（百分比基准是 root，不是 target）
  threshold: [0, 0.25, 0.5, 1]       // 必须非降序；仅当『阈值索引』变化才产生通知
});

io.observe(target); // previousThresholdIndex 初始为 -1，故首次计算必定投递一条初始 entry

// ============ 2. 验证『计算与投递解耦』 ============
// A. observe 之后同步读取：当前任务内不可能有 entry
console.log(io.takeRecords().length); // 0 —— 相交计算只在渲染更新阶段发生

// B. 改布局 + 强制 layout，依然不会产生 entry
target.style.marginTop = (window.innerHeight * 2) + 'px'; // 立刻移出视口
void target.getBoundingClientRect();                      // 强制同步布局
console.log(io.takeRecords().length);                     // 0 —— layout 不等于 intersection 计算

// C. rAF 处于渲染更新早期，本帧的相交计算尚未执行
requestAnimationFrame(() => {
  console.log('rAF', io.takeRecords().length);            // 通常只能拿到上一帧遗留的 entry（首次为 0）
  setTimeout(() => {
    console.log('task after frame', io.takeRecords().length); // >= 1：entry 已在本帧被计算并排队
  }, 0);
});

// ============ 3. 对照：传统实现暴露的两项成本 ============
window.addEventListener('scroll', () => {
  const r = target.getBoundingClientRect(); // 读几何：若本帧此前有写操作，则触发强制同步 layout
  const visible = r.top < innerHeight && r.bottom > 0;
  // 每个 scroll 事件执行一次；事件频率与渲染帧不对齐（同一帧可能派发多个 scroll）
}, { passive: true });
```

### 4. 常见误区与进阶思考
误区一：把 isIntersecting 当作『用户看得见』。IO 只解几何裁剪：被 position: fixed 遮罩覆盖、被 z-index 更高的兄弟节点压住、opacity: 0、filter 全透明、visibility: hidden（仍有 layout box）时，isIntersecting 依然为 true；只有 display: none、无 layout box、或被祖先 overflow / iframe 视口裁掉才为 false。真正的曝光判定需要『面积比 + 连续停留时长』的状态机（IAB viewability 语义），或 Chromium 的 IntersectionObserver v2 trackVisibility（基于遮挡查询，非标准、跨浏览器不可用），或 elementFromPoint 采样（昂贵且与合成器结果不一致）。把 IO 直接当曝光信号，是埋点虚高与漏报的根因。

误区二：把『回调节流』理解成按时间节流，并把 entry 当连续量。IO 的输出是阈值跨越的采样点：一帧内 ratio 从 0 跳到 1 只投递 ratio=1 的一条，中间阈值不补发；callback 是 task，与 rAF 无同步关系；entry.boundingClientRect 是计算时刻快照，回调执行时可能已过期。在回调里用 entry 的几何做布局决策，等于基于陈旧数据决策，典型后果是跨帧振荡（元素反复跨越阈值，触发反复加载/卸载），以及回调内写样式触发布局、放大单帧成本。

三个高频细节：rootMargin 的百分比基准是 root 的宽/高；root 必须是 target 的可达祖先（跨域 iframe 内只能为 null，否则抛错）；observer 持有 target 的强引用并持续参与每帧计算，必须 unobserve / disconnect。

思考题：某虚拟列表用 IntersectionObserver（threshold: [0]，rootMargin: '0px 0px 200px 0px'）做『进入预取区即拉取下一页』。移动端惯性滚动时一帧内 scrollTop 变化 800px，导致 3 个本应进入预取区的行被整帧跳过。(1) 这 3 行是否会产生 entry？请基于阈值索引状态机给出推理。(2) 若某元素在连续两帧内 ratio 在 0.5 附近来回跨越 threshold: [0, 0.5]，最坏情况会产生多少条 entry、多少次 callback 调用？与 threshold: [0] 相比差多少？(3) 要保证『任何被跳过的行最终都会被预取』，不变式应建立在 IO 的哪个性质之上（提示：不是『entry 的到达』，而是『某个时刻的计算结果』）？给出实现方向。
