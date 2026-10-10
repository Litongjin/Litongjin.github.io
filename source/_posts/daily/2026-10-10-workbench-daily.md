---
title: "工作台日报 · 2026-10-10"
date: 2026-10-10 15:53:13
categories: [工作日记]
tags: ["日报"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-10-10

## 🔥 行业热点

- （暂无热点）

## 🌟 GitHub 热门开源项目

- [Panniantong/Agent-Reach（Agent Skills）](https://github.com/Panniantong/Agent-Reach) — *GitHub · Python · +15.8k/周 · 总 95.2k star*
  - 📌 **是什么**：为 AI 代理提供跨多个主流平台搜索与读取能力的命令行工具，帮助代理获取网页内容而无需支付官方 API 费用。
  - 💡 **学习点**：可学习如何把多平台数据源封装成统一接口，并将抓取结果整理为适合 LLM 消费的文本。
  - 🧭 **上手**：按 README 的 CLI 示例对同一关键词分别搜索两个平台，对比返回文本的结构和清洗逻辑。
- [Graphify-Labs/graphify（Agent Skills）](https://github.com/Graphify-Labs/graphify) — *GitHub · Python · +8.2k/周 · 总 125.1k star*
  - 📌 **是什么**：把代码库及其文档、SQL schema、配置和 PDF 等资料转换为可查询的知识图谱，并作为 Claude Code、Cursor 等工具的技能使用。
  - 💡 **学习点**：可学习基于 AST 的代码解析、知识图谱构建，以及如何为 LLM 提供更结构化的代码上下文。
  - 🧭 **上手**：在本地小项目上运行 /graphify 生成图谱，然后查询一个函数或数据表关系，观察检索结果。
- [Leonxlnx/taste-skill（Agent Skills）](https://github.com/Leonxlnx/taste-skill) — *GitHub · JavaScript · +8.1k/周 · 总 94.2k star*
  - 📌 **是什么**：一个用于提升 AI 生成内容审美与品质的技能，避免输出乏味、通用的前端或设计结果。
  - 💡 **学习点**：可学习如何通过技能注入设计约束、风格指南和判断标准来约束大模型生成质量。
  - 🧭 **上手**：在 Cursor 或 Claude Code 中启用该技能后，生成同一个 UI 组件并与未启用时对比风格差异。
- [harry0703/MoneyPrinterTurbo（LLM 应用）](https://github.com/harry0703/MoneyPrinterTurbo) — *GitHub · Python · +7.0k/周 · 总 129.4k star*
  - 📌 **是什么**：一个基于大模型和自动化工作流、按主题或关键词一键生成高清短视频的应用。
  - 💡 **学习点**：可学习多模态生成流程如何串联文案、语音、字幕和视频合成等环节。
  - 🧭 **上手**：按 README 快速开始传入一个主题，跑通一次短视频生成并查看中间产物。
- [JuliusBrussee/caveman（Agent Skills）](https://github.com/JuliusBrussee/caveman) — *GitHub · Go · +6.0k/周 · 总 110.8k star*
  - 📌 **是什么**：一种通过极简表达方式降低编码代理 token 消耗的技能与代理工具。
  - 💡 **学习点**：可学习提示词压缩和成本优化思路，以及如何在不显著损失输出质量的前提下减少上下文长度。
  - 🧭 **上手**：对同一个编码任务分别用默认模式和该技能运行，对比 token 用量与生成结果差异。

## 🚀 技能提升点（工作总结汇总）

### 1. 数组配置需展平后再传给 Quill
- **技能点**：掌握适配器层对多按钮配置的展平处理；理解 Quill addControls 只接受一维 controls。
- **坑点**：getDefaultButtonConfig 的 list/indent 返回数组，order.map 保留二维结构，Quill 把数组当对象读到 format='0'，生成 ql-0 空按钮。
- **解决方案**：调用方改用 order.flatMap(...) 展平，数组项各自独立成 {list:'ordered'} 等 control。
```text
const controls = order.flatMap(key => getDefaultButtonConfig(key))
```
- **拓展**：可封装 normalizeControls 统一处理配置返回单按钮或多按钮，避免各调用方遗漏展平。

### 2. Vue watch 双向同步防递归
- **技能点**：掌握 watch 之间互相触发导致 Maximum recursive updates 的根因与值比较守卫。
- **坑点**：状态 A→数组→状态 B→状态 A 的 watch 链中，每次写新数组引用都会触发 deep watch 回写标志，即使值未变也无限循环。
- **解决方案**：在每一步写入前做比较守卫：下一个数组与当前逐项相等则 return；回写派生标志前先比较当前值。
```text
const same = next[0] === sizes.value[0] && next[1] === sizes.value[1] && next[2] === sizes.value[2]
if (same) return
sizes.value = next
```
- **拓展**：同类的状态镜像、localStorage 同步、v-model 双向同步都应在写入前判断值是否实际改变。
- *来源：admin-workspace 2026-08-11*

### 3. 点击冒泡与通用组件复用边界
- **技能点**：掌握事件冒泡在业务组件与共享组件之间的隔离：修复放在调用方而非污染通用组件。
- **坑点**：CopyText 点击冒泡到外层容器 handleQuestionClick，导致复制触发解析展开/收起；若在共享 copyText 内 stopPropagation 会限制通用性。
- **解决方案**：在业务组件使用处加 @click.stop，copyText prop 只复制题目编号；共享组件保持无 event/stopPropagation。
```text
<CopyText :text="题目编号: xxx" :copyText="data.questionId" @click.stop />
```
- **拓展**：类似按钮、下拉、卡片内嵌操作均可通过插槽/事件修饰符在边界处阻断冒泡，保持底层组件纯净。
- *来源：admin-workspace 2026-08-11*

### 4. 富文本工具栏图标“丢失”排查
- **技能点**：区分视觉缺失与 DOM 缺失；先用 devtools 确认真实 DOM 结构，再决定是否改 CSS。
- **坑点**：抽屉宽度有限 + 工具栏 flex-wrap 导致后方 picker 换行被裁出可视区域，主观表现为图标丢失，而非 SVG/DOM 缺失。
- **解决方案**：查 .ql-toolbar 实际 children 数量与 .ql-formats 个数，确认是换行/裁剪问题后再做布局调整；不提前加猜测性 CSS。
- **拓展**：所有 UI 报“元素不见了”时，先用 Elements 面板确认渲染和盒模型/溢出，再定位是样式裁剪、换行还是条件渲染。

### 5. 第三方面板自定义折叠图标插槽透传
- **技能点**：掌握组件库插槽透传与 $slots 检测，隐藏官方图标并提供自定义插槽。
- **坑点**：ResizablePanels 官方折叠图标与业务想要的 svgIcon 不一致；直接改组件会破坏通用性。
- **解决方案**：在 ResizablePanels 增加 #start-collapsible 插槽透传（条件 $slots 检测），隐藏官方图标 display:none；业务侧传 svgIcon。
```text
<template v-if="$slots['start-collapsible']" #start-collapsible><slot name="start-collapsible" /></template>
```
- **拓展**：对 Element/第三方组件做轻度定制时，优先用插槽透传而非内部 DOM/CSS hack，保证可维护性。
- *来源：admin-workspace 2026-08-10*

### 6. 多副本仓库版本一致性检查
- **技能点**：在多 clone/多分支仓库间复用经验和代码前，先确认目标副本的文件结构和版本。
- **坑点**：同项目多个独立 clone 可能停在不同分支/commit，直接套用记忆中的结论会因文件不存在而失效。
- **解决方案**：引用排坑结论前先检查目标副本里对应文件是否存在，必要时 git status/show 确认当前分支与内容。
- **拓展**：可沉淀成 per-repo 版本指纹或快速检查脚本，减少跨副本误判。
- *来源：admin-workspace MEMORY.md*

