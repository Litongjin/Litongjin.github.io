---
title: "每日基础技术总结 · 2026-11-04 · Web Animations API (WAAPI)：取代 CSS Transition 的控制力与时间轴管理"
date: 2026-11-04 07:01:40
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-04 · Web Animations API (WAAPI)：取代 CSS Transition 的控制力与时间轴管理

## 📚 今日主题

> **Web Animations API (WAAPI)：取代 CSS Transition 的控制力与时间轴管理**（前端底层与计算机基础）

### 1. 核心概念速览
Web Animations API（WAAPI）是 W3C Web Animations 规范把浏览器内部动画模型（timing model + keyframe effect + 播放状态机）暴露为 JavaScript 可持有、可读取、可修改的接口。它的本质是一套可寻址的时间状态机，并非又多了一套动画语法。

对象图：
Animation（播放器：状态机 + 时间控制）
  ├─ startTime / holdTime / playbackRate / currentTime / playState
  └─ effect: KeyframeEffect（把时间映射为属性值的纯函数）
        └─ keyframes + { duration, delay, endDelay, easing, iterations, direction, fill, composite }
AnimationTimeline（时间源）：document.timeline（随文档帧推进）/ ScrollTimeline / ViewTimeline

关键事实：CSS Transition 与 CSS Animation 在引擎内部同样被编译为这套模型的实例（el.getAnimations() 返回的 CSSTransition、CSSAnimation 都是 Animation 的子类）。也就是说，CSS 是“样式变更 diff 隐式派生动画”的声明式入口，WAAPI 是“显式构造并持有动画对象”的命令式入口，两者共享同一时间模型与同一条渲染管线。

它解决的问题（全部来自 CSS Transition 的结构性缺陷）：
1. 不可寻址：transition 对象没有句柄，随 style recalc 生灭，无法在运行时针对某个具体动画做操作。
2. 不可精确控制：不能 pause、seek、变速、倒放、finish，只能通过 duration/delay 间接影响。
3. 中断语义不可预测：连续修改目标值会触发 reversing shortening factor 与 transition 重定向规则，实际时长被缩短且不可控。
4. 无运行时可变性：无法在播放中替换 keyframes 或更新 timing。
5. 生命周期通知粗糙：只有 transitionend/transitioncancel 事件，无 Promise 语义，被打断时的行为不明确。
6. 无统一时间源：无法与滚动、视图进入等时间源编排。

在整个体系中的位置：它位于 JavaScript 主线程与 Blink 的 animation 层、合成器（cc::AnimationHost / cc::AnimationTimeline）之间，是渲染流水线中“时间维度”的公开接口。掌握它，是判断一个动画是否掉帧、是否占用主线程、是否触发 layout/paint 的唯一可靠依据；也是实现手势联动、可逆中断过渡、FLIP、滚动驱动动画、View Transitions 的前提。对于有工程经验但缺乏渲染层认知的开发者，这里是从“写样式”跨到“理解引擎如何按帧推进状态”的分界线。

### 2. 底层原理剖析
一、时间模型的分层（Web Animations timing model，规范定义，不是实现细节）

timeline time（document.timeline.currentTime，约等于当前帧时间）
  → animation currentTime：
      running: (timelineTime - startTime) * playbackRate
      paused / idle: holdTime
  → effect local time = currentTime
  → active time = localTime - delay，再 clamp 到 [0, activeDuration]
  → phase 判定：before / active / after（由 delay、activeDuration、endDelay 与 fill 决定该阶段是否仍应用样式）
  → iteration progress：overallProgress = activeTime / iterationDuration；再按 iterations 取整取余得 simpleIterationProgress；按 direction 翻转得 directedProgress；施加 easing 得 transformedProgress
  → keyframe 插值：在相邻 keyframe 的 offset 区间内 lerp，再按 composite（replace / add / accumulate）合成到目标属性值

核心洞察：Animation 只存三个独立状态——startTime（时间轴锚点）、holdTime（冻结值）、playbackRate（时间缩放因子）。currentTime 是这三者的纯函数，不独立存储。这正是“可 seek、可倒放、可变速而不破坏 keyframes”的根本原因。

伪代码：

play():
  if playState == finished: currentTime = (playbackRate >= 0 ? 0 : activeDuration)
  startTime = null; holdTime = null; playState = pending
  // pending 阶段浏览器在下一个 rendering opportunity 前 resolve ready，并写入 startTime = timelineTime
pause():
  holdTime = currentTime; startTime = null; playState = paused
cancel():
  清空 startTime / holdTime，effect 不再应用到目标元素，finished 以 AbortError reject
finish():
  currentTime = (playbackRate >= 0 ? activeDuration : 0)
setKeyframes() / updateTiming():
  只改 effect（值映射），不触碰 Animation 的时间状态，因此可在播放中热替换

二、线程模型：什么时候真的“不占主线程”

1. 仅当（a）动画属性属于可合成集合（transform、opacity、filter 等），且（b）目标元素已被提升为合成层时，动画才会被序列化为 cc::AnimationCurve 下发到 compositor 线程。此时主线程被长任务阻塞，动画仍按 timeline 推进——因为时间模型副本在合成器侧。
2. 非可合成属性（width / left / margin / background-color / box-shadow / top）走 main-thread ticking：每帧由 timeline 采样 → 修改 style → Recalc Style → Layout → Paint → Commit。
3. 在 rAF 中每帧读 getComputedTiming()、getBoundingClientRect() 再写样式，会强制主线程同步布局（layout thrashing），等于主动放弃合成器路径。

三、与 requestAnimationFrame 手写插值的差异（前端已有的最接近概念）

rAF 方案把时间模型放在 JS 里：你持有 t，自己算 progress，自己写 style。主线程掉一帧，这个 t 就“丢”了，动画在时间轴上被拉长，且无法与浏览器其它动画同步。
WAAPI 的时间模型由引擎持有，采样发生在渲染流水线的固定位置（与合成器时钟同源），主线程卡顿不会让动画时间轴失真，且 getAnimations() 能把 CSS 侧与 JS 侧的动画纳入同一批对象统一编排。

四、与 CSS Transition 的本质异同（类比“结构化类型 vs 名义类型”）

| 维度 | CSS Transition | WAAPI |
| 对象身份 | 无句柄，样式 diff 的副产物，形状由外部状态推断（结构化） | 有独立身份的实例，有生命周期与引用（名义） |
| 触发模型 | 计算值变化隐式生成 | 构造函数显式声明 |
| 时间控制 | 仅 duration/delay | currentTime 读写、seek、变速、倒放、finish |
| 运行时可变更 | 只能改类名/变量间接影响 | setKeyframes / updateTiming 直接改 effect |
| 完成通知 | transitionend 事件，冒泡、语义粗糙 | ready / finished Promise，cancel 时 AbortError reject |
| 时间源 | 固定为文档时钟 | document.timeline / ScrollTimeline / ViewTimeline |
| 中断行为 | reversing shortening factor，时长被缩短，不可预测 | 在中断点精确续播，位置与速率完全可控 |

五、动画替换（Animation.replaceState）

fill 生效且已 finished 的动画会被浏览器自动移除（removed），以释放合成层与样式优先级。若要保留终态视觉效果，必须 commitStyles() 把计算值写回元素内联样式，再 persist() 阻止自动移除；否则 getAnimations() 中的对象消失、样式回退。这是 WAAPI 从 CSS 动画继承来的“填充只负责呈现，不负责持久化”的语义边界。

### 3. 基础代码与实战验证
```text
// index.html
// <div id=box style=width:100px;height:100px;background:#333></div>
// <button id=seek>seek</button>

const box = document.getElementById('box');

// 1) KeyframeEffect 只描述「时间 → 属性值」的纯映射，不携带任何播放状态
const effect = new KeyframeEffect(
  box,
  [
    { transform: 'translateX(0px)',   opacity: 1,   offset: 0 },
    { transform: 'translateX(200px)', opacity: 0.5, offset: 0.5, easing: 'ease-in' },
    { transform: 'translateX(400px)', opacity: 1,   offset: 1 }
  ],
  {
    duration: 1000,        // iterationDuration：iteration progress 的分母
    delay: 200,            // 决定 localTime -> activeTime 的偏移与 before phase 长度
    iterations: 1,         // 影响 simpleIterationProgress 的取整取余
    direction: 'normal',   // 决定 directedProgress 是否翻转
    fill: 'both',          // 决定 before/after phase 是否仍然应用样式
    composite: 'replace'   // 与其它动画叠加时的合成算子
  }
);

// 2) Animation = (timeline, effect) 的绑定；时间源与值映射完全解耦
const anim = new Animation(effect, document.timeline);

// 3) idle -> pending -> running：startTime 在首个渲染时机才被写入
anim.play();
console.log(anim.playState);   // pending：startTime === null，此时 currentTime 由 holdTime 承载
anim.ready.then(() => {
  // 到这里 startTime 才非 null；currentTime = (timelineTime - startTime) * playbackRate
  console.log('startTime', anim.startTime, 'currentTime', anim.currentTime);
});

// 4) 精确 seek：写 currentTime 只是修改引擎内部时间状态，渲染在下一帧采样时发生
document.getElementById('seek').onclick = () => {
  anim.pause();            // 冻结：holdTime = currentTime，startTime = null
  anim.currentTime = 500;  // localTime=500 -> activeTime=500-200=300 -> progress=30%
  const ct = effect.getComputedTiming();
  // 读取引擎按规范算出的中间量，验证上面的分层映射
  console.log(ct.localTime, ct.activeDuration, ct.progress, ct.currentIteration);
  // 注意：此处不会触发同步样式重算，视觉更新在下一个 rendering opportunity
};

// 5) playbackRate 是时间缩放因子，负值即倒放，不改变 keyframes
function setSpeed(v) {
  const t = anim.currentTime;   // 变速前先取锚点，保证位置连续
  anim.playbackRate = v;
  anim.currentTime = t;
}

// 6) 生命周期 Promise：cancel 会以 AbortError reject，用于可取消的动画编排
anim.finished
  .then(() => console.log('finished'))
  .catch((e) => console.log('aborted', e.name));  // DOMException: AbortError

// 7) 同构验证：声明式产生的 CSS transition 也是同一批 Animation 对象
box.style.transition = 'transform 300ms';
box.style.transform = 'translateX(50px)';
requestAnimationFrame(() => {
  box.getAnimations().forEach((a) => {
    console.log(a.constructor.name);  // CSSTransition —— 与手动构造的 Animation 同类
    a.pause();                        // 声明式动画同样可被命令式接管
  });
});

// 8) fill 生效且 finished 的动画会被自动移除，需显式持久化终态
// anim.commitStyles();  // 把当前计算值写回内联样式
// anim.persist();       // 阻止 replaceState 变为 removed

// 9) 性能验证：给它改成 anim.effect.target.style.width 的动画（非可合成属性），
// 在 DevTools Performance 面板中会看到每帧 Recalc Style + Layout，而 transform 版本没有。
```

### 4. 常见误区与进阶思考
误区一：把 WAAPI 当成“CSS 动画的 JS 语法糖”。方向恰好相反：CSS 动画是 WAAPI 模型的上层投影。由此衍生出两个具体错误：(a) 认为 anim.currentTime = x 会立刻触发渲染——它只改写引擎内部时间状态，样式值在下一个 rendering opportunity 才被采样，同步读取 getComputedTiming() 读到的是“时间模型真值”，而非“已渲染的像素真值”；(b) 忽略 pending 阶段——play() 之后、ready resolve 之前 startTime 为 null，currentTime 实际由 holdTime 承载，此时用 (now - startTime) 手算进度会得到 NaN 或错值。

误区二：认为“用了 WAAPI 就自动走合成器、零主线程开销”。合成器路径有三个必要条件：属性可合成（transform/opacity/filter）、元素已提升为合成层、没有同属性的非可合成动画或每帧读写样式的代码干扰它。给 width/left/background 做 WAAPI 动画依然是每帧 Recalc Style + Layout + Paint；在 rAF 里轮询 getComputedTiming() 或 getBoundingClientRect() 再写样式，会额外引入强制同步布局，反而比 CSS 路径更慢。另一处被忽略的语义是 fill：'forwards' 的动画在 finished 后会被浏览器按 replace 规范自动移除，导致视觉回退——必须 commitStyles() + persist() 显式接管终态。

思考题：实现一个“可中断、可反向、且中断瞬间位置与速度都连续”的拖拽回弹过渡（例如手指拖到任意位置后释放，元素带初速度弹性归位）。
1) 请只用 WAAPI 的时间模型（startTime / holdTime / playbackRate / currentTime，配合 effect.updateTiming() 与多个并行 Animation 的 composite: 'add'）设计完整的状态迁移，说明在哪些时刻必须把 currentTime 反解回 startTime、哪些时刻必须切换为 holdTime。
2) 为什么 CSS transition 在连续两次修改目标值时必然触发 reversing shortening factor、导致实际 duration 短于声明值，并且无法表达“以当前速度为初速度”的物理回弹？请从“transition 是由计算值 diff 派生”这一点出发论证。
3) 若把时间源换成 ScrollTimeline，playbackRate 与 currentTime 的语义分别发生什么变化？此时 pause() 意味着什么，为什么 scroll 驱动的动画不允许自由 seek？
