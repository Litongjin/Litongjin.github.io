---
title: "工作台日报 · 2026-11-02"
date: 2026-11-02 07:01:51
categories: [工作日记]
tags: ["日报", "网络安全", "AI Agent", "AI工具", "C语言"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-02

## 🔥 行业热点

- [AI agent swarms in Minecraft fail to build civilization, drown in bureaucracy](https://www.tomshardware.com/tech-industry/artificial-intelligence/ai-agent-swarms-unleashed-in-minecraft-to-build-a-thriving-civilization-50-agent-teams-fail-after-falling-into-paralyzing-bureaucracy) — *Hacker News*
  - 📌 **内容**：多智能体在开放沙盒环境中协作实验，结果因层层审批与协调开销而无法推进，暴露多 Agent 系统的治理难题。
  - 💡 **学习**：观察多 Agent 协调开销与典型失败模式，思考角色分工、投票机制带来的额外成本。
  - 🧭 **拓展**：可在 Minecraft 或模拟沙盒中复现类似多 Agent 实验，对比有无中心调度器时的表现。
- [The AI cost of JavaScript frameworks](https://nolanlawson.com/2026/10/28/the-ai-cost-of-javascript-frameworks/) — *Hacker News*
  - 📌 **内容**：从 AI 辅助开发与上下文消耗角度审视前端框架的隐性成本，讨论框架选择在 AI 时代的新权衡。
  - 💡 **学习**：关注框架体积、结构与文档形态如何影响 AI 生成代码的质量与 token 成本。
  - 🧭 **拓展**：可在同一功能需求下用不同框架让 AI 生成代码，对比返工率与 token 消耗。
- [Leaking Secrets from Air-Gap Computers by Spelling Radio Signals from RAM (2024)](https://arxiv.org/abs/2409.02292) — *Hacker News*
  - 📌 **内容**：利用内存读写产生的无线电信号，把气隙隔离机器中的数据以类似拼写的方式向外泄。
  - 💡 **学习**：学习侧信道与电磁泄漏的攻击思路，理解物理隔离环境的真实威胁模型。
  - 🧭 **拓展**：可查阅 TEMPEST 相关资料，评估自建隔离机房的物理防护与屏蔽措施。
- [No AI, 52 min. Build all Linux Files utils in C by Hand [video]](https://www.youtube.com/watch?v=gQRmSlcTq9c) — *Hacker News*
  - 📌 **内容**：不借助 AI，在约一小时内用 C 手写实现常见的 Linux 文件工具，展示系统调用与命令行工具的底层实现。
  - 💡 **学习**：通过手写 cat、ls 等工具理解文件 I/O、目录遍历与系统调用接口。
  - 🧭 **拓展**：自己用 C 重写 coreutils 子集，并对照 POSIX 行为编写测试。
- [Understanding Email Encryption (2025)](https://www.fastmail.com/blog/email-encryption/) — *Hacker News*
  - 📌 **内容**：系统性梳理邮件加密的现状与实践路线，包括端到端加密与传统方案各自的取舍。
  - 💡 **学习**：理解邮件加密的信任模型、密钥分发与元数据泄露等关键难点。
  - 🧭 **拓展**：用 GnuPG 配置一次签名与加密邮件，与 Web 邮箱流程对比验证。
- [HackerTube: YouTube built from videos recommended by HN](https://karpathy.bearblog.dev/hacker-tube/) — *Hacker News*
  - 📌 **内容**：把 HN 上被推荐的视频汇聚成一个类 YouTube 的浏览站点，属于对社区数据再利用的小型项目。
  - 💡 **学习**：学习如何抓取并重排社区推荐内容，构建轻量前端聚合页面。
  - 🧭 **拓展**：可基于 HN 的公开 API 自建类似的推荐聚合页面并验证排序效果。
- [TCP Use-After-Free: CVE-2026-98432 / CVSS 9.8](https://atlasum.net/blog/cve-2026-98432-linux-ipv4-tcp-uaf) — *Hacker News*
  - 📌 **内容**：披露 TCP 协议栈中的释放后使用漏洞，评分极高，涉及内核网络栈的内存安全问题。
  - 💡 **学习**：了解 UAF 类漏洞的成因、触发条件与网络栈的攻击面。
  - 🧭 **拓展**：查阅该漏洞补丁的 diff，分析修复手法并评估本地环境是否需要升级。
- [Memdoor: A team of 5 Qwen3.8-27B models beats Fable 5 on LiveCodeBench Hard](https://github.com/guregodevo/memdoor/tree/main/examples/workflows/team-solve) — *Hacker News*
  - 📌 **内容**：用多个中等规模模型组队协作推理，在困难代码基准上超过更大的单模型，探索多模型集成路线。
  - 💡 **学习**：学习多模型投票与集成、采样预算分配以及成本与准确率的权衡。
  - 🧭 **拓展**：在自选代码基准上用多个小模型做集成，与单一大模型对比准确率和成本。
- [Lenovo's 'encrypted' fingerprint USB is just an unencrypted SD card in a case](https://www.reddit.com/r/technology/comments/1xl2pr3/lenovos_encrypted_fingerprint_usb_drive_is_just/) — *Hacker News*
  - 📌 **内容**：号称加密的指纹 U 盘实际只是普通 SD 卡加外壳，暴露硬件安全营销与实际实现之间的差距。
  - 💡 **学习**：提醒对宣称加密的硬件产品做实际验证，理解主控与固件在加密中的作用。
  - 🧭 **拓展**：可对同类设备做拆解与取证测试，用 dd 等工具检验存储是否为明文。
- [Jepsen: ParticleDB 1.1.4](https://jepsen.io/analyses/particledb-1.1.4) — *Hacker News*
  - 📌 **内容**：Jepsen 针对 ParticleDB 发布一致性测试报告，检验其在故障场景下宣称的正确性保证。
  - 💡 **学习**：学习 Jepsen 方法论：故障注入、历史记录采集与线性一致性校验。
  - 🧭 **拓展**：阅读报告细节，尝试用 Jepsen 测试自己在用的分布式数据库。

## 🌟 GitHub 热门开源项目

> 📈 本周候选 61 个仓库中「Agent 框架」占 21 个，是当前最活跃赛道；周增量破千的仓库：DietrichGebert/ponytail、affaan-m/ECC、obra/superpowers。

- [Snailclimb/JavaGuide](https://github.com/Snailclimb/JavaGuide) — *GitHub · JavaScript · +362/周 · 总 159.4k star*
  - 📌 **是什么**：一份面向前端/后端开发者的大厂面试与知识体系指南，覆盖计算机基础、数据库、分布式、高并发、系统设计，并新增了 AI 应用开发、MCP、上下文工程等方向的内容整理。
  - 💡 **学习点**：可以把它当作"AI 应用开发"这一新章节的索引，用来补齐从浏览器端转向服务端与 LLM 工程时最容易缺失的底层概念（网络、并发、存储）。
  - 🧭 **上手**：先定位仓库中 AI 应用开发 / MCP / context-engineering 相关目录，挑一篇通读并对照自己已有的前端知识做一份差异笔记。
- [jeecgboot/JeecgBoot（LLM 应用）](https://github.com/jeecgboot/JeecgBoot) — *GitHub · Java · +313/周 · 总 48.4k star*
  - 📌 **是什么**：企业级 AI 低代码平台，主打用自然语言生成表单、流程、报表乃至整站代码，并内置 AI 聊天、知识库、流程编排、MCP 插件等 AI 应用能力，兼容多家大模型。
  - 💡 **学习点**：能直观看到"提示词/技能 → 结构化配置 → 代码生成 → 人工合并"这条 AI 辅助研发流水线是如何被工程化落地的。
  - 🧭 **上手**：从 README 的快速启动步骤跑起在线演示，重点观察一次"一句话生成表单"前后端产物分别长什么样，再对照自己写过的表单代码。
- [DietrichGebert/ponytail（Agent Skills）](https://github.com/DietrichGebert/ponytail) — *GitHub · JavaScript · +31.6k/周 · 总 182.1k star*
  - 📌 **是什么**：一个给 AI 编程助手注入"最懒资深工程师"人格的技能/规则包，核心主张是最佳代码是没写过的代码，倾向用 YAGNI 思路压制过度设计。
  - 💡 **学习点**：可以学到如何用一份可复用的规则文件约束 LLM 的输出倾向，这对前端工程师做提示词工程与代码评审自动化很有参考价值。
  - 🧭 **上手**：读仓库里的规则/提示词定义文件，把它与自己项目里的 cursor rules 或 CLAUDE.md 对照，挑一条"禁止过度抽象"的规则试加进去。
- [affaan-m/ECC（Agent Skills）](https://github.com/affaan-m/ECC) — *GitHub · JavaScript · +24.1k/周 · 总 294.8k star*
  - 📌 **是什么**：面向 Claude Code、Codex、Cursor 等编码代理的运行框架优化系统，围绕技能、直觉、记忆、安全与"先调研再动手"的开发流程做增强。
  - 💡 **学习点**：可以理解编码代理除了模型本身，还需要记忆、工具约束与安全边界这几层工程化设计，是前端同学理解 Agent 运行时的好切口。
  - 🧭 **上手**：先读 README 的架构/概念章节，理清 skills 与 memory 各自的职责，再挑一个最小技能配置在自己的编码代理里启用。
- [obra/superpowers（Agent Skills）](https://github.com/obra/superpowers) — *GitHub · Shell · +16.7k/周 · 总 310.6k star*
  - 📌 **是什么**：一套以 Shell 为主的代理技能框架与软件开发方法论，把头脑风暴、编码、代码评审、子代理协作等环节沉淀成可复用的技能与流程。
  - 💡 **学习点**：重点看它如何把软件开发生命周期拆解成可被代理调用的技能，这正是把前端工程规范迁移到 AI 协作流程的思路。
  - 🧭 **上手**：从技能目录清单入手，选一个最贴近自己日常的环节（如代码评审或方案头脑风暴），按其流程手动走一遍再交给代理执行。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步防死循环
- **技能点**：能识别「状态 A → 数组 → 状态 B → 状态 A」的 watch 回写循环，并用值比较守卫切断递归
- **坑点**：deep watch 数组配合 watch 标志位互相回写，即使值没变，也会因新数组引用不断唤醒对方，直到报出 Maximum recursive updates exceeded
- **解决方案**：每次写入前比较三项当前值与推导值，相等就直接 return；deep watch 内也先判断当前值与推导值是否不同，再回写
```text
function syncSizesFromFlags() {
  const next = [leftWidth, 'auto', rightWidth]
  const cur = localPanelSizes.value
  if (next[0] === cur[0] && next[2] === cur[2]) return
  localPanelSizes.value = next
}

watch(localPanelSizes, (v) => {
  const folded = v[0] === 0
  if (isFilePreviewFolded.value !== folded) isFilePreviewFolded.value = folded
}, { deep: true })
```
- **拓展**：可抽象成 useTwoWaySync 之类的工具，统一封装「比较后再写」的守卫；更彻底的做法是只保留单一数据源
- *来源：admin-workspace-new / admin-workspace-test，2026-08-11*

### 2. 事件冒泡在业务层拦截
- **技能点**：能判断共享组件与业务组件的修复边界，用 @click.stop 在最靠近问题的地方阻断冒泡
- **坑点**：footer 里的复制按钮点击冒泡到外层卡片的 @click，导致复制同时触发展开/收起；如果直接在共享 CopyText 组件里加 stopPropagation，会污染其他调用方
- **解决方案**：在业务组件的使用处加 @click.stop，共享组件回退为通用实现（handleCopy 不带 event）
```text
<CopyText
  :text="`题目编号: ${data.questionId}`"
  :copyText="data.questionId"
  @click.stop
/>
```
- **拓展**：可约定：共享组件只暴露语义事件，阻止冒泡的责任永远由调用方承担
- *来源：admin-workspace-test，2026-08-11*

### 3. 显隐用 v-if 而非魔法字符串
- **技能点**：掌握状态职责最小化：可见性由业务布尔量控制，尺寸数组只负责尺寸
- **坑点**：用 sizes 数组里的 "hidden" 字符串兼作显隐开关，导致同一个状态承载两种语义，折叠、持久化、显隐互相纠缠
- **解决方案**：右栏改用 <template v-if="isAiReviewer" #panel-2>，尺寸数组退回纯拖拽内部状态，显隐不再进入尺寸计算
```text
<template v-if="isAiReviewer" #panel-2>
  <AiReviewerPanel />
</template>
```
- **拓展**：类似地，面板折叠尽量复用组件原生的 collapsible 能力，不要自建折叠标志和尺寸换算
- *来源：admin-workspace-new，2026-08-11*

### 4. 共享组件能力靠调用方裁剪
- **技能点**：理解「组件通用能力 + 调用方按需启用」的分层，能用空值或默认值关闭特性而不破坏组件
- **坑点**：业务方不想要尺寸持久化，容易顺手去改共享的 ResizablePanels；但改组件会影响其他调用方
- **解决方案**：组件保留 storageKey 能力（空 key 时 saveStorage 自动跳过、无副作用），只让 batchInput 不再传 storageKey，并删掉配套的双向同步与折叠标志
```text
const saveStorage = () => {
  if (!props.storageKey) return
  localStorage.setItem(props.storageKey, JSON.stringify(props.sizes))
}
// 调用方：<ResizablePanels v-model:sizes="localPanelSizes" />  // 不传 storageKey
```
- **拓展**：可把这类可选能力做成显式 feature flag 或默认关闭的 props，并在组件文档里标注「空值即禁用」
- *来源：admin-workspace-new，2026-08-11*

### 5. 编辑器工具栏常量 + 联合类型
- **技能点**：会用 as const 常量表 + keyof 联合类型为配置项建模，让业务方在 IDE 里就能列出所有可用的按钮名
- **坑点**：业务组件不知道 :toolbar-order 支持哪些名字；公式、填空必须额外配 customButtons 才能用，内置 handler 被错误地绑定在自定义通道上
- **解决方案**：新增 TOOLBAR_BUTTONS 常量与 ToolbarButtonName 类型，补全默认按钮映射，formula、image 改为无条件注册内置 handler，业务侧用 B.xxx 声明工具栏
```text
export const TOOLBAR_BUTTONS = {
  undo: 'undo', redo: 'redo', divider: 'divider',
  fillInInsert: 'fillInInsert', formula: 'formula', image: 'image',
} as const
export type ToolbarButtonName = keyof typeof TOOLBAR_BUTTONS
type Props = { toolbarOrder?: (ToolbarButtonName | string)[] }
```
- **拓展**：同类配置项（如 panel 名、事件名）都可以照此模式建立注册表，并在类型层做受控扩展（ToolbarButtonName | string）
- *来源：admin-workspace-new，2026-08-11*

### 6. 主干合并不要用镜像格式化分支
- **技能点**：掌握大规模重构分支合并主干的正确顺序：先直接 merge，再统一格式化，最后按清单重做语义改动
- **坑点**：我方做过全量 prettier，又把主干也格式化后 merge（镜像分支法），实测冲突反而放大（117 个文件、414 个冲突块，而直接 merge 为 51 个文件、118 个冲突块），因为合并 base 是未格式化版本，两侧行布局错位
- **解决方案**：直接 git merge 主干 → 冲突处整体采纳主干版 → npm run format 统一排版 → 按 grep 生成的影响清单在主干基线上重做语义成果
```text
git merge origin/master
# 冲突处整体采纳主干版（theirs）
npm run format
grep -rn "<本次改造特征符号>" src/   # 按清单逐项重做语义改动
```
- **拓展**：把「合并前先确认两侧格式化基线一致」写进合并检查清单，避免再次误用镜像分支
- *来源：admin-workspace-hr，2026-10-14*

### 7. 改动落盘与残留校验
- **技能点**：具备「编辑未必真的写入磁盘」的警惕性，改动后用 git diff、grep 做闭环验证
- **坑点**：会话切换后，此前对工具栏配置和业务组件的部分 edit 实际没有写入磁盘，磁盘上仍是旧版本；只看会话里的操作记录会误以为已经改完
- **解决方案**：改完重新读取文件确认内容，用 grep 校验已删符号 0 处残留引用，再跑 lint 确认没有新增错误
```text
grep -rn "isFilePreviewFolded\|syncSizesFromFlags\|PAPER_BATCH_INPUT_PANEL_SIZES" src/
# 期望 0 命中
git diff --stat
```
- **拓展**：可以把「grep 已删符号 + 读取关键文件头」固化为提交前的自检脚本
- *来源：admin-workspace-new，2026-08-11*

