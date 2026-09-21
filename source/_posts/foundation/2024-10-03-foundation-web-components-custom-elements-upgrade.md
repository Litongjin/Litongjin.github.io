---
title: "每日基础技术总结 · 2024-10-03 · Web Components Custom Elements 升级阶段（Upgrade）的生命周期钩子时序"
date: 2024-10-03 20:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-10-03 · Web Components Custom Elements 升级阶段（Upgrade）的生命周期钩子时序

## 📚 今日主题

> **Web Components Custom Elements 升级阶段（Upgrade）的生命周期钩子时序**（前端底层与计算机基础）

### 1. 核心概念速览
Custom Elements 'Upgrade'（升级）阶段是浏览器解析 HTML 后、将自定义元素节点实例化并插入 DOM 树之前的关键过渡期。其核心机制在于：当浏览器检测到符合自定义标签名的元素时，若在连接文档前已完成定义注册，则触发 upgrade 回调；若未注册，则进入待办列表等待注册后延迟调用。该阶段解决的是组件生命周期早期状态初始化与 DOM 挂载前的数据/依赖注入问题。在计算机体系位置中，它位于 HTML Parser 语义解析之后、DOM Mutation Observer 激活之前，属于浏览器渲染引擎的‘创建-链接’子系统的核心逻辑。专业工程师必须掌握此阶段，因为它是理解 Web Components 与 React/Vue 虚拟 DOM diff 算法根本差异的关键——原生组件的挂载时机早于框架的挂载检查点，且在 SSR/ hydrate 场景下，upgrade 钩子是服务端输出脚本与客户端行为对齐的唯一同步窗口。

### 2. 底层原理剖析
底层运行机制遵循严格的时序与条件分支：
1. HTML Parser 遇到开始标签 <my-element>。
2. 调用 document.createElement('my-element') 生成 Element 实例。
3. 检查 globalState.registeredTags 是否包含 'my-element' 且对应的 constructor.prototype.connectedCallback 已存在。
   - Case A (已注册)：立即执行 upgrade() 函数（即 CustomElementRegistry.prototype.define 中注册的 callback）。此时 element.isConnected 为 false，但构造器（constructor）可能已在 createElement 时被调用（取决于浏览器实现细节，通常 constructor 在 createElement 时触发，而 upgrade 在插入文档或定义完成时触发，标准规定 define 后若有未升级节点则批量调用 upgrade）。
4. 若 Case B (未注册)：将元素对象加入 pendingUpgradeQueue。
5. 后续任意时刻调用 customElements.define('my-element', MyClass)：
   - 更新全局注册表。
   - 遍历 pendingUpgradeQueue，对每个节点执行 upgrade 回调。
6. 元素插入文档树：触发 connectedCallback。

对比 TS Interface：TS Interface 是编译时静态类型契约，零运行时成本，用于确保数据结构一致性；Custom Element Upgrade 是运行时动态生命周期钩子，由浏览器引擎内核驱动，属于对象原型链扩展后的实例初始化协议。二者维度不同：前者是‘形态约束’，后者是‘行为时序’。

### 3. 基础代码与实战验证
```text
// 模拟浏览器内部处理逻辑的核心伪代码，展示升级阶段与时序关系
const elementsInDocument = []; // 存储当前文档中所有自定义元素节点

function parseHTML(htmlString) {
  const parser = new DOMParser();
  const doc = parser.parseFromString(htmlString, 'text/html');
  
  doc.querySelectorAll('custom-counter').forEach(el => {
    // 关键点 1: createElement 瞬间触发 constructor
    // 但此时还未关联 Prototype 的 lifecycle methods
    if (!customElements.get('custom-counter')) {
      // 关键点 2: 若未注册，入队等待（Deferred Upgrade）
      scheduleUpgrade(el);
    } else {
      // 关键点 3: 若已注册，立即触发 Upgrade 钩子
      el.__upgrade__();
    }
  });
  return doc;
}

customElements.define('custom-counter', class extends HTMLElement {
  constructor() {
    super();
    // 构造函数优先于 Upgrade 钩子执行
    this.state = { value: 0 }; 
  }
  
  static get observedAttributes() { return ['min']; }
  
  attributeChangedCallback(name, oldVal, newVal) {
    console.log('Attr Change:', name, oldVal, newVal);
  }
  
  // 【核心】Upgrade 钩子
  // 注：现代浏览器规范中，若无特定需求，极少直接使用此钩子替代 connectedCallback，
  // 因其执行时机可能多次触发（如先 createElement 再 appendChild，中间穿插 define）
  // 此处仅演示其存在性与时机：发生在 constructor 之后，connectedCallback 之前
  upgrade(oldInstance) {
    console.log('[UPGRADE] Node attached to registry but not yet connected to DOM.');
    if (oldInstance) {
      // 复用旧实例状态（如从序列化恢复）
      this.state = oldInstance.state;
    }
  }
  
  connectedCallback() {
    console.log('[CONNECTED] Ready for rendering.');
  }
});
```

### 4. 常见误区与进阶思考
误区 1：混淆 Constructor 与 Upgrade/ConnectedCallback 的执行顺序与作用域。Constructor 在 createElement 时调用，此时节点尚未连接到文档（isConnected=false），不可进行需访问父级布局或样式的内容操作；Upgrade 钩子主要用于状态迁移，而 ConnectedCallback 才是 UI 渲染的正确起点。误用 Constructor 做副作用操作会导致闪烁或布局错误。

误区 2：忽视异步定义的竞态条件。如果在 JS 模块加载延迟导致 customElements.define 晚于 HTML 解析完成，所有已存在的自定义元素都处于‘半成品’状态（只有原生 Element 能力）。开发者常在此阶段丢失事件监听或属性绑定，因为这些操作若写在 Upgrade 外且假设组件已就绪，将在 define 执行后被覆盖或忽略。

思考题：在 SSR（服务端渲染）场景下，服务器输出的 HTML 中包含大量自定义元素，但未执行 client-side script。当浏览器加载完 bundle 并执行 customElements.define 时，请推导这些已存在于 DOM 中的元素会经历怎样的内部状态流转？特别是，如果组件逻辑依赖于初始化的 DOM 高度计算，仅在 Upgrade 钩子中处理为何不足以解决问题，必须结合何种机制才能确保数据一致性？
