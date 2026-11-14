---
title: "工作台日报 · 2026-11-15"
date: 2026-11-15 07:01:22
categories: [工作日记]
tags: ["日报", "大模型", "编程语言", "网络安全", "AI工具"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-15

## 🔥 行业热点

- [Reverse engineering my Coway air purifier to ditch the cloud](https://scottjg.com/posts/2026-11-11-reverse-engineering-my-coway-air-purifier-to-ditch-the-cloud/) — *Hacker News*
  - 📌 **内容**：通过逆向空气净化器控制链路来摆脱厂商云，体现 IoT 设备本地化和隐私自主化问题。
  - 💡 **学习**：可学习设备通信抓包、协议逆向和本地网关设计。
  - 🧭 **拓展**：可用局域网抓包和协议模拟验证设备是否能脱离云端控制。
- [Lovable apps expose passport details and more than 200K private messages](https://www.crissyfield.de/research/lovable-en/) — *Hacker News*
  - 📌 **内容**：快速构建类应用被曝泄露敏感身份信息，提示低门槛开发产品容易引入数据暴露风险。
  - 💡 **学习**：可学习权限最小化、数据隔离、接口鉴权和误配置审计。
  - 🧭 **拓展**：可用 OWASP Top 10 对类似 SaaS 应用做访问控制复测。
- [Generative AI uses more energy on non-English text](https://news.umich.edu/generative-ai-uses-more-energy-on-non-english-text/) — *Hacker News*
  - 📌 **内容**：研究指出生成式 AI 处理非英语文本可能消耗更多能量，说明语言差异会影响模型成本。
  - 💡 **学习**：可学习在多语言评估中纳入 tokenization、长度和计算开销。
  - 🧭 **拓展**：可在自有多语言样本上测量吞吐、显存和能耗变化。
- [Show HN: I have trained 15 blood glucose forecasting models on my own data](https://github.com/0xdeadf1sh/T1DMAI) — *Hacker News*
  - 📌 **内容**：用个人血糖数据训练多种预测模型，展示时序预测和小规模机器学习实验思路。
  - 💡 **学习**：可学习特征工程、模型比较、交叉验证和避免过拟合。
  - 🧭 **拓展**：可用时间切分评估不同模型在预测误差上的表现。
- [Show HN: Agweb – a browser for agents that stays out of your way](https://github.com/paul-nameless/agweb) — *Hacker News*
  - 📌 **内容**：为 AI Agent 设计的低干扰浏览器，关注 Agent 如何更高效地访问和操作用户界面。
  - 💡 **学习**：可学习 Agent 的网页解析、动作执行和人机交互边界设计。
  - 🧭 **拓展**：可让 Agent 在测试站点中完成抓取、点击和表单填写流程。
- [Terminal borders that don't cost a cell](https://www.joshka.net/cell-borders/) — *Hacker News*
  - 📌 **内容**：终端边框绘制优化关注不占用额外字符单元，体现 TUI 布局与渲染细节。
  - 💡 **学习**：可学习终端单元格宽度、Unicode 边框和界面重绘技巧。
  - 🧭 **拓展**：可在不同终端模拟器中验证字符宽度和渲染差异。
- [Ggml v1.0.0](https://github.com/ggml-org/ggml/releases/tag/v1.0.0) — *Hacker News*
  - 📌 **内容**：GGML 发布稳定版本，作为轻量模型计算库适合本地 AI 推理与集成。
  - 💡 **学习**：可学习张量计算抽象、模型格式和后端适配思路。
  - 🧭 **拓展**：可用小模型测试不同后端的吞吐与资源占用。
- [PyPI's redesigned project pages with clearer security signals are now live](https://blog.pypi.org/posts/2026-11-06-ui-updates-complete/) — *Hacker News*
  - 📌 **内容**：PyPI 项目页强化安全信号，帮助开发者更快判断依赖包风险。
  - 💡 **学习**：可学习包元数据、安全通告和版本健康度审查方法。
  - 🧭 **拓展**：可抓取 PyPI 包页安全字段并接入 CI 做依赖审计。
- [Show HN: DeepSeek V3 in ~200 lines of pure Jax](https://github.com/Vaibhaav-Tiwari/deepseek-v3-jax) — *Hacker News*
  - 📌 **内容**：用约两百行纯 JAX 复现 DeepSeek V3 相关实现，展示模型代码的最小化思路。
  - 💡 **学习**：可学习 JAX 的函数式编程、向量化和模型结构简化。
  - 🧭 **拓展**：可在短序列输入上验证核心算子输出是否合理。
- [Formalizing US Tax Code (Using Lean)](https://lexifina.com/blog/formalizing-the-us-tax-code) — *Hacker News*
  - 📌 **内容**：用 Lean 形式化美国税法，体现复杂规则系统如何转化为可验证逻辑模型。
  - 💡 **学习**：可学习形式化规范、定理证明和规则建模方法。
  - 🧭 **拓展**：可把简单税务规则改写成形式化命题并验证样例。

## 🌟 GitHub 热门开源项目

- [thedotmack/claude-mem（Agent Skills）](https://github.com/thedotmack/claude-mem) — *GitHub · TypeScript · +16.4k/周 · 总 117.1k star*
  - 📌 **是什么**：TypeScript 实现的 agent 记忆层，能把会话过程压缩并在后续 session 中注入相关上下文。
  - 💡 **学习点**：适合学习 agent 长期记忆、上下文压缩与状态保持的设计方式。
  - 🧭 **上手**：从 README 的安装/使用示例入手，跑一次多轮 session 并观察记忆写入与召回流程。
- [paperclipai/paperclip（LLM 应用）](https://github.com/paperclipai/paperclip) — *GitHub · TypeScript · +11.8k/周 · 总 113.9k star*
  - 📌 **是什么**：面向工作场景的开源 agent 管理应用，围绕 agent orchestration、skills 与治理提供控制台。
  - 💡 **学习点**：适合前端工程师学习如何把 agent 能力做成可运营的产品界面。
  - 🧭 **上手**：阅读项目结构中的 dashboard/agent orchestration 页面，理解它如何展示 agent 任务、权限和状态。
- [firecrawl/firecrawl（开发工具）](https://github.com/firecrawl/firecrawl) — *GitHub · TypeScript · +10.2k/周 · 总 201.8k star*
  - 📌 **是什么**：面向 AI agent 的网页抓取与数据抽取工具，把网页内容转换为适合 LLM 使用的 markdown 或结构化数据。
  - 💡 **学习点**：学习如何给 agent 接入外部网页上下文与数据检索能力。
  - 🧭 **上手**：先看 examples 中 scrape/crawl API 的输入输出，再用本地最小调用测试页面转 markdown。
- [herdrdev/herdr（Agent 框架）](https://github.com/herdrdev/herdr) — *GitHub · Rust · +6.9k/周 · 总 51.0k star*
  - 📌 **是什么**：面向 coding agents 的终端运行时/编排层，用于承载和调度多个代码 agent。
  - 💡 **学习点**：适合研究 agent 如何在终端、会话和子进程里被编排与隔离。
  - 🧭 **上手**：对比 README 中的 CLI 启动流程与 agent multiplexing 机制，跑一个最小 demo 看 runtime 如何接管编码 agent。
- [browser-use/browser-use（Agent 框架）](https://github.com/browser-use/browser-use) — *GitHub · Python · +3.0k/周 · 总 121.0k star*
  - 📌 **是什么**：用于构建 AI browser agent 的 Python 库，可通过浏览器自动化执行网页操作，并提供云端/托管方案。
  - 💡 **学习点**：学习 browser agent 如何将自然语言目标转成网页动作与状态反馈。
  - 🧭 **上手**：从 quickstart 示例跑一个登录或表单任务，观察 DOM 到动作的链路。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue 双向同步防死循环
- **技能点**：掌握框架状态下派生集合与双向绑定的同步守卫能力，避免响应式循环。
- **坑点**：折叠标志触发新数组赋值，deep watch 又回写标志；即使逻辑值不变，引用变化仍会持续唤醒 watcher。
- **解决方案**：在每次赋值前做值比较，只有推导结果不同才写状态；优先把双向同步改成单向状态源。
```text
watch([isFold, isAi], () => {
  const next = [isFold.value ? 0 : 300, 'auto', isAi.value ? 400 : 0]
  if (next.join('|') === sizes.value.join('|')) return
  sizes.value = next
})
watch(sizes, (v) => {
  const fold = v[0] === 0
  if (fold !== isFold.value) isFold.value = fold
}, { deep: true })
```
- **拓展**：可扩展为通用双向同步工具函数，统一处理数组/对象引用造成的无意义触发。
- *来源：admin-workspace-new | 2026-08-11*

### 2. 事件冒泡与组件边界
- **技能点**：掌握从 DOM/组件事件链路定位误触发根因，并区分共享组件与业务组件的职责边界。
- **坑点**：内嵌复制按钮点击冒泡到外层展开/收起处理器；如果直接把 stopPropagation 写进共享组件，会污染通用组件语义。
- **解决方案**：在业务调用处显式 @click.stop，共享组件保持无副作用；必要时通过 emits 或 props 暴露事件控制契约。
```text
<CopyText :text='questionNo' :copyText='id' @click.stop />
```
- **拓展**：把常见嵌套交互抽象成事件冒泡契约表，减少同类业务组件返工。
- *来源：admin-workspace-new | 2026-08-11*

### 3. 组件插槽透传定制
- **技能点**：掌握通过插槽透传扩展第三方/基础组件的可视区，同时保留组件原生交互与状态同步。
- **坑点**：自定义折叠图标若直接覆盖全局样式或替换内部点击逻辑，容易破坏组件原生折叠状态。
- **解决方案**：基础组件新增具名插槽并按 $slots 透传默认图标，业务侧仅替换图标，折叠动作仍由原组件处理。
```text
<div class='collapsible' @click='collapse'>
  <slot name='start-collapsible'><i class='default-collapse-icon' /></slot>
</div>
<template #panel-2-start-collapsible>
  <SvgIcon name='arrow-left' />
</template>
```
- **拓展**：沉淀为通用插槽透传清单，后续自定义图标、按钮、提示均优先走 slot 而不是 fork 组件。
- *来源：admin-workspace-new | 2026-08-10*

### 4. 编辑器工具栏配置化
- **技能点**：掌握将富文本编辑器能力沉淀为声明式配置、常量枚举和内置 handler 的抽象能力。
- **坑点**：业务侧不知道可用按钮名，默认能力必须依赖 customButtons 才能启用，导致配置入口不统一且类型不可发现。
- **解决方案**：补全默认按钮映射，无条件注册内置 handler，并导出 TOOLBAR_BUTTONS 常量与联合类型供业务声明。
```text
export const TOOLBAR_BUTTONS = {
  undo: 'undo',
  redo: 'redo',
  clean: 'clean',
  formula: 'formula',
  fillInInsert: 'fillInInsert',
  image: 'image'
} as const

type ToolbarButtonName = keyof typeof TOOLBAR_BUTTONS

editor.getModule('toolbar').addHandler(TOOLBAR_BUTTONS.formula, () => {
  insertFormula(editor)
})
```
- **拓展**：可延伸到编辑器预设模板、功能开关和权限裁剪，形成业务可复用的低配编辑器能力注册体系。
- *来源：admin-workspace-new | 2026-08-11*

### 5. React 异步 effect 治理
- **技能点**：掌握用异步 effect、重挂载和初始值同步替代不规范 setState，收敛 hooks 相关 lint 问题。
- **坑点**：在普通 useEffect 里 fetch 后 setState 或弹窗打开时 reset 表单，会触发 set-state-in-effect、依赖不完整或状态残留。
- **解决方案**：初始化类 effect 使用 useAsyncEffect；弹窗复位优先调用方 key 重挂载并同步 useState/Form initial 值；复杂依赖先解构成局部变量。
```text
<Modal key={record.id} initialRecord={record} />

function Modal({ initialRecord }) {
  const [values] = useState(initialRecord || {})
  return <Form initialValues={values} />
}
```
- **拓展**：可沉淀 React hooks 治理规则集，并作为代码评审 checklist，确保新增代码不复活同类 lint 问题。
- *来源：admin-workspace-hr-talent | 2026-10-19*

### 6. 排查部署分支旧代码
- **技能点**：掌握线上现象与本地代码不一致时的版本定位方法，先确认部署基线再进入逻辑排查。
- **坑点**：本地已修复或迁移的功能仍在线上报错，根因可能是测试/生产环境跑的是未提交工作区以外的 master 旧代码。
- **解决方案**：先用 git show origin/master 或部署分支查看对应文件内容并 diff，确认功能是否真实存在，再排查请求、接口或业务逻辑。
```text
git show origin/master:src/service/upload.ts > /tmp/master.ts
diff -u /tmp/master.ts src/service/upload.ts
```
- **拓展**：可延伸为发布/副本一致性校验脚本，在联调前自动比对部署分支与本地分支关键文件差异。
- *来源：admin-workspace-hr | 2026-09-24*

