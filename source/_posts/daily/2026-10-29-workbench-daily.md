---
title: "工作台日报 · 2026-10-29"
date: 2026-10-29 07:06:06
categories: [工作日记]
tags: ["日报", "大模型", "AI Agent", "开发工具", "AI"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-10-29

## 🔥 行业热点

- [Dayjob: Tax – evaluating AI agents on 50 professional tax assignments](https://surgehq.ai/blog/dayjob-tax) — *Hacker News*
  - 📌 **内容**：该条目用 50 个专业税务任务评测 AI Agent，关注智能体在真实专业场景中的可靠性与任务完成质量。
  - 💡 **学习**：可学习如何为垂直领域设计 Agent 评测集、拆解任务并衡量工具调用与推理效果。
  - 🧭 **拓展**：可参考其思路自建小型行业基准，比较不同 Agent 框架与提示策略。
- [Survival LLM: AI assistant and emergency library that runs on your own computer](https://www.brangus.io/survival-llm) — *Hacker News*
  - 📌 **内容**：介绍可在本地运行的生存类 LLM 助手与应急资料库，强调离线可用和个人自托管。
  - 💡 **学习**：可了解本地大模型部署、离线知识库与低资源环境下的推理优化。
  - 🧭 **拓展**：可在本地用小型模型+RAG 搭建离线问答原型，验证断网可用性。
- [The Agent SDK](https://cursor.com/blog/agent-sdk) — *Hacker News*
  - 📌 **内容**：Agent SDK 指向为开发者提供构建 AI Agent 的软件开发套件，通常封装工具调用、编排与运行时能力。
  - 💡 **学习**：可学习 Agent SDK 的抽象方式，如工具注册、会话状态、权限与多步任务编排。
  - 🧭 **拓展**：可对比现有 Agent 框架并用同一任务测试其扩展性与可观测性。
- [Sleep framework: A new way for LLMs to retain knowledge in continual learning](https://research.google/blog/the-sleep-framework-a-new-way-for-llms-to-retain-knowledge-in-continual-learning/) — *Hacker News*
  - 📌 **内容**：Sleep framework 关注 LLM 在持续学习中保留已学知识，缓解新任务学习导致的能力遗忘。
  - 💡 **学习**：可了解持续学习、知识保留与灾难性遗忘的评估和训练思路。
  - 🧭 **拓展**：可用小模型在序列任务上复现持续学习实验，对比微调与知识保留策略。
- [Decompiling Games with AI: Virtueel Zoetermeer](https://bou.ke/blog/zoetermeer/) — *Hacker News*
  - 📌 **内容**：该条目讨论用 AI 辅助反编译游戏，探索把模型用于二进制分析与逆向工程。
  - 💡 **学习**：可学习 AI 辅助逆向的流程、静态/动态分析结合以及结果验证方法。
  - 🧭 **拓展**：可在合法授权的旧软件或自制程序上尝试 AI 辅助反编译并人工校验。
- [Ghostty 1.4](https://ghostty.org/docs/install/release-notes/1-4-0) — *Hacker News*
  - 📌 **内容**：Ghostty 1.4 是终端模拟器的新版本发布，面向开发者日常命令行使用。
  - 💡 **学习**：可关注现代终端的渲染性能、配置体验与跨平台实现。
  - 🧭 **拓展**：可安装试用并对比现有终端在启动速度、字体渲染和分屏工作流上的差异。
- [Apple's git still trusts the compromised eMudhra certificates](https://emudhra.we-have-a-problem.com/apple.html) — *Hacker News*
  - 📌 **内容**：该条目指出 Apple 的 git 仍信任已遭入侵的 eMudhra 证书，涉及软件供应链与证书信任问题。
  - 💡 **学习**：可学习证书链校验、CA 信任配置与供应链安全排查。
  - 🧭 **拓展**：可检查本地/CI 环境的信任库与证书吊销配置，验证风险。
- [Gemini 4 Carbon](https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-carbon/) — *Hacker News*
  - 📌 **内容**：该条目讨论 Gemini 4 Carbon，指向 Gemini 相关的新一代 AI 能力或工具发布。
  - 💡 **学习**：可关注新 AI 模型/工具的能力边界、调用方式与评测方法。
  - 🧭 **拓展**：可用公开接口做小规模对比测试，验证其在代码或文本任务上的表现。
- [Why is this fingerprinting script creating bioweapons?](https://nullpt.rs/fingerprinting-bioweapons) — *Hacker News*
  - 📌 **内容**：该条目讨论浏览器指纹识别脚本引发的安全与伦理疑问，涉及前端追踪脚本审计。
  - 💡 **学习**：可学习浏览器指纹识别机制、脚本行为审计与隐私风险识别。
  - 🧭 **拓展**：可在隔离环境抓包复现，检查其数据收集与网络请求。
- [MacBook Pro with PROLED XDR Display and Touch](https://www.apple.com/macbook-pro/proled-xdr-display/) — *Hacker News*
  - 📌 **内容**：该条目介绍带 PROLED XDR 显示与触控的 MacBook Pro，属于笔记本硬件与显示技术方向。
  - 💡 **学习**：可关注高端显示、触控交互与移动工作站硬件趋势对开发适配的影响。
  - 🧭 **拓展**：可查阅官方规格与开发者文档，评估显示/触控变化对应用界面的适配。

## 🌟 GitHub 热门开源项目

- [firecrawl/firecrawl（MCP 工具）](https://github.com/firecrawl/firecrawl) — *GitHub · TypeScript · +10.9k/周 · 总 196.4k star*
  - 📌 **是什么**：面向 AI agent 的网页抓取与数据提取库，把网页内容清洗成 Markdown 或结构化数据供 LLM 消费。
  - 💡 **学习点**：学它如何把抓取、清洗、转 Markdown 这串流程封装成 agent 可直接调用的工具接口，这是给 LLM 接外部数据的典型范式。
  - 🧭 **上手**：先跑通 README 里的 scrape/crawl 最小示例，再读 HTML→Markdown 转换那一层代码，对比自己手写的解析逻辑。
- [blader/humanizer（Agent Skills）](https://github.com/blader/humanizer) — *GitHub · Python · +9.6k/周 · 总 61.9k star*
  - 📌 **是什么**：一个 agent skill，用规则与提示词集合去掉文本里的 AI 写作痕迹，让输出更像真人写的。
  - 💡 **学习点**：学它如何把一套写作风格规则组织成可复用的技能说明与提示词，这是 prompt 工程产品化的最小单元。
  - 🧭 **上手**：通读技能定义文件里的规则清单，再拿一段模型生成的文案跑一遍，对比处理前后的差别。
- [pbakaus/impeccable（Agent Skills）](https://github.com/pbakaus/impeccable) — *GitHub · Rust · 周增量统计中 · 总 92.5k star*
  - 📌 **是什么**：给编码 agent 的一套设计词汇技能，用 polish、bolder、critique、audit 这类指令驱动设计打磨与审查。
  - 💡 **学习点**：学它把设计评审经验拆成 agent 可调用的短指令和检查清单，前端的设计判断力可以这样被工程化。
  - 🧭 **上手**：读技能里的指令定义与 critique/audit 检查项，然后挑自己一个前端页面让 agent 跑一次 audit。
- [thedotmack/claude-mem（Agent Skills）](https://github.com/thedotmack/claude-mem) — *GitHub · TypeScript · +9.2k/周 · 总 104.0k star*
  - 📌 **是什么**：给 agent 做跨会话持久记忆的工具：记录会话过程、用 AI 压缩，再把相关上下文注入后续会话。
  - 💡 **学习点**：学“记录—压缩—检索—注入”这条上下文管理流水线，这是长会话 agent 应用的核心工程问题。
  - 🧭 **上手**：先看压缩与注入两个环节的实现与配置，再改一次压缩用的提示词，观察注入内容如何变化。
- [paperclipai/paperclip（Agent 框架）](https://github.com/paperclipai/paperclip) — *GitHub · TypeScript · +7.3k/周 · 总 108.3k star*
  - 📌 **是什么**：开源的“工作中管理 agent”应用，围绕多 agent 编排、协作与治理做产品化封装。
  - 💡 **学习点**：学它如何设计 agent 编排层与权限治理抽象，理解把 agent 从脚本变成团队工作流产品的关键设计。
  - 🧭 **上手**：按 README 快速启动一个示例工作流，重点看编排配置的数据结构长什么样。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步防死循环
- **技能点**：掌握用「值比较守卫」切断状态 A→数组→状态 B→状态 A 的回写环，能定位并修复 Maximum recursive updates 一类的响应式递归。
- **坑点**：两个 watch 互相回写，其中一侧每次赋新数组引用（即使内容相同），deep watch 必被唤醒，标志位最终值未变也照样递归，直到报错。
- **解决方案**：每个回写点先比较再赋值：数组逐项比对相等直接 return；标志位先判 `当前值 !== 推导值` 才写。根治做法是评估是否真的需要双向同步，能删则删。
```text
const next = [leftWidth, 'auto', rightWidth];
if (next[0] === localPanelSizes.value[0] && next[2] === localPanelSizes.value[2]) return; // 无变化不写
localPanelSizes.value = next;

// watch(localPanelSizes) 回写标志前先比较
if (isAiCheckContentFolded.value !== folded) isAiCheckContentFolded.value = folded;
```
- **拓展**：凡「UI 受控状态 ↔ 业务状态」双向绑定的场景（面板尺寸、分页、筛选 URL 同步）都适用同一守卫模式。
- *来源：admin-workspace-new 2026-08-11*

### 2. 事件冒泡修复的层级选择
- **技能点**：判断 bug 该修在共享组件还是业务组件：共享组件保持通用语义，业务侧的交互冲突由业务侧 `@click.stop` 消化。
- **坑点**：复制按钮的点击冒泡到外层容器 `@click="handleQuestionClick"`，误触发解析展开/收起；若直接在通用复制组件里加 stopPropagation，会污染所有调用方。
- **解决方案**：在业务组件调用处加 `@click.stop`，共享组件回退到无副作用实现；同时用 `copyText` prop 分离「显示文案」与「实际复制值」。
```text
<CopyText :text="`题目编号: ${data.questionId}`" :copyText="data.questionId" @click.stop />
```
- **拓展**：沉淀一条判据：组件库只做能力、不做业务决策；冲突一律在调用方收敛。
- *来源：admin-workspace-new 2026-08-11*

### 3. 通用组件用插槽透传扩展
- **技能点**：在不侵入组件内部逻辑的前提下扩展通用组件外观：新增具名插槽透传 + 默认 UI 隐藏，能力仍由组件原生处理。
- **坑点**：为了换折叠图标去改 ResizablePanels 内部实现，会把通用组件绑死业务样式；同时持久化等默认能力容易顺手打开，产生非预期状态源。
- **解决方案**：组件侧加 `v-if="$slots.xxx"` 的条件插槽透传并隐藏官方图标，业务侧在插槽里放 svgIcon + rotate 区分收起/展开；`storageKey` 保持可选，调用方不传即自动跳过持久化。
```text
<!-- 组件 -->
<slot v-if="$slots['start-collapsible']" name="start-collapsible" />
<!-- 业务 -->
<template #panel-0-start-collapsible><svgIcon class="collapse-icon" /></template>
```
- **拓展**：可作为通用组件的标准扩展套路：插槽 > props > 改源码，优先级依次递减。
- *来源：admin-workspace 2026-08-10*

### 4. 注册表常量与联合类型
- **技能点**：把散落的字符串注册项收口为 `as const` 常量数组 + 派生联合类型，让调用方有类型提示、组件内注册逻辑不再依赖额外配置。
- **坑点**：业务方不知道 `:toolbar-order` 能传哪些按钮名，公式/填空必须额外配 `customButtons` 才能用，用法隐晦且易错。
- **解决方案**：导出 `TOOLBAR_BUTTONS` 与 `ToolbarButtonName`，默认按钮全量映射进 `getDefaultButtonConfig` 并无条件注册内置 handler，业务侧统一用 `B.xxx` 常量声明工具栏。
```text
export const TOOLBAR_BUTTONS = ['undo','redo','divider','fillInInsert','formula','image'] as const;
export type ToolbarButtonName = typeof TOOLBAR_BUTTONS[number];
// 调用方：const toolbar = [B.undo, B.bold, B.divider, B.formula];
```
- **拓展**：可复用于任何枚举型配置（字典、状态码、列定义），并配合类型守卫做运行时校验。
- *来源：admin-workspace-new 2026-08-11*

### 5. 去状态化精简
- **技能点**：识别「派生状态」与「必要状态」的边界，删得多余状态源，用 v-if 直接控制显隐，缩短状态链也就消除了一整类同步 bug。
- **坑点**：为记住尺寸/折叠态引入多个 ref + 双向 watch + 持久化，任何一环变化都要回写，既是递归的温床，也让需求方实际要的「拖拽+折叠+最小宽」被淹没。
- **解决方案**：删掉折叠标志、双向同步、storageKey 传参，`localPanelSizes` 只作为纯拖拽/折叠内部状态；右栏显隐改由 `isAiReviewer` + `v-if` 决定，不再用 sizes 里的 `'hidden'` 字符串。
```text
<template v-if="isAiReviewer" #panel-2>...</template>
<!-- 不再：sizes = [760, 'auto', isAiReviewer ? 400 : 'hidden'] -->
```
- **拓展**：需求收敛时优先问「这个状态非有不可吗」，删状态比修同步更便宜。
- *来源：admin-workspace-new 2026-08-11*

### 6. 重构分支合并主干策略
- **技能点**：掌握大改造分支与主干的合并路径选择：直接 merge 后在主干基线上重做语义成果，而不是先镜像格式化再合并。
- **坑点**：我方做过全量 prettier 时，把主干也格式化后再 merge 反而放大冲突（117 文件/414 冲突块 vs 直接 merge 的 51/118）——合并 base 是未格式化版本，两侧行布局错位，还把「仅我方改」变成「双方都改」。
- **解决方案**：直接 `git merge <主干>` → 冲突处整体采纳主干版 → 统一 `npm run format` → 按 grep 清单在主干基线上重做语义改动。
```text
git merge master
# 冲突处整体采纳主干版
npm run format
# 按影响清单 grep 驱动地重做语义成果
```
- **拓展**：同理适用于大规模自动格式化、重命名类改动，切忌把它们和跨分支合并叠在同一层。
- *来源：admin-workspace-hr 2026-10-14*

### 7. 改动落盘与残留校验
- **技能点**：建立「改完即验证」的闭环习惯：确认编辑真的落盘、grep 断言已删符号无残留、lint 只对新增错误负责。
- **坑点**：会话切换后部分 edit 实际未落盘，磁盘仍是旧版；并行编辑同一文件基于同一快照互相覆盖，导致改动看似完成实则丢失或半套。
- **解决方案**：同一文件多处编辑逐条顺序执行；改完 grep 目标符号确认残留为 0，再跑 lint/tsc 区分「新增错误」与「存量基线」。
```text
grep -rn "isFilePreviewFolded" src/   # 期望 0 处残留
npx tsc --noEmit                     # 仅关注新增报错
```
- **拓展**：可固化为提交前检查清单，或用脚本做命中数断言式批量替换。
- *来源：admin-workspace-new 2026-08-11*

