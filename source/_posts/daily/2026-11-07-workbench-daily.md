---
title: "工作台日报 · 2026-11-07"
date: 2026-11-07 07:01:31
categories: [工作日记]
tags: ["日报", "大模型", "AI", "AI Agent", "AI工具"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-07

## 🔥 行业热点

- [Claude Code sends your email address to websites](https://github.com/anthropics/claude-code/issues/107757) — *Hacker News*
  - 📌 **内容**：有用户发现 AI 编码工具在访问网页时会携带用户邮箱等身份信息，引发对 AI 代理隐私边界的讨论。
  - 💡 **学习**：使用 AI 编程代理时应关注其出站请求携带的元数据，可用代理或抓包方式审计其网络行为。
  - 🧭 **拓展**：用 mitmproxy 等本地代理抓取代理工具的 HTTP 请求，验证哪些身份信息被外泄。
- [AI is dramatically accelerating AI research](https://www.planned-obsolescence.org/p/ai-is-dramatically-accelerating-ai) — *Hacker News*
  - 📌 **内容**：讨论 AI 正被用来加速自身的算法探索与实验研究，形成研究闭环的趋势。
  - 💡 **学习**：关注 AI 辅助科研的范式：自动文献调研、实验脚本生成与结果迭代。
  - 🧭 **拓展**：在自己的项目中尝试让 agent 承担文献检索与实验脚本生成，对比人工效率。
- [Restaurant Delivery Payment-Skimming Campaign](https://www.infohawk.com/news/2026-11-05-scamwich-restaurant-skimming) — *Hacker News*
  - 📌 **内容**：披露针对外卖点餐支付流程的网页表单劫持式盗刷攻击活动。
  - 💡 **学习**：了解 Magecart 类前端脚本注入窃取支付信息的原理与第三方脚本风险。
  - 🧭 **拓展**：用 CSP、SRI 和第三方脚本清单自查站点支付页面的加载链路。
- [Show HN: Agents infer Lego mechanisms from sources, then animate them](https://github.com/anteloc/ldraw-mecha/tree/v0.5.0) — *Hacker News*
  - 📌 **内容**：一个展示项目：让 Agent 从图纸或说明中推断乐高机械结构并生成动画。
  - 💡 **学习**：可学习多模态 Agent 的“解析图纸→结构化建模→渲染动画”流水线编排思路。
  - 🧭 **拓展**：参照该流程做一个从 CAD 图或结构图到 3D 动画的最小原型。
- [SketchAgentBench – evaluate LLMs on 3D spatial reasoning](https://sketchagent.dev/) — *Hacker News*
  - 📌 **内容**：提出面向 3D 空间与几何推理能力的 LLM 评测基准。
  - 💡 **学习**：了解如何为空间推理类能力设计任务、数据与评测指标。
  - 🧭 **拓展**：借鉴其构造方式，为自己关注的领域搭建小规模定制评测集。
- [Blender's UI and Runtime: Much Faster Than GPUI](https://github.com/ppmpreetham/BUI) — *Hacker News*
  - 📌 **内容**：分析 Blender 的 UI 与运行时架构为何更快，并与 GPUI 做性能对比。
  - 💡 **学习**：可了解立即模式 GUI、渲染循环与事件处理设计对界面性能的影响。
  - 🧭 **拓展**：阅读 Blender UI/GHOST 相关源码，自建微基准复现对比结论。
- [Dutch 'idealist' bank ASN blocks non-Google Android, including GrapheneOS](https://github.com/PrivSec-dev/banking-apps-compat-report/issues/105) — *Hacker News*
  - 📌 **内容**：荷兰银行 App 因设备完整性校验限制非 Google 认证的 Android 系统（含 GrapheneOS）使用。
  - 💡 **学习**：了解 Play Integrity/SafetyNet 类设备认证机制及其与隐私定制 ROM 的冲突。
  - 🧭 **拓展**：研究 GrapheneOS 的兼容方案以及同类金融 App 的检测策略差异。
- [Gemini Omni 1.2 Flash](https://blog.google/innovation-and-ai/technology/developers-tools/introducing-gemini-omni-1-2-flash/) — *Hacker News*
  - 📌 **内容**：Gemini 系列新模型版本发布，方向仍是速度与多模态能力的平衡。
  - 💡 **学习**：关注新模型在延迟、成本与多模态输入上的定位，评估替换现有模型的性价比。
  - 🧭 **拓展**：用自有任务集对新旧模型做 A/B 对比，记录质量、延迟与费用。
- [A Jev that can ask Opus for help gives Opus accuracy at 4x lower cost](https://hightouch.com/blog/llm-judges-identity-resolution) — *Hacker News*
  - 📌 **内容**：通过让小模型在不确定时向大模型求助的级联策略，以显著更低成本逼近大模型精度。
  - 💡 **学习**：学习模型级联/路由与“按需求助”的降本思路及置信度阈值设计。
  - 🧭 **拓展**：在自有问答集上实现阈值触发的级联，绘制成本-精度曲线验证收益。
- [First steps in bare-metal programming using UEFI](https://kevinboone.me/efitest_index.html) — *Hacker News*
  - 📌 **内容**：介绍以 UEFI 为切入点进行裸机编程的入门路径与实践步骤。
  - 💡 **学习**：可了解 UEFI 启动流程、引导加载机制与无操作系统环境下的编程基础。
  - 🧭 **拓展**：用 QEMU 配合 OVMF 固件运行一个最小 UEFI 应用来验证。

## 🌟 GitHub 热门开源项目

- [thedotmack/claude-mem（Agent Skills）](https://github.com/thedotmack/claude-mem) — *GitHub · TypeScript · +16.8k/周 · 总 114.0k star*
  - 📌 **是什么**：一个为编码 Agent 提供跨会话持久记忆的 TypeScript 组件：它会记录 Agent 在会话中的行为，用 AI 压缩成摘要，再把相关上下文注入到之后的会话中。
  - 💡 **学习点**：可以学习如何设计“写入—压缩—检索注入”的记忆闭环，以及如何用向量库（如 Chroma）为 Agent 拼接上下文。
  - 🧭 **上手**：先读它的存储与压缩模块，弄清一次会话被压缩成什么结构，再用本地 Claude 或兼容模型跑一遍“写入会话→新会话召回”的最小流程。
- [calesthio/OpenMontage（Agent 框架）](https://github.com/calesthio/OpenMontage) — *GitHub · Python · +13.0k/周 · 总 77.7k star*
  - 📌 **是什么**：一个开源、以 Agent 驱动的视频制作系统，内置多条制作流水线、大量工具以及成体系的技能与制作知识文件，能把 AI 编码助手变成视频生产助手。
  - 💡 **学习点**：可以观察如何把复杂的多媒体流水线（FFmpeg、Remotion、图像与音频生成）拆成 Agent 可调用的技能与工具。
  - 🧭 **上手**：先看它的 skills/知识文件目录结构和一条最简流水线的编排脚本，再照着跑通一个“文生图→配音→合成”的示例。
- [paperclipai/paperclip（Agent 框架）](https://github.com/paperclipai/paperclip) — *GitHub · TypeScript · +10.7k/周 · 总 111.7k star*
  - 📌 **是什么**：一个基于 TypeScript 的开源工作台，用于在工作场景中管理和编排多个 Agent，覆盖任务分派、治理与协作。
  - 💡 **学习点**：可以学习多 Agent 产品化的产品形态：任务编排、权限与治理、运行状态可视化，这些都是前端工程师容易上手的方向。
  - 🧭 **上手**：先跑起来并查看 Agent 编排配置示例，对比它如何定义一次任务从分派到回收结果的完整链路。
- [firecrawl/firecrawl（MCP 工具）](https://github.com/firecrawl/firecrawl) — *GitHub · TypeScript · +10.0k/周 · 总 199.2k star*
  - 📌 **是什么**：面向 AI 应用的网页数据抓取与检索服务，把网页内容转成适合大模型使用的 Markdown 或结构化数据，为 Agent 提供实时外部信息。
  - 💡 **学习点**：可以学习如何把“抓取—清洗—转 Markdown—切分检索”做成供 LLM 调用的标准工具接口。
  - 🧭 **上手**：先用自托管或云端 API 抓取一个文档站并观察返回的 Markdown 结构与分块结果，再对照它的 MCP 接口说明，把它作为工具接入自己的 Agent。
- [herdrdev/herdr（开发工具）](https://github.com/herdrdev/herdr) — *GitHub · Rust · +6.0k/周 · 总 48.7k star*
  - 📌 **是什么**：用 Rust 编写的编码 Agent 运行时，为 Claude Code、Codex 之类的命令行 Agent 提供多路复用与运行管理的基础设施。
  - 💡 **学习点**：可以理解 Agent 运行时与终端多路复用是如何解耦的，以及如何为长任务维护稳定会话。
  - 🧭 **上手**：读它的 CLI 命令入口与会话管理模块，然后并排启动两个编码 Agent，观察它们的会话隔离与复用效果。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue 双向 watch 死循环
- **技能点**：识别并拆解「状态A→数组→状态B→状态A」的双向同步环路，用值比较校验切断递归触发
- **坑点**：syncSizesFromFlags 每次写入新数组引用，deep watch 回写折叠标志，标志再触发上一个 watch，即使值未变也无限互相唤醒，报 Maximum recursive updates exceeded
- **解决方案**：每个写入点先比较再赋值：算出的 next 与现有值全等则直接 return；回写标志前判断「当前值 !== 推导值」才写
```text
const next = [leftWidth, 'auto', rightWidth]
if (next[0] === sizes.value[0] && next[2] === sizes.value[2]) return
sizes.value = next
if (folded.value !== (next[0] === 0)) folded.value = next[0] === 0
```
- **拓展**：把「派生状态」改为 computed，或只保留单一数据源（尺寸或标志二选一），从结构上消除双向同步
- *来源：admin-workspace-new，2026-08-11*

### 2. 事件冒泡与共享组件边界
- **技能点**：定位「点击子元素却触发父级行为」的冒泡链路，并判断修复应落在业务层还是共享组件层
- **坑点**：footer 复制按钮的 click 冒泡到外层容器的 handleQuestionClick，导致复制时题目卡片被展开或收起；直接改共享组件加 stopPropagation 会污染所有调用方
- **解决方案**：在业务组件调用处加 @click.stop 拦住冒泡，共享 CopyText 保持无 event 参数、不做 stopPropagation 的通用形态
```text
<CopyText :text="`题目编号: ${data.questionId}`" :copyText="data.questionId" @click.stop />
```
- **拓展**：共享组件若要默认阻断冒泡，应做成显式 prop（如 stopClick），而不是写死行为
- *来源：admin-workspace-new，2026-08-11*

### 3. 组件通用能力不被业务裁剪
- **技能点**：区分「组件能力」与「调用方用法」，用空值跳过与插槽透传满足业务定制而不破坏通用性
- **坑点**：业务只想要拖拽+折叠，容易顺手删掉 ResizablePanels 的 storageKey 或改坏共享组件；官方折叠图标样式也不可控
- **解决方案**：通用组件保留能力（空 storageKey 时 saveStorage 自动跳过），调用方不传即可；定制图标通过 $slots 检测透传插槽，仅隐藏官方图标
```text
<slot v-if="$slots['start-collapsible']" name="start-collapsible" />
// 调用方
<template #panel-0-start-collapsible><svgIcon name="arrow" /></template>
```
- **拓展**：把「可选能力 + 插槽透传」沉淀为共享组件扩展范式，避免为一处业务在组件内加 if
- *来源：admin-workspace-new，2026-08-10*

### 4. 工具栏配置化注册与类型导出
- **技能点**：把散落在 customButtons 里的按钮改为配置驱动注册，并导出常量+联合类型收敛调用方入参
- **坑点**：业务方不知道 :toolbar-order 能传哪些名字，formula/image 必须额外配 customButtons 才生效，靠文档口口相传易错
- **解决方案**：默认按钮映射补全并在 buildToolbarHandlers 中无条件注册内置 handler；导出 TOOLBAR_BUTTONS 常量与 ToolbarButtonName 类型，业务用 B.xxx 声明顺序
```text
export const TOOLBAR_BUTTONS = ['undo','redo','divider','bold','italic','formula','image'] as const
export type ToolbarButtonName = typeof TOOLBAR_BUTTONS[number]
// 组件 props
toolbarOrder?: ToolbarButtonName | (string & {})
```
- **拓展**：类型定义为 ToolbarButtonName | (string & {}) 保留自定义扩展，同时给默认值自动补全
- *来源：admin-workspace-new，2026-08-11*

### 5. 合并主干不要镜像格式化分支
- **技能点**：掌握长分支合并主干的正确姿势：先直接 merge，冲突处采纳主干版，再统一格式化并重做语义改动
- **坑点**：为消除格式差异，把主干也全量 prettier 成镜像分支再 merge，因合并 base 是未格式化版本导致行布局错位，冲突从 51 文件/118 块放大到 117 文件/414 块，还把「仅我方改」的文件变成双方都改
- **解决方案**：直接 git merge <主干> → 冲突整体采纳主干版 → npm run format 统一排版 → 按清单 grep 驱动地在主干基线上重做语义成果
```text
git merge origin/master   # 冲突处整体采纳主干版
npm run format
# 按清单 grep 定位并重做语义改动
```
- **拓展**：合并前生成「语义改动清单」（文件+改动点），合并后按清单逐项勾销，可复用为合并 checklist
- *来源：admin-workspace-hr，2026-10-14*

### 6. 改动落盘与分支版本核对
- **技能点**：养成「编辑是否真正落盘、目标副本是否为预期分支」的核对习惯，避免基于旧快照白改
- **坑点**：会话切换后磁盘仍是旧版，前期部分 edit 实际未生效；排查线上问题时不看部署分支，误判本地代码
- **解决方案**：改完 grep 复核关键符号与引用数；版本疑问先 git show <部署分支>:<文件> 对比，再回写
```text
git show master:src/xxx.ts   # 对比部署分支与本地
grep -rn "已删符号" src/      # 删除后确认 0 残留引用
```
- **拓展**：重要改动后跑一次 lint/tsc 并区分「新增错误」与「预存基线」，避免被存量噪音误导
- *来源：admin-workspace-new，2026-08-11*

