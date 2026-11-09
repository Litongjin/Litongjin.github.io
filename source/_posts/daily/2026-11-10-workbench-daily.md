---
title: "工作台日报 · 2026-11-10"
date: 2026-11-10 07:03:20
categories: [工作日记]
tags: ["日报", "大模型", "AI写作", "AI编程", "AI芯片"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-10

## 🔥 行业热点

- [Google Gemini routes personal Drive files and emails to human annotators](https://github.com/thisisme6598/gemini_files/) — *Hacker News*
  - 📌 **内容**：标题指向 Gemini 在处理个人 Drive 文件与邮件时可能引入人工标注环节，涉及 AI 产品中的隐私与数据治理问题。
  - 💡 **学习**：关注 AI 产品数据流向、访问控制与用户授权设计，思考如何在功能与隐私间取舍。
  - 🧭 **拓展**：可查阅该产品隐私政策与数据保留说明，并对照合规要求做数据流审计。
- [DeepX claims inference up to 100B parameters at less than 5 W](https://www.eetimes.com/deepx-tapes-out-second-gen-chip-projects-6-7x-performance-gain-with-lpddr5x-pim/) — *Hacker News*
  - 📌 **内容**：DeepX 宣称可在低于 5W 功耗下运行最高 100B 参数模型推理，指向端侧/低功耗 AI 推理硬件方向。
  - 💡 **学习**：了解模型量化、稀疏化、专用加速器如何降低推理功耗与内存带宽需求。
  - 🧭 **拓展**：可对照其技术白皮书或基准测试，验证不同批量与序列长度下的功耗和吞吐。
- [Show HN: STE80 Humanizer – de-AI text by writing 80% of the way to ASD-STE100](https://github.com/Dodaios/ste80-humanizer) — *Hacker News*
  - 📌 **内容**：该项目用受控英语规范改写文本，试图让 AI 生成内容读起来更自然、更不像机器写作。
  - 💡 **学习**：学习规则约束加改写流程如何用于文本风格控制与后处理。
  - 🧭 **拓展**：可拿同一段 AI 文本比较不同规则阈值下的可读性与保真度。
- [A year after the 100% AI mandate nobody understands our code](https://www.reddit.com/r/ClaudeCode/comments/1xq2n3t/a_year_after_the_100_ai_mandate_nobody/) — *Hacker News*
  - 📌 **内容**：标题讨论全面推行 AI 生成代码一年后，团队对现有代码理解不足的问题，反映 AI 编码带来的可维护性风险。
  - 💡 **学习**：在 AI 辅助编程中重视代码审查、文档、测试与知识共享，避免只追求生成速度。
  - 🧭 **拓展**：可在团队内做小范围 AI 代码贡献度审计，跟踪缺陷率与新人上手时间。
- [Supply chain attack on Qubole customers](https://sentra.io/blog/breaches/inside-the-qubole-supply-chain-attack) — *Hacker News*
  - 📌 **内容**：Qubole 客户遭遇供应链攻击，说明依赖第三方平台或工具链可能成为攻击入口。
  - 💡 **学习**：学习软件供应链安全：依赖锁定、构建产物校验、最小权限与供应商风险评估。
  - 🧭 **拓展**：可梳理自身 CI/CD 与第三方依赖清单，做一次 SBOM 和权限审计。
- [Claude Fable 5.2 and Mythos 5.2](https://www.anthropic.com/claude-fable-and-mythos-5-2) — *Hacker News*
  - 📌 **内容**：条目提到 Claude Fable 5.2 与 Mythos 5.2，可能是新模型或版本发布与对比，关注能力与定位变化。
  - 💡 **学习**：跟踪大模型版本迭代中的上下文、推理、工具调用与成本差异，建立评测习惯。
  - 🧭 **拓展**：用自有任务集对新旧模型做 A/B 评测，记录质量、延迟和费用。
- [Is Opus 5.5 nerfed? Thinking patterns changed on November 3](https://twitter.com/shanemelaugh/status/2119192075505893695) — *Hacker News*
  - 📌 **内容**：讨论 Opus 5.5 是否被削弱以及思考模式变化，涉及模型行为随版本或时间漂移的问题。
  - 💡 **学习**：理解模型行为并非静态，需版本固定、回归测试和提示词兼容性管理。
  - 🧭 **拓展**：对关键提示词建立回归用例，在新版本上复测输出稳定性与推理链差异。
- [E-means: 10x faster k-means at equal loss](https://e-means.github.io/) — *Hacker News*
  - 📌 **内容**：E-means 宣称在损失相当的情况下把 k-means 加速约 10 倍，属于聚类算法优化。
  - 💡 **学习**：学习近似、并行化与初始化优化如何提升经典机器学习算法效率，同时保持损失。
  - 🧭 **拓展**：在公开或自建数据集上复现实验，对比不同簇数、维度和数据规模下的速度与惯性。
- [Our Smallest, Most Efficient Model: Muse Ember](https://research.meta.ai/blog/muse-ember) — *Hacker News*
  - 📌 **内容**：Muse Ember 被描述为更小、更高效的模型，体现大模型向轻量化和推理成本优化发展的趋势。
  - 💡 **学习**：关注小模型蒸馏、量化与架构优化，以及小模型在边缘和低延迟场景的应用。
  - 🧭 **拓展**：用公开基准或业务样例比较小模型与大模型的质量和成本曲线。
- [What is malloc doing, anyway?](https://benmandrew.com/articles/what-is-malloc-doing-anyway) — *Hacker News*
  - 📌 **内容**：文章从 malloc 出发解释动态内存分配器在底层做了什么，属于系统编程基础话题。
  - 💡 **学习**：理解堆管理、内存碎片、分配策略与性能开销，有助于写出更高效的 C/C++ 程序。
  - 🧭 **拓展**：可阅读 glibc 或 jemalloc 源码，并用简单分配模式做性能剖析。

## 🌟 GitHub 热门开源项目

- [Snailclimb/JavaGuide](https://github.com/Snailclimb/JavaGuide) — *GitHub · JavaScript · +580/周 · 总 159.5k star*
  - 📌 **是什么**：面向前后端程序员的 Java 与后端通用面试知识库，覆盖计算机基础、数据库、分布式、系统设计，并延伸出 AI 应用开发、Agent、MCP 等主题。
  - 💡 **学习点**：可当作从前后端开发转向 AI 应用开发的知识地图，顺着它理解 Java 生态里的 Spring AI、MCP、上下文工程是如何实现的。
  - 🧭 **上手**：先定位文档中 AI 应用开发相关章节通读，并对照自己熟悉的前端概念做一份差异笔记。
- [jeecgboot/JeecgBoot（LLM 应用）](https://github.com/jeecgboot/JeecgBoot) — *GitHub · Java · +280/周 · 总 48.4k star*
  - 📌 **是什么**：企业级 AI 低代码平台，把权限、多租户、工作流、报表、大屏与 AI 知识库、聊天、流程编排、MCP 等通用能力做成开箱即用的基座，让 AI 只生成业务差异部分。
  - 💡 **学习点**：看成熟业务系统如何把代码生成、RAG 知识库与 Agent 编排组装成产品，理解“AI 生成”与“平台保底”之间的工程边界。
  - 🧭 **上手**：本地启动一套单体模式 Demo，走一遍按一句话建表并生成前后端页面与菜单的流程，观察生成产物落在哪些目录。
- [DietrichGebert/ponytail（Agent Skills）](https://github.com/DietrichGebert/ponytail) — *GitHub · JavaScript · +27.7k/周 · 总 187.7k star*
  - 📌 **是什么**：一个面向 AI 编码代理的技能插件，用 YAGNI 和“最懒的资深工程师”思路，约束代理产出更少、更必要的代码。
  - 💡 **学习点**：学会用规则文件给编码代理划定边界，这比不断加功能更能提升代理产出质量。
  - 🧭 **上手**：读懂它的技能和规则定义文件，挑一条规则改写成自己项目里的 CLAUDE.md 或 cursor rules 条目后试效果。
- [affaan-m/ECC（Agent Skills）](https://github.com/affaan-m/ECC) — *GitHub · JavaScript · +24.1k/周 · 总 300.2k star*
  - 📌 **是什么**：面向 Claude Code、Codex、Cursor 等编码代理的 harness 性能优化配置集，涵盖技能、直觉式规则、记忆、安全与研究优先的开发流程。
  - 💡 **学习点**：理解编码代理 harness 由技能、记忆、权限、流程等哪些层组成，以及如何系统地调优。
  - 🧭 **上手**：先看仓库中技能与记忆目录的组织方式，再挑记忆或安全一节对照自己现有配置做增补。
- [obra/superpowers（Agent Skills）](https://github.com/obra/superpowers) — *GitHub · Shell · +18.1k/周 · 总 315.1k star*
  - 📌 **是什么**：把代理技能与软件开发方法论结合起来的 agentic 框架，用头脑风暴、子代理驱动开发等流程约束代理行为。
  - 💡 **学习点**：学它如何把拆解、评审、子代理分工这些软件工程流程固化成可复用的技能与工作流。
  - 🧭 **上手**：从技能目录和一个端到端示例流程入手，照该流程在自己项目里完整跑一遍。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步防死循环
- **技能点**：能识别并拆解「状态A→数组→状态B→状态A」的双向 watch 回写环，用值比较守卫收敛同步逻辑
- **坑点**：标志位变化写新数组引用，deep watch 再回写标志位，即使最终值未变也会互相唤醒，直至报 Maximum recursive updates exceeded
- **解决方案**：每个同步方向先算推导值并与当前值比较，相等直接 return；不回写无变化的标志，不产生新的数组引用
```text
function syncSizesFromFlags() {
  const next = [foldLeft ? 0 : leftWidth, 'auto', foldRight ? 0 : rightWidth]
  if (next[0] === localPanelSizes.value[0] && next[2] === localPanelSizes.value[2]) return
  localPanelSizes.value = next
}
watch(localPanelSizes, (v) => {
  const foldLeft = v[0] === 0
  if (isFilePreviewFolded.value !== foldLeft) isFilePreviewFolded.value = foldLeft
}, { deep: true })
```
- **拓展**：多个状态派生同一份数据时优先改成单向派生（computed），从源头消除回写环
- *来源：admin-workspace-new 2026-08-11*

### 2. 点击冒泡归属与共享组件边界
- **技能点**：能区分「组件通用缺陷」与「调用方语义问题」，把修复落在最小责任点，避免污染公共组件
- **坑点**：卡片 footer 的复制按钮点击冒泡到外层容器的 @click，触发了不该有的展开/收起；若直接给共享 CopyText 内部加 stopPropagation 会改变其通用行为
- **解决方案**：共享组件保持无副作用（不拦截事件），在业务调用处加 @click.stop 切断冒泡
```text
<CopyText :text="`题目编号: ${data.questionId}`" @click.stop />
```
- **拓展**：公共组件只保留能力与默认行为，业务语义一律由调用方通过事件修饰符或 props 表达
- *来源：admin-workspace-new 2026-08-11*

### 3. 状态源单一化与魔法值清除
- **技能点**：能判断哪些状态是冗余的派生状态，敢于删掉持久化与双向同步，只保留最小可用状态
- **坑点**：用 sizes 数组里的 'hidden' 字符串控制面板显隐、维护两套折叠标志并与尺寸数组互相同步，状态源分散导致偶发错乱与维护成本
- **解决方案**：显隐改由业务标志直接驱动（v-if），尺寸只作为拖拽内部状态；共享组件保留 storageKey 能力，调用方不再传参即可关闭持久化
```text
<template v-if="isAiReviewer" #panel-2>...</template>
<!-- 不再用 sizes 中的 'hidden' 字符串切换 -->
```
- **拓展**：凡「A 能由 B 推出」的状态都不必独立存储；组件能力保留、调用方取舍，是删功能又不破坏通用性的通用手法
- *来源：admin-workspace-new 2026-08-11*

### 4. 组件能力显式导出
- **技能点**：能用常量对象 + 联合类型把组件的隐式参数契约显式化，让调用方无需读源码即可正确使用
- **坑点**：业务方不知道 toolbar-order 能传哪些按钮名，公式/填空还必须额外配 customButtons 才生效，能力藏在实现细节里
- **解决方案**：导出 TOOLBAR_BUTTONS 常量与 ToolbarButtonName 类型，内置按钮全部无条件注册内置 handler，业务用常量声明工具栏
```text
export const TOOLBAR_BUTTONS = { undo: 'undo', redo: 'redo', formula: 'formula', fillInInsert: 'fillInInsert' } as const
export type ToolbarButtonName = typeof TOOLBAR_BUTTONS[keyof typeof TOOLBAR_BUTTONS]
```
- **拓展**：所有可枚举的配置项都应收敛为导出的常量字典，并让类型对「未知字符串」保留兜底（ToolbarButtonName | string）
- *来源：admin-workspace-new 2026-08-11*

### 5. 主干合并禁用格式化镜像分支
- **技能点**：掌握重构分支与主干合并的正确路径：先语义合并，再统一排版，最后按清单重做改动
- **坑点**：我方全量 prettier 后，为减少冲突把主干也格式化再 merge，实测冲突从 51 文件/118 块放大到 117 文件/414 块——合并 base 是未格式化版本，行布局错位且把「仅我方改」变成「双方都改」
- **解决方案**：直接 git merge 主干，冲突处整体采纳主干版，再 npm run format 统一排版，最后按 grep 清单在主干基线上重做语义成果
```text
git merge origin/master
# 冲突整体采纳主干版 → npm run format → 按清单 grep 重做语义改动
```
- **拓展**：把该流程写进合并实施方案模板与项目规则，避免下个重构分支重复试错
- *来源：admin-workspace-hr 2026-10-14*

### 6. 改动落盘校验与验证口径
- **技能点**：养成「改完回读、逐条顺序编辑、按成本选验证命令」的工程纪律，能区分本次引入与存量问题
- **坑点**：会话切换后部分 edit 实际未落盘，磁盘仍是旧版；同一文件并行编辑基于同一快照会互相覆盖；每次跑 build 耗时 13–35s，且容易把预存 lint/any 错误当成自己引入的
- **解决方案**：同文件多处修改逐条顺序执行并回读关键行确认；默认用 npx tsc --noEmit 验证，lint 结论明确标注「预存、非本次引入」，规则违规清零后才升级为 error
```text
npx tsc --noEmit
# 回读确认落盘
Select-String -Path src/components/toolbar/toolbarConfig.ts -Pattern 'formula'
```
- **拓展**：把验证命令清单与「存量不入账」的判定口径固化到项目规则，新会话直接复用
- *来源：admin-workspace-new 2026-08-11 / admin-workspace-hr 2026-09-08*

