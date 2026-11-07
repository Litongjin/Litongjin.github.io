---
title: "工作台日报 · 2026-11-08"
date: 2026-11-08 07:01:21
categories: [工作日记]
tags: ["日报", "AI Agent", "AI工具", "AI研究", "Linux"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-08

## 🔥 行业热点

- [US FAA says it will phase out floppy disks from air traffic control by 2028](https://www.reuters.com/world/us/us-faa-says-it-will-phase-out-floppy-disks-air-traffic-control-by-2028-2026-11-05/) — *Hacker News*
  - 📌 **内容**：美国空管系统仍依赖软盘等遗留介质，FAA 计划在 2028 年前逐步淘汰。反映关键基础设施中遗留硬件与系统迁移的挑战。
  - 💡 **学习**：可了解遗留系统现代化、介质替换与高可靠性场景下的迁移规划。
  - 🧭 **拓展**：可延伸研究航空/工业控制系统中的遗留技术栈与迁移风险。
- [The end of Microsoft Research's AI Frontiers group](https://hunch.net/?p=13764006) — *Hacker News*
  - 📌 **内容**：微软研究院 AI Frontiers 团队据称结束，反映大厂 AI 研究组织与资源投入的调整。可关注研究机构与产品化团队之间的关系变化。
  - 💡 **学习**：理解 AI 研究组织如何随战略和商业化节奏调整；关注研究到产品的转化路径。
  - 🧭 **拓展**：可追踪微软后续 AI 研究发布与团队变动来验证趋势。
- [The Components of Agent Memory](https://www.anthropic.com/research/components-of-agent-memory) — *Hacker News*
  - 📌 **内容**：文章拆解 Agent 记忆系统的组成，涉及短期上下文、长期存储与检索等方向。对构建有状态 AI Agent 有架构参考价值。
  - 💡 **学习**：学习 Agent 记忆分层、检索增强与状态管理思路。
  - 🧭 **拓展**：可用现有 Agent 框架实现简单记忆模块并对比效果。
- [Claude Code's "Don't Ask Again" exposes passwords in 1,122 public GitHub files](https://apiiro.com/blog/coding-agents-leaking-secrets-github-dont-ask-again) — *Hacker News*
  - 📌 **内容**：标题指出 Claude Code 的“不再询问”选项可能导致密码等敏感信息被写入公开 GitHub 文件。反映 AI 编程工具权限与秘密管理风险。
  - 💡 **学习**：学习 AI 编码助手使用中的权限最小化、密钥管理与提交前扫描。
  - 🧭 **拓展**：可在 CI 中接入 secret scanning，并检查本地 AI 工具配置。
- [Driver AI in Stunt Car Racer, Commodore 64](https://gamesexplained.com/c64/stunt-car-racer/opponents.html) — *Hacker News*
  - 📌 **内容**：讨论 Commodore 64 赛车游戏中驾驶 AI 的实现，属于复古平台上的游戏编程与 AI 逻辑。可观察有限硬件下的算法设计。
  - 💡 **学习**：学习资源受限环境中的游戏 AI、状态机/路径控制与性能取舍。
  - 🧭 **拓展**：可用模拟器或复古开发工具复现并调参。
- [Libghostty 1.0](https://mitchellh.com/writing/libghostty-1-0) — *Hacker News*
  - 📌 **内容**：Libghostty 1.0 发布，作为终端模拟器相关库/开源项目值得关注。涉及终端渲染、跨平台与开发者工具生态。
  - 💡 **学习**：了解现代终端模拟器的架构、性能优化与库化复用思路。
  - 🧭 **拓展**：可阅读源码并尝试在自有项目中集成或对比其他终端库。
- [A Jev that can ask Opus for help gives Opus accuracy at 4x lower cost](https://hightouch.com/blog/llm-judges-identity-resolution) — *Hacker News*
  - 📌 **内容**：标题描述一种让较小模型在需要时调用更强模型求助的机制，以更低成本接近强模型准确率。体现模型路由/级联与成本优化思路。
  - 💡 **学习**：学习模型级联、按需升级与置信度判断等降本策略。
  - 🧭 **拓展**：可在 Agent 或推理服务中实现路由策略，用评测集验证成本与准确率。
- [Every Mac comes with an (undocumented) family member classifier](https://8rank.ai/game/apple-intelligence-personal-knowledge-graph) — *Hacker News*
  - 📌 **内容**：标题称 Mac 内置未公开的家庭成员分类器，涉及本地机器学习与隐私/照片分类能力。可关注端侧模型与用户数据边界。
  - 💡 **学习**：了解端侧 ML、系统级分类器与隐私设计的讨论点。
  - 🧭 **拓展**：可检查系统文档/隐私设置并用本地照片库做小范围验证。
- [Asahi Linux Progress Linux 7.3](https://asahilinux.org/2026/11/progress-report-7-3/) — *Hacker News*
  - 📌 **内容**：Asahi Linux 发布进展，继续推进 Apple Silicon 上的 Linux 支持。涉及内核、驱动与逆向工程等系统级工作。
  - 💡 **学习**：学习平台移植、驱动适配与开源社区协作方式。
  - 🧭 **拓展**：可在支持的 Mac 上安装体验或跟踪内核补丁。
- [We know 15k websites are infected. how do we tell them?](https://scotthelme.co.uk/we-know-15-579-websites-are-infected-how-do-we-tell-them/) — *Hacker News*
  - 📌 **内容**：讨论如何通知大量被感染网站，涉及网络安全事件响应与披露流程。重点在检测后的沟通与修复协作。
  - 💡 **学习**：学习漏洞/入侵通知、负责任披露与自动化扫描结果处理。
  - 🧭 **拓展**：可设计通知模板与验证流程，结合扫描数据做小规模演练。

## 🌟 GitHub 热门开源项目

- [pbakaus/impeccable（Agent Skills）](https://github.com/pbakaus/impeccable) — *GitHub · Rust · +13.0k/周 · 总 105.5k star*
  - 📌 **是什么**：一套给编码智能体用的「设计词汇表」技能包，把「更精致」「更大胆」「挑毛病」这类主观审美指令变成 agent 能执行的评审与打磨流程，用来拦住 AI 生成的低质设计。
  - 💡 **学习点**：学习如何把模糊的设计品味拆解成可复用的提示词/技能文件，这是前端工程师转 AI 应用时最容易变现的领域知识。
  - 🧭 **上手**：先读仓库里 skill/提示词定义文件，看它如何把 critique、audit 这类词映射成具体检查项，再拿自己项目的页面让 agent 跑一遍对比效果。
- [vectorize-io/hindsight（Agent 框架）](https://github.com/vectorize-io/hindsight) — *GitHub · Python · +12.9k/周 · 总 59.7k star*
  - 📌 **是什么**：面向智能体的记忆层，让 agent 能从历史交互中沉淀并「学会」可复用的经验，而不是每次从零开始。
  - 💡 **学习点**：理解 agent memory 的写入、检索、遗忘抽象，前端做的会话状态管理经验可以直接迁移到长期记忆设计上。
  - 🧭 **上手**：跑通仓库里的最小记忆示例，重点看记忆写入与检索的接口设计，再对比 mem0 的 API 差异。
- [blader/humanizer（Agent Skills）](https://github.com/blader/humanizer) — *GitHub · Python · +11.3k/周 · 总 66.0k star*
  - 📌 **是什么**：一个 agent 技能，用于去除文本中的「AI 味」，让模型生成的文案更接近真人写作。
  - 💡 **学习点**：学习如何用提示词规则系统性地消除模型输出的模板化痕迹，这是所有 LLM 产品文案层的通用技能。
  - 🧭 **上手**：把仓库的技能规则文件通读一遍，然后喂一段自己写的 AI 生成文案进去，逐条对照哪些规则真正生效。
- [K-Dense-AI/scientific-agent-skills（Agent Skills）](https://github.com/K-Dense-AI/scientific-agent-skills) — *GitHub · Python · +8.5k/周 · 总 56.4k star*
  - 📌 **是什么**：面向科研场景的 Agent Skills 集合，把生物、化学、临床等领域的标准工作流封装成可被 agent 直接调用的技能与数据库接口。
  - 💡 **学习点**：观摩一个垂直领域如何把专业知识结构化成技能库，这种「领域知识 → 可执行技能」的映射是所有行业 AI 产品的核心。
  - 🧭 **上手**：挑一个与你背景最接近的技能目录读它的定义文件，看输入输出契约怎么约定，再照葫芦画瓢写一个自己的技能。
- [career-ops-hq/career-ops（LLM 应用）](https://github.com/career-ops-hq/career-ops) — *GitHub · JavaScript · +7.2k/周 · 总 80.9k star*
  - 📌 **是什么**：一个开源的 AI 求职 agent：扫描招聘信息、按你的简历打分筛岗、生成适配 ATS 的简历与求职信，并准备面试。
  - 💡 **学习点**：这是一个完整的 LLM 应用范例，可以学到多步骤工作流编排、结构化打分与文档生成如何串成一条产品链路。
  - 🧭 **上手**：先从它的 prompt 与打分逻辑模块读起，看岗位与简历的匹配分数是怎么被约束和解析的，再用自己的简历跑一次。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步防死循环
- **技能点**：能识别并拆解「状态A→数组→状态B→状态A」的隐式回写链，在任何 watch 回调里养成先比较再赋值的守卫习惯。
- **坑点**：折叠标志与分栏尺寸数组互相 watch，赋值新数组引用触发 deep watch，deep watch 又回写标志，值没变也反复唤醒，最终报 Maximum recursive updates exceeded。
- **解决方案**：在两条 watch 中均加值比较守卫：推导结果与原值相同则直接 return，不写新数组、不回写标志；能省则省，砍掉不必要的双向同步。
```text
const next = [leftWidth, "auto", rightWidth];
if (next[0] === localPanelSizes.value[0] && next[2] === localPanelSizes.value[2]) return;
localPanelSizes.value = next;

watch(localPanelSizes, (v) => {
  const folded = v[0] === 0;
  if (isFilePreviewFolded.value !== folded) isFilePreviewFolded.value = folded;
}, { deep: true });
```
- **拓展**：通用化为「派生状态只单向流动」，尺寸/折叠这类 UI 状态优先做单一数据源 + computed 派生。
- *来源：admin-workspace-new，2026-08-11*

### 2. 事件冒泡与共享组件边界
- **技能点**：能用事件链定位「点A触发了B」的怪异交互，并判断修复该落在业务层还是共享层。
- **坑点**：footer 复制按钮的 click 冒泡到外层卡片容器，误触发展开/收起；直觉改法是给共享 copyText.vue 加 stopPropagation，但会污染所有调用方。
- **解决方案**：在业务组件调用处加 @click.stop 就地拦截，共享组件保持通用（无 event 参数、不做阻止冒泡），并回退共享组件改动。
```text
<CopyText
  :text="`题目编号: ${data.questionId}`"
  :copyText="data.questionId"
  @click.stop
/>
```
- **拓展**：沉淀为约定：共享组件只提供原子能力，交互冲突一律在业务调用点解决。
- *来源：admin-workspace-new，2026-08-11*

### 3. 共享组件扩展靠插槽透传
- **技能点**：掌握在不破坏公共组件默认行为的前提下做定制扩展：透传插槽、条件渲染、默认样式隐藏而非删除。
- **坑点**：业务要自定义折叠图标与面板外观，直接改共享 ResizablePanels 的默认行为会波及其他调用方。
- **解决方案**：共享组件新增具名插槽透传（$slots 检测有才渲染）、默认图标 display:none 隐藏；业务侧用自己的插槽与 CSS 实现；不传 storageKey 时组件内部自动跳过持久化，无需改动组件。
```text
<template v-if="$slots['start-collapsible']" #start-collapsible>
  <slot name="start-collapsible" />
</template>
<style>
.el-splitter-bar .collapsible-icon { display: none; }
</style>
```
- **拓展**：把「插槽透传 + 默认降级」做成公共组件扩展范式，减少为单点需求加 prop 开关。
- *来源：admin-workspace-new，2026-08-10*

### 4. 魔法字符串升级为常量类型
- **技能点**：能把散落的字符串开关收敛为导出常量 + 联合类型，让配置项具备可发现性和编译期校验。
- **坑点**：业务方不知道 toolbar-order 能传哪些名字，公式/填空还必须额外配 customButtons 才生效——配置靠口口相传，注册逻辑还依赖调用方传参。
- **解决方案**：导出 TOOLBAR_BUTTONS 常量与 ToolbarButtonName 类型，类型放宽为 ToolbarButtonName | string 保持兼容；内置 handler 改为无条件注册；业务侧用常量声明工具栏，写错即报错。
```text
export const TOOLBAR_BUTTONS = ['undo','redo','divider','fillInInsert','formula','image'] as const;
export type ToolbarButtonName = typeof TOOLBAR_BUTTONS[number];

// 业务侧
const B = TOOLBAR_BUTTONS;
const toolbar = [B.undo, B.redo, B.divider, B.bold, B.formula];
```
- **拓展**：同一手法可推广到所有「组件配置枚举」场景，并配合文档自动生成可选值列表。
- *来源：admin-workspace-new，2026-08-11*

### 5. 删除改动要闭环验证落盘
- **技能点**：建立了「改动前 grep 建影响清单、改动后 grep 验残留」的验证习惯，并能识别「以为改了其实没落盘」的假象。
- **坑点**：删掉一批折叠标志与同步逻辑后，若不做全量检索可能留下悬空引用；更隐蔽的是会话切换/编辑丢失导致部分 edit 实际未写入磁盘，基于旧文件判断会得出错误结论。
- **解决方案**：删除类改动先 grep 全引用列影响面，改后再 grep 已删符号确认 0 命中；关键文件读回或 git diff 核对确实落盘；只跑 lint/tsc 做增量校验，区分预存基线错误与本次新增。
```text
grep -rn "isFilePreviewFolded\|syncSizesFromFlags\|PAPER_BATCH_INPUT_PANEL_SIZES" src/ || echo "0 residual"
git diff --stat src/views/paper/batchInput.vue
```
- **拓展**：固化为删改脚本：符号清单 → 断言命中数为 0 → 再看 diff，避免人工肉眼核对。
- *来源：admin-workspace-new，2026-08-11*

### 6. 知识分层沉淀与索引
- **技能点**：能把一次性排坑经验收口成可复用的领域知识：硬约束常驻、细节按场景索引、过程流水账只进日报。
- **坑点**：把所有踩坑都堆进一个记忆文件，越长越没人读；过程性试错与被覆盖的旧方案混在结论里，反而误导后续判断。
- **解决方案**：分三层：README 式索引用「硬约束（每次必守）+ 场景→领域文档索引表」，日常随手记写当日日报，收口时把稳定结论写进对应领域文档并只加一行索引。
- **拓展**：配合脚本做多副本一致性体检，结论标注日期与来源副本，过时结论显式标记失效而非静默删除。
- *来源：admin-workspace / admin-workspace-hr，MEMORY.md*

