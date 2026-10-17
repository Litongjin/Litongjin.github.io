---
title: "每日基础技术总结 · 2026-10-18 · Web Components Custom Elements 生命周期钩子与升级（Upgrade）过程"
date: 2026-10-18 07:05:15
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-18 · Web Components Custom Elements 生命周期钩子与升级（Upgrade）过程

## 📚 今日主题

> **Web Components Custom Elements 生命周期钩子与升级（Upgrade）过程**（前端底层与计算机基础）

### 1. 核心概念速览
Custom Elements 是 Web Components 标准的核心组成部分，本质是一套由浏览器原生提供的、允许开发者向 HTML 解析器注册自定义标签名并绑定其行为的机制。生命周期钩子（Lifecycle Callbacks）是定义在自定义元素类上的、由浏览器在特定时刻自动调用的标准方法，包括 `connectedCallback`, `disconnectedCallback`, `attributeChangedCallback`, 和 `adoptedCallback`。这套机制解决了在不依赖任何外部框架的情况下，创建可复用、可封装、与框架无关的 UI 组件的问题。

升级（Upgrade）过程是 Custom Elements 的核心魔法。当一个自定义元素被定义（`customElements.define`）时，浏览器会扫描整个文档树，将所有已存在的、匹配的占位元素（undefined elements）原地“升级”为功能完备的自定义元素实例。这个过程包括：创建类的新实例、调用构造函数、并将原型链指向自定义元素的原型。理解升级机制是掌握 Custom Elements 异步、渐进增强特性的关键，它体现了浏览器如何将声明式的 HTML 与命令式的 JavaScript 逻辑无缝链接起来。专业工程师必须掌握它，因为这是理解现代前端组件化思想的基石，也是对浏览器渲染引擎、DOM 操作和原型链机制的深度实践。

### 2. 底层原理剖析
底层运行机制剖析：

1. **定义 (Definition)**: `customElements.define('my-el', class MyEl extends HTMLElement {...})` 调用将标签名 `'my-el'` 和一个类构造函数 `MyEl` 关联，存入浏览器内部的 `CustomElementRegistry`。此操作是全局且唯一的。

2. **解析与占位 (Parsing & Placeholding)**: 当 HTML 解析器遇到 `<my-el>` 时，如果它尚未被定义，解析器会将其视为一个普通的、未知的 `HTMLElement` 实例。它是一个“占位符”，没有自定义行为，但保留了标签名和属性。

3. **升级 (Upgrade)**: 当 `customElements.define` 完成后，`CustomElementRegistry` 会触发一个升级过程。这是一个同步过程（在 `define` 调用期间）：
   a. 浏览器遍历 DOM 树中所有未升级的、标签名为 `my-el` 的元素。
   b. 对每个元素，执行一个类似 `Reflect.construct(MyEl, [], MyEl)` 的操作来创建一个新的 `MyEl` 实例。
   c. 将这个新实例的 `[[Prototype]]` 链接到 `MyEl.prototype`，并将 DOM 中原有的占位元素替换为这个新实例。原始的占位元素被丢弃。
   d. **关键点**: 此时会立即调用新实例的 `constructor()`。

4. **生命周期钩子触发时机**:
   - `constructor()`: 元素实例被创建或升级时调用。此时元素还没有子节点，属性也未被解析。不应该在这里操作 DOM 或读取属性。
   - `connectedCallback()`: 元素首次被插入到文档的 DOM 中时调用。这是进行初始化设置（如获取属性、添加事件监听、渲染初始内容）的安全位置。
   - `disconnectedCallback()`: 元素从 DOM 中移除时调用。用于清理工作，如移除事件监听、取消网络请求，防止内存泄漏。
   - `attributeChangedCallback(name, oldValue, newValue)`: 当在 `static observedAttributes` 数组中声明的属性被添加、移除或更改时调用。这是一个响应式系统的基础。
   - `adoptedCallback()`: 元素被移动到新文档时调用（例如通过 `document.adoptNode`）。

**与前端工程师已有概念对比（以 React 为例）**:
- **注册机制**: `customElements.define` 类似于 React 的组件定义，但它是全局注册到浏览器，而非通过模块导入。
- **生命周期**: `connectedCallback` 类似于 `componentDidMount` / `useEffect(() => {...}, [])`。`disconnectedCallback` 类似于 `componentWillUnmount` / `useEffect` 的清理函数。`attributeChangedCallback` 类似于 `componentDidUpdate` 中对 props 变化的响应，但它是基于 HTML 属性的字符串变化。
- **实例化**: React 组件是 JavaScript 中的虚拟 DOM 对象，最终由 React Renderer 转换为 DOM。Custom Elements 类本身就是对真实 DOM 节点的描述，实例化后直接就是 DOM 节点。升级过程是 React 中没有的，因为 React 组件总是在 JS 内存中先创建，然后一次性渲染到 DOM，不存在“占位符”状态。

### 3. 基础代码与实战验证
```text
// 1. 定义一个自定义元素类
class MyElement extends HTMLElement {
  // 声明需要监听的属性
  static observedAttributes = ['color'];

  constructor() {
    super(); // 必须首先调用 super()，以正确设置原型链和初始化 HTMLElement
    console.log('1. Constructor: 元素实例被创建。this 指向新实例。');
    // 此时 this.attributes 为空，this.children 为空
    this.attachShadow({ mode: 'open' }); // 创建 Shadow DOM
  }

  connectedCallback() {
    console.log('2. ConnectedCallback: 元素被插入到 DOM。');
    // 这是读取属性、操作子元素、渲染内容的最佳时机
    this.render();
  }

  disconnectedCallback() {
    console.log('4. DisconnectedCallback: 元素从 DOM 中移除。');
    // 在这里进行清理工作
  }

  attributeChangedCallback(name, oldValue, newValue) {
    console.log(`3. AttributeChangedCallback: 属性 ${name} 从 '${oldValue}' 变为 '${newValue}'。`);
    // 注意：首次设置属性时，oldValue 为 null
    this.render();
  }

  render() {
    const color = this.getAttribute('color') || 'black';
    this.shadowRoot.innerHTML = `<p style="color: ${color};">Hello, Web Components!</p>`;
  }
}

// 2. 注册自定义元素
customElements.define('my-element', MyElement);

// 3. 实战验证：
const container = document.getElementById('app');

// 验证升级过程：
// 步骤 A: 在 JS 中先创建一个占位元素
const placeholder = document.createElement('my-element');
placeholder.setAttribute('color', 'red');
// 此时 placeholder 是一个普通的 HTMLElement，不是 MyElement 的实例
console.log('Is placeholder an instance of MyElement?', placeholder instanceof MyElement); // false

// 步骤 B: 调用 define 之后，这个占位元素会被“升级”
// 浏览器会创建一个新的 MyElement 实例，并替换掉 placeholder
customElements.upgrade(placeholder); // 手动触发升级，或在 define 时自动触发
console.log('Is placeholder an instance of MyElement?', placeholder instanceof MyElement); // true

// 将升级后的元素添加到 DOM
container.appendChild(placeholder); // 触发 connectedCallback 和 attributeChangedCallback

// 动态修改属性
placeholder.setAttribute('color', 'blue'); // 再次触发 attributeChangedCallback

// 从 DOM 中移除
setTimeout(() => {
  container.removeChild(placeholder); // 触发 disconnectedCallback
}, 2000);
```

### 4. 常见误区与进阶思考
**常见误区**:

1. **在 `constructor` 中进行 DOM 操作**: 专业工程师最常犯的错误是在 `constructor` 中尝试访问或修改子元素、读取属性。在 `constructor` 执行时，元素的 HTML 结构尚未被解析，属性和子节点都不可用。这会导致不可预测的行为或错误。正确的做法是将所有依赖于 DOM 结构的初始化逻辑推迟到 `connectedCallback` 中执行。

2. **忘记 `attributeChangedCallback` 的初始调用**: 当一个带有 `observedAttributes` 中声明的属性的元素被升级时，`attributeChangedCallback` 会在 `connectedCallback` 之前被调用一次（此时 `oldValue` 为 `null`）。如果渲染逻辑只放在 `connectedCallback` 中，那么初始属性值将不会被用于第一次渲染。必须在 `attributeChangedCallback` 中也处理渲染逻辑，或者确保 `connectedCallback` 中的渲染能正确读取初始属性。

**进阶思考题**:

描述一下 `customElements.define('x-foo', ...)` 执行期间，浏览器引擎内部处理已存在的 `<x-foo>` 元素的精确步骤。为什么说这个“升级”过程是同步的，以及这种同步性对依赖 `<x-foo>` 元素在 `define` 之前存在的脚本逻辑有何潜在影响？（提示：考虑一个在 `define` 之前通过 `document.querySelector('x-foo')` 获取了元素引用，并计划在 `define` 之后立即调用该引用方法的脚本。）
