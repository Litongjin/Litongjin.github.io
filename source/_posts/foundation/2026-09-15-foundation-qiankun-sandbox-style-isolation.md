---
title: "每日基础技术总结 · 2026-09-15 · 微前端：qiankun 的沙箱与样式隔离"
date: 2026-09-15 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端框架层（Vue / React / 工程化）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-15 · 微前端：qiankun 的沙箱与样式隔离

## 📚 今日主题

> **微前端：qiankun 的沙箱与样式隔离**（前端框架层（Vue / React / 工程化））

### 1. 核心概念速览
qiankun 的沙箱是对子应用全局副作用（window 属性、全局函数、定时器等）的运行时隔离机制；样式隔离是对子应用注入 CSS 的作用域约束机制。本质不是应用层模块封装，而是在浏览器运行时层对 JS 全局对象访问和 CSSOM 注入做代理/改写，目标是让多个独立构建、独立部署的前端应用在同一个页面聚合时互不污染、可独立升级。JS 沙箱核心用 `with + Proxy` 构造一个虚拟的全局查找链；样式隔离核心用 Shadow DOM 或选择器加作用域前缀的方式，让子应用样式只命中自己的挂载子树。它在计算机体系中的定位接近‘用户态运行时沙箱’：比 iframe 的浏览器级隔离轻，比重写应用代码的编译器隔离强。专业工程师必须掌握，因为微前端的稳定性、性能和排障完全依赖对这两个机制的精确理解；理解它也就理解了浏览器全局对象、作用域链、CSSOM 和运行时副作用管理的边界。

### 2. 底层原理剖析
JS 沙箱的本质是改变子应用代码的标识符解析终点。正常脚本顶层声明的变量和普通赋值最终落到 `window`；沙箱让这些读写先落在一个由 Proxy 创建的 `fakeWindow` 上，避免污染 `rawWindow`。

核心工作流如下：

1. 创建空对象 `fakeWindow`，绑定 Proxy，拦截 `get`、`set`、`has`。
2. `get` 中如果属性在 `fakeWindow` 上则返回；否则返回 `rawWindow[prop]`；对 `window`、`self`、`globalThis` 返回 proxy 自身。
3. `set` 记录新增属性和被修改属性的原始 descriptor，然后写入 `fakeWindow`。
4. 通过 `with(proxy)` 执行子应用代码，让未声明标识符的读写在作用域链上先命中 proxy。
5. 挂载时激活，卸载时用保存的 descriptor 恢复 `rawWindow`，并删除 `fakeWindow` 上新增的属性。

伪代码：

    const fakeWindow = Object.create(null);
    const added = new Set();
    const original = new Map();
    const proxy = new Proxy(fakeWindow, {
      has(target, key) {
        return key in target || key in rawWindow;
      },
      get(target, key) {
        if (key === 'window' || key === 'self' || key === 'globalThis') return proxy;
        return key in target ? target[key] : rawWindow[key];
      },
      set(target, key, value) {
        if (!(key in rawWindow)) added.add(key);
        else if (!original.has(key)) {
          original.set(key, Object.getOwnPropertyDescriptor(rawWindow, key));
        }
        target[key] = value;
        return true;
      }
    });

    function run(proxy, source) {
      // with 使未声明标识符的解析先经过 proxy；需要非严格模式
      return eval('with(proxy) {' + source + '}');
    }

    function restore() {
      added.forEach((key) => delete fakeWindow[key]);
      original.forEach((descriptor, key) => Object.defineProperty(rawWindow, key, descriptor));
    }

关键点是 `has` 陷阱：`with` 环境在作用域链上查找标识符时不会直接调用 `get`，而是先调用 `has` 判断该对象是否拥有该属性；只有 `has` 返回 true，后续 `get` 才会生效。因此 `window`、`self`、`globalThis` 必须在 `has` 里对 rawWindow 为真，才能被重定向到 proxy。

样式隔离有两种实现：

- `strictStyleIsolation`：把子应用容器挂到一个 `shadowRoot` 下。浏览器原生将 CSS 边界限制在 shadow tree 内，子应用样式不会外泄，外部全局样式也不能直接打到 shadow DOM 内部。
- `experimentalStyleIsolation`：在动态解析子应用 `<style>`/`<link>` 得到 CSSOM 后，对所有规则的选择器重写，加上容器作用域前缀（例如原始 `div` 变为 `[data-qiankun='app'] div`，或按内部 CSSProcessor 的实现改写），然后放回 style 标签。对运行期 `document.head.appendChild(style)` 插入的动态 CSS，qiankun 通过 patch 或机制捕获后同样处理。

与前端已有概念的异同：

- 与 iframe 相比：iframe 是浏览器级硬隔离，天然隔离 window/document/CSS，但无法融入主应用 DOM 树；qiankun 是运行时级软隔离，保持单 DOM 树，但隔离强度取决于 Proxy/patch 覆盖范围。
- 与 CSS Modules/Vue scoped 相比：CSS Modules 是构建期静态改写选择器，qiankun 的样式隔离是运行期针对任意动态样式文本的 CSSOM 改写，不要求子应用用特定工具链。
- 与 Java 接口和 TS 接口的差异类似：同名概念（interface、沙箱、隔离）在不同层级的语义可能完全不同，不能只看 API 名；qiankun 的沙箱不是虚拟机，它只隔离了从作用域链能触达的属性，不隔离宿主环境。

注意：`with + eval` 只能在非严格模式运行。模块化产物默认 strict mode，因此 qiankun 对 script 的处理是转成普通脚本或降级执行，这也是沙箱机制的边界之一。

### 3. 基础代码与实战验证
```text
以下代码可直接在浏览器控制台或非严格模式脚本中运行，验证 Proxy 沙箱和 CSSOM 选择器改写两个核心机制。

    // 1. Proxy 沙箱：拦截全局赋值，并支持恢复
    function createSandbox(rawWindow) {
      const fakeWindow = Object.create(null); // 子应用的“虚拟全局对象”
      const addedKeys = new Set();            // 记录子应用新增的全局属性
      const originalDescriptors = new Map();  // 记录被修改属性的原始描述符

      const proxy = new Proxy(fakeWindow, {
        has(target, key) {
          // 让 with 环境认为 proxy 拥有这些属性
          return key in target || key in rawWindow;
        },
        get(target, key) {
          // 把 window/self/globalThis 重定向到 proxy，防止用 window.xxx 绕过沙箱
          if (key === 'window' || key === 'self' || key === 'globalThis') return proxy;
          // 子应用自己写入的属性优先；未写入的再回落到真实 window
          return key in target ? target[key] : rawWindow[key];
        },
        set(target, key, value) {
          // 首次写入时记录新增属性
          if (!(key in rawWindow)) addedKeys.add(key);
          // 首次修改 rawWindow 上已有属性时，保存原始描述符
          else if (!originalDescriptors.has(key)) {
            originalDescriptors.set(key, Object.getOwnPropertyDescriptor(rawWindow, key));
          }
          target[key] = value; // 写入 fakeWindow，而不是 rawWindow
          return true;
        }
      });

      function run(source) {
        // with 让 source 里的未声明标识符先查 proxy；源码本身不能是 strict mode
        return eval('with(proxy) {' + source + '}');
      }

      function restore() {
        addedKeys.forEach((key) => { delete fakeWindow[key]; });
        originalDescriptors.forEach((descriptor, key) => {
          Object.defineProperty(rawWindow, key, descriptor);
        });
      }

      return { run, restore };
    }

    const sandbox = createSandbox(window);
    sandbox.run('sandboxVar = 42; window.sandboxWindowVar = 43;');
    console.log(window.sandboxVar);       // undefined，因为写入的是 fakeWindow
    console.log(window.sandboxWindowVar); // undefined，因为 window 被重定向到 proxy
    console.log(sandbox.run('sandboxVar')); // 42，proxy 内可以读到
    sandbox.restore();

    // 2. 极简样式作用域：把普通 CSS 改成只匹配挂载子树
    function scopeCss(cssText, scopeSelector) {
      const style = document.createElement('style');
      style.textContent = cssText;
      document.head.appendChild(style); // 让浏览器解析 CSSOM
      const sheet = style.sheet;
      const rules = Array.from(sheet.cssRules);
      let result = '';
      for (const rule of rules) {
        if (rule.selectorText) {
          // 给每一条规则的选择器前追加 scopeSelector，形成子树后代选择器
          const scoped = rule.selectorText
            .split(',')
            .map((sel) => `${scopeSelector} ${sel.trim()}`)
            .join(',');
          result += `${scoped} { ${rule.style.cssText} }\n`;
        }
      }
      document.head.removeChild(style);
      return result;
    }

    console.log(scopeCss('div { color: red } .btn { padding: 4px }', '[data-qiankun="app"]'));
    // 输出：
    // [data-qiankun="app"] div { color: red }
    // [data-qiankun="app"] .btn { padding: 4px }
```

### 4. 常见误区与进阶思考
误区一：以为 qiankun 沙箱是“虚拟机级物理隔离”。实际上它只是对 `with(proxy)` 作用域链上可访问的全局属性读写做了拦截，并且依赖 `has` 陷阱和 `window/self/globalThis` 的重定向。只要子应用代码能够拿到真实宿主对象（例如通过底层逃逸路径、未被 patch 的原生方法、或第三方插件绕过执行环境），沙箱就存在被绕过的可能；qiankun 也从未承诺隔离 `history`、`localStorage`、`document` 副作用。正确理解是“尽力而为的全局对象代理”，不是安全边界。

误区二：以为样式隔离能处理所有 CSS 副作用。实验样式隔离改写的是选择器，但无法隔离 `:root` 上的 CSS 变量、`@font-face` 的全局字体注册、`@keyframes` 的全局动画名，以及通过 JS API `CSSStyleSheet.insertRule` 动态插入的样式；strictStyleIsolation 的 Shadow DOM 能让大多数选择器失效，但 `position: fixed`、弹层、拖拽层以及挂载到 `document.body` 的节点仍可能以真实 DOM 树或屏幕为参照逃出 shadowRoot。样式隔离只能控制静态作用域，不能完全控制层叠上下文和渲染目标的逃逸。

思考题：如果子应用代码执行 `window.eval('var hack = 1')`，为什么 qiankun 的 Proxy 沙箱无法阻止 `hack` 写入真实 `window`？这个漏洞暴露了 Proxy 沙箱在“成员访问”和“作用域解析”上的哪个根本边界？
