---
title: "工作台日报 · 2026-11-09"
date: 2026-11-09 07:03:21
categories: [工作日记]
tags: ["日报", "AI工具", "大模型", "网络安全", "AI Agent"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-09

## 🔥 行业热点

- [Claude Code's "Don't Ask Again" exposes passwords in 1,122 public GitHub files](https://apiiro.com/blog/coding-agents-leaking-secrets-github-dont-ask-again) — *Hacker News*
  - 📌 **内容**：标题指向 Claude Code 的“不再询问”授权选项可能让敏感信息被写入文件并随仓库公开，反映出 AI 编码工具在权限与凭据处理上的风险面。
  - 💡 **学习**：使用 AI 编码助手时应审视自动授权范围，避免让工具读取或写入含密钥的配置，并配合 secret scanning 做提交前检查。
  - 🧭 **拓展**：可在自己的仓库接入 gitleaks 或 trufflehog 做一次全历史扫描，验证是否存在无意提交的凭据。
- [Outlook.com Is Quietly Rejecting Email from Domains Without DMARC](https://mailroute.net/blog/outlook-rejecting-email-without-dmarc) — *Hacker News*
  - 📌 **内容**：大型邮件服务商开始对缺少 DMARC 策略的域名采取更严格的拒收行为，邮件送达能力正与域名认证配置强绑定。
  - 💡 **学习**：可借此理解 SPF、DKIM、DMARC 三件套的作用与部署顺序，以及邮件身份认证对系统可靠性的影响。
  - 🧭 **拓展**：用自己域名发布一条从 p=none 起步的 DMARC 记录，再用公开检测服务观察报告与策略生效情况。
- [Agent memory needs its OpenTelemetry moment](https://www.lorekit.io/blog/open-agent-memory-format) — *Hacker News*
  - 📌 **内容**：标题主张 Agent 的记忆层缺少统一的可观测与互操作标准，需要类似 OpenTelemetry 那样的通用规范来追踪记忆读写。
  - 💡 **学习**：可以借鉴分布式追踪的 span/attribute 思路，为 Agent 的记忆检索、写入和召回过程设计可观测埋点。
  - 🧭 **拓展**：先阅读 OpenTelemetry 语义约定，再为自己的 Agent 原型加入记忆读写链路追踪并对比不同记忆策略效果。
- [What I run on my $10 USD OpenStick homelab](https://blog.bosscher.ch/posts/openstick/) — *Hacker News*
  - 📌 **内容**：作者分享在极低成本的便携硬件棒上搭建自托管服务的实践，展示轻量 homelab 的可行边界。
  - 💡 **学习**：可了解资源受限设备上运行容器化服务、反向代理与常驻守护进程的取舍与优化思路。
  - 🧭 **拓展**：用树莓派或旧笔记本复现一套最小自托管栈，记录内存占用与稳定性表现。
- [Goodbye Claude, hello Pi](https://caleb.software/posts/claude-to-pi.html) — *Hacker News*
  - 📌 **内容**：标题显示开发者从一款 AI 编码助手迁移到另一款工具的使用体验对比，属于 AI 编程工具选型话题。
  - 💡 **学习**：可关注不同 AI 编码工具在上下文组织、工具调用与工作流集成上的差异，形成自己的评估维度。
  - 🧭 **拓展**：用同一组真实任务在两个工具上做对照测试，记录修改成功率与人工返工量。
- [Goldman Sachs is pulling Claude Code from its bankers and traders](https://www.thestateofai.com/news/goldman-sachs-claude-code-traders-bankers) — *Hacker News*
  - 📌 **内容**：大型金融机构调整内部 AI 编码工具的使用范围，反映出企业级 AI 工具在合规、审计与数据边界上的现实约束。
  - 💡 **学习**：可从中思考企业引入 AI 编码工具的权限模型、审计日志与数据脱敏等工程配套能力。
  - 🧭 **拓展**：在团队内梳理一份 AI 工具准入清单，覆盖数据分级、日志留存与可回滚策略并小范围试点。
- [MaxxBench – "Nice DeepSWE score. But is it true?"](https://maxxbench.unlap.ai/research/why-maxxbench/) — *Hacker News*
  - 📌 **内容**：标题对现有代码类基准分数的可信度提出质疑，讨论评测方法与得分能否反映真实能力。
  - 💡 **学习**：可学习如何设计抗污染的评测集、如何识别基准泄漏与过拟合，以及阅读榜单时的批判性视角。
  - 🧭 **拓展**：自建一套小规模、私有且贴近业务的评测任务，定期对候选模型重跑并对比公开榜单。
- [Llama-modchip: 284B DeepSeek on 64GB RAM, no GPU](https://github.com/synchopate/llama-modchip) — *Hacker News*
  - 📌 **内容**：标题显示在无 GPU、仅 64GB 内存的机器上运行超大参数模型的尝试，属于推理侧内存与量化优化方向。
  - 💡 **学习**：可了解权重流式加载、量化压缩与内存换入换出等让大模型脱离高端显卡运行的技术路径。
  - 🧭 **拓展**：在本地用 llama.cpp 类工具尝试低比特量化模型，记录内存峰值与每秒生成速度作为对比基线。
- [Show HN: Coldframe, a honeypot that poses as a supermarket fridge controller](https://github.com/hedgerow-dev/coldframe) — *Hacker News*
  - 📌 **内容**：一个伪装成超市冷柜控制器的蜜罐项目，用于诱捕扫描与攻击流量并观察物联网设备的真实威胁形态。
  - 💡 **学习**：可学习蜜罐的诱饵服务设计、协议伪装方式以及如何采集和分析攻击日志。
  - 🧭 **拓展**：在隔离网络或云主机上部署一个低交互蜜罐，观察一周内的扫描来源与常见探测路径。
- [Declarative Dragging](https://declarative-dragging.github.io/) — *Hacker News*
  - 📌 **内容**：标题指向以声明式方式实现拖拽交互的编程思路，通常涉及状态描述与手势事件解耦。
  - 💡 **学习**：可学习如何把拖拽的位置、约束与落点规则抽象为声明式配置，减少命令式事件处理代码。
  - 🧭 **拓展**：用现有前端框架实现一个可排序列表，对比声明式方案与手写事件方案的可维护性。

## 🌟 GitHub 热门开源项目

> 📈 本周候选 63 个仓库中「Agent 框架」占 23 个，是当前最活跃赛道；周增量破千的仓库：cathrynlavery/diagram-design、tt-a1i/archify、rohitg00/ai-engineering-from-scratch。

- [cathrynlavery/diagram-design（Agent Skills）](https://github.com/cathrynlavery/diagram-design) — *GitHub · HTML · +19.6k/周 · 总 65.9k star*
  - 📌 **是什么**：面向编码智能体的图表设计技能，用自包含 HTML 和 SVG 生成编辑级图表，强调摆脱粗糙的 Mermaid 风格。
  - 💡 **学习点**：学习如何把设计规范封装成可复用的 agent skill，并用 HTML/SVG 约束可视化产出的质量。
  - 🧭 **上手**：读其 skill 定义与一个示例 HTML，跑通同一个小图表并对比 Mermaid 输出效果。
- [tt-a1i/archify（Agent Skills）](https://github.com/tt-a1i/archify) — *GitHub · JavaScript · +14.8k/周 · 总 94.7k star*
  - 📌 **是什么**：把想法、计划或代码库转成可交互架构图的 agent skill，适配多种编码智能体。
  - 💡 **学习点**：学习 agent 如何分析代码库结构并生成可交互的前端图表，以及技能入口与渲染层的分工。
  - 🧭 **上手**：用一个小型前端项目跑通从代码库到交互图的示例，重点读 skill 入口和图表渲染代码。
- [rohitg00/ai-engineering-from-scratch](https://github.com/rohitg00/ai-engineering-from-scratch) — *GitHub · Python · +11.9k/周 · 总 77.8k star*
  - 📌 **是什么**：从零学习 AI 工程的课程型仓库，覆盖智能体、深度学习、生成式 AI 等方向，强调边学边构建。
  - 💡 **学习点**：系统补齐 AI 工程基础，理解从模型概念到 LLM 应用落地的完整链路。
  - 🧭 **上手**：按课程目录选一个 LLM 或 agent 基础章节，跟着实现一个最小可运行项目。
- [OpenMontageInc/OpenMontage（Agent 框架）](https://github.com/OpenMontageInc/OpenMontage) — *GitHub · Python · 周增量统计中 · 总 78.7k star*
  - 📌 **是什么**：开源智能体视频生产系统，用多条生产流水线、工具和技能知识文件把 AI 编码助手扩展成视频生产环境。
  - 💡 **学习点**：学习多阶段 agent pipeline、工具编排，以及把领域生产知识文件化为技能的方法。
  - 🧭 **上手**：读一条最小生产流水线的定义并跑通示例，理清工具调用链和产物生成步骤。
- [usestrix/strix（Agent 框架）](https://github.com/usestrix/strix) — *GitHub · Python · +7.1k/周 · 总 74.4k star*
  - 📌 **是什么**：开源 AI 渗透测试工具，用智能体自动发现并修复应用漏洞。
  - 💡 **学习点**：学习安全场景下 agent 的规划、工具使用、漏洞验证与修复闭环。
  - 🧭 **上手**：在本地靶场跑一次扫描，读 agent 的任务分解、工具调用和报告生成流程。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue 双向 watch 死循环
- **技能点**：能识别并切断「状态A→数组→状态B→状态A」的循环依赖，用值比较判断打破递归更新
- **坑点**：deep watch 加上每次赋新数组引用，即使折叠标志最终值没变也会互相唤醒，触发 Maximum recursive updates exceeded
- **解决方案**：每步写回前做值比较判断：推导结果与当前值相等就直接 return；回写标志前先比较「当前值 !== 推导值」，避免无变化回写
```text
const next = [leftWidth, 'auto', rightWidth]
if (same(next, localPanelSizes.value)) return
localPanelSizes.value = next

watch(localPanelSizes, (v) => {
  const folded = v[0] === 0
  if (isFilePreviewFolded.value !== folded) isFilePreviewFolded.value = folded
}, { deep: true })
```
- **拓展**：双向同步本身就是隐患，能去掉就改成单向派生（computed 或 v-if），只保留一侧为唯一数据源
- *来源：admin-workspace-new，2026-08-11*

### 2. 事件冒泡与修复边界
- **技能点**：顺 click 链路定位误触发来源，并把修复放在业务调用点而非通用组件里
- **坑点**：footer 复制按钮的点击冒泡到卡片容器的 @click，导致点复制时误触发解析展开/收起；在共享组件里加 stopPropagation 会污染所有调用方
- **解决方案**：业务组件挂 @click.stop 阻断冒泡，共享 CopyText 回退为无事件参数的通用实现
```text
<CopyText
  :text="`题目编号: ${data.questionId}`"
  :copyText="data.questionId"
  @click.stop
/>
```
- **拓展**：通用组件只暴露语义（复制什么），是否阻断冒泡交由调用方决定
- *来源：admin-workspace-new，2026-08-11*

### 3. 共享组件插槽透传扩展
- **技能点**：用条件插槽（$slots 检测）为通用组件扩展视觉定制点，同时不破坏其原有通用能力
- **坑点**：为改折叠图标而直接改组件内部会耦合业务；顺手删掉通用能力（如 storageKey 持久化）会波及别的调用方
- **解决方案**：组件新增 #start-collapsible 条件透传插槽并隐藏官方图标，业务侧用 svgIcon 加 rotate 区分展开/收起；通用能力保留，调用方不传参自动跳过
```text
<!-- ResizablePanels -->
<slot v-if="$slots['start-collapsible']" name="start-collapsible" />
<!-- 调用方 -->
<template #panel-0-start-collapsible>
  <svgIcon class="collapse-icon" :class="{ expanded }" name="arrow-left" />
</template>
```
- **拓展**：通用组件统一遵循「新增插槽或开关，不删旧能力」的向后兼容扩展法
- *来源：admin-workspace，2026-08-10*

### 4. 能力注册表 + 常量类型
- **技能点**：把组件可配置能力收敛为常量对象加联合类型导出，让使用方有据可查、编辑器按钮按统一规则注册
- **坑点**：业务方不知道 :toolbar-order 能传哪些名字，且公式/填空等按钮必须额外配 customButtons 才生效，形成隐式依赖
- **解决方案**：导出 TOOLBAR_BUTTONS 常量与 ToolbarButtonName 类型，补全默认按钮映射，内置 handler 改为无条件注册，业务侧用 B.xxx 常量声明工具栏
```text
export const TOOLBAR_BUTTONS = {
  undo: {}, redo: {}, divider: {}, fillInInsert: {}, formula: {}, image: {}
} as const
export type ToolbarButtonName = keyof typeof TOOLBAR_BUTTONS

type Props = { toolbarOrder?: Array<ToolbarButtonName | string> }
```
- **拓展**：同类可配置组件（表格列、表单控件）都可沉淀「常量 + keyof 类型」的能力清单，配合类型提示避免拼写错
- *来源：admin-workspace-new，2026-08-11*

### 5. 状态精简与单一数据源
- **技能点**：用「删多于加」把可推导、可持久化的冗余状态收敛到唯一来源，降低同步链复杂度
- **坑点**：折叠标志、尺寸数组、storageKey 三者互相同步，任何一侧新增逻辑都要维护整条同步链，容易再引入循环或状态漂移
- **解决方案**：删掉持久化与折叠标志，尺寸只作拖拽内部状态；右栏显隐改用 v-if="isAiReviewer" 而不是在 sizes 里塞 "hidden" 字符串
```text
<template v-if="isAiReviewer" #panel-2>...</template>
const localPanelSizes = ref([760, 'auto', 400]) // 仅拖拽/折叠内部状态
```
- **拓展**：组件「该记的记、该推导的推导」：显隐类交给模板条件，纯交互中间态留在组件内部，持久化按需再开
- *来源：admin-workspace-new，2026-08-11*

### 6. 重构分支合并主干策略
- **技能点**：掌握「格式化分支合并主干」的正确路径，能预判冲突量与行布局错位风险
- **坑点**：用「本地镜像格式化分支」把主干也格式化后再 merge，实测冲突反而放大（117 文件/414 冲突 vs 直接 merge 51/118），因为合并 base 是未格式化版本，还让「仅我方改」的文件变成双方都改
- **解决方案**：直接 git merge 主干 → 冲突处整体采纳主干版 → npm run format 统一排版 → 按清单用 grep 驱动，在主干基线上重做语义改动
```text
git merge origin/master
# 冲突处整体采纳主干版，然后：
npm run format
# 最后按 grep 清单重做语义改动
```
- **拓展**：冲突量差异可作为合并策略的量化验证手段，结论沉淀进 docs/合并master 实施方案
- *来源：admin-workspace-hr，2026-10-14*

### 7. 线上现象≠本地代码
- **技能点**：线上异常时先比对部署分支代码而非本地工作区，避免在未提交改动上找错方向
- **坑点**：测试环境报接口 404（No static resource file/getStsToken），排查后发现报错版本其实是 master 旧代码，迁移改动全在未提交的工作区里
- **解决方案**：先 git show master:<文件> 对照部署分支，确认线上跑的是哪版代码，再决定修哪边
```text
git show master:src/xxx.vue
git log --oneline -5 origin/master
```
- **拓展**：形成「先定版本、再定代码」的排障顺序：确认分支/commit、确认构建产物、再读本地源码
- *来源：admin-workspace-hr，2026-09-24*

### 8. 多副本记忆目录联接
- **技能点**：用 NTFS Junction 让多个代码 clone 共享同一份记忆和规范文档，并能自检联接状态、失效后重建
- **坑点**：主库目录被改名或移动会让联接静默失效，副本「看不见」记忆；且副本在 Web\ 子目录下，按根目录找会误判为不存在
- **解决方案**：先 rmdir 删掉失效链接目录（只删链接不动内容），再 mklink /J 重建，最后用校验脚本复检内容是否可见
```text
rmdir "c:\Work Space\Web\admin-workspace-new\.codebuddy\memory"
mklink /J "c:\Work Space\Web\admin-workspace-new\.codebuddy\memory" "c:\Work Space\admin-workspace\.codebuddy\memory"
python ~/.codebuddy/skills/memory-hub/verify_memory.py
```
- **拓展**：在规范里写清路径与校验命令，任何跨副本引用结论前先确认目标副本存在对应文件（版本可能不同）
- *来源：admin-workspace-new，2026-10-30*

### 9. lint 存量基线区分
- **技能点**：在遗留项目里区分「预存错误」与「本次新增错误」，用差异对比而非绝对值判断改动质量
- **坑点**：项目本身有存量 any、未使用变量等 ERROR，直接看 lint 结果会误判为本次改动引入的问题
- **解决方案**：先确认基线范围，改后只比对新增项；已删符号用 grep 确认 0 处残留引用
```text
grep -rn "syncSizesFromFlags" src/   # 期望 0 命中
npx eslint src/xxx.vue --quiet      # 只关注新增行
```
- **拓展**：可落成「改动前后 lint 快照 diff + 引用残留检查」的固定自查清单
- *来源：admin-workspace-new，2026-08-11*

### 10. 会话编辑未落盘核对
- **技能点**：对「工具已返回成功但磁盘仍是旧版」的异常保持警觉，用实测确认改动是否真正生效
- **坑点**：会话切换后前期对 toolbarConfig.ts / detailComment.vue 的部分 edit 实际未落盘，磁盘仍是旧版，导致后续改动基于错误前提
- **解决方案**：改动后回读文件或跑 lint/构建验证，发现不一致就按清单重新正确应用一遍
- **拓展**：长会话或多副本环境里，关键改动完成后做一次「磁盘复核」再继续下一步
- *来源：admin-workspace-new，2026-08-11*

