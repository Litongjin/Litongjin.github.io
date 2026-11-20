---
title: "工作台日报 · 2026-11-21"
date: 2026-11-21 07:01:55
categories: [工作日记]
tags: ["日报", "AI", "互联网产品", "大模型", "网络安全"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-21

## 🔥 行业热点

- [Frostbite: A new CPU architecture discovered, designed and verified by AI](https://architectlabs.com/blog/frostbite) — *Hacker News*
  - 📌 **内容**：该条目关注由AI辅助发现、设计并验证的CPU架构，强调自动化方法可能改变硬件设计流程。
  - 💡 **学习**：可学习如何用AI/形式化验证结合的方式，加速芯片架构探索、设计验证和缺陷发现。
  - 🧭 **拓展**：可尝试用一个小型处理器核心复现“AI辅助生成架构+自动化测试验证”的流程。
- [Google Workers Tell Their Employer Not to Punish Schools That Don't Want AI](https://www.motherjones.com/politics/2026/11/google-alphabet-union-school-ai-protest-gemini-workers-petition/) — *Hacker News*
  - 📌 **内容**：该条目反映科技员工对AI在教育场景推广方式的关注，核心是AI采用与企业责任之间的张力。
  - 💡 **学习**：可学习AI产品落地时如何兼顾用户选择权、隐私风险、合规约束与组织接受度。
  - 🧭 **拓展**：可调研学校或企业AI采纳政策，对比自愿采用、默认关闭和例外处理的机制差异。
- [OPM Data Was Breached on the Same Day as the FAA Ground Stop, per Internal Email](https://www.muellershewrote.com/p/breaking-opm-data-was-breached-on) — *Hacker News*
  - 📌 **内容**：该条目报道疑似政府数据泄露与航空关键系统中断同日发生，突出安全事件对基础设施的影响。
  - 💡 **学习**：可学习数据泄露检测、事件响应、供应链安全和关键系统异常关联分析。
  - 🧭 **拓展**：可基于泄露邮件场景搭建异常登录、数据外发与权限变更告警规则。
- [Claude solves two open problems in the physics of black holes](https://www.anthropic.com/research/claude-black-hole-physics) — *Hacker News*
  - 📌 **内容**：该条目显示Claude被用于尝试解决黑洞物理中的开放问题，体现大模型在科研推理中的潜在能力。
  - 💡 **学习**：可学习如何把大模型用于科学问题分解、假设提出与推导检查，同时配合符号计算验证结果。
  - 🧭 **拓展**：可在小尺度物理问题上让模型提出推导路径，并用数值/符号工具逐步验证。
- [Signal Is Serious About Protecting Your Privacy](https://signal.org/blog/protecting-your-privacy/) — *Hacker News*
  - 📌 **内容**：该条目关注Signal在隐私保护上的产品立场，可能涉及端到端加密、数据最小化与元数据治理。
  - 💡 **学习**：可学习即时通讯产品的隐私架构、安全默认设置和元数据风险控制。
  - 🧭 **拓展**：可对比不同IM应用的元数据处理方式，并围绕威胁模型做安全检查表。
- [Why there are still no non-WebKit browsers on iOS](https://blog.mozilla.org/en/mozilla/internet-policy/apple-ios-browser-engine-rules/) — *Hacker News*
  - 📌 **内容**：该条目讨论iOS上非WebKit浏览器长期缺席的原因，核心涉及平台政策、引擎限制和生态治理。
  - 💡 **学习**：可学习浏览器内核、移动端审核政策与系统权限如何影响前端兼容和独立引擎发展。
  - 🧭 **拓展**：可调研iOS浏览器审核规则，并用测试页验证不同引擎下的兼容差异。
- [If you stop writing code, you also get worse at reviewing it](https://cekrem.github.io/posts/if-you-stop-writing-code-you-also-get-worse-at-reviewing-it/) — *Hacker News*
  - 📌 **内容**：该条目强调亲手写代码对代码审查能力的维持作用，关注工程判断力的实践来源。
  - 💡 **学习**：可学习通过持续小步编码、重构和缺陷复盘，保持对代码坏味道、边界条件与维护成本的敏感度。
  - 🧭 **拓展**：可设定每周亲手编码与审查样本，跟踪缺陷发现率与审查耗时变化。
- [Language Models Can Rewrite Their Own Weights](https://arxiv.org/abs/2611.21885) — *Hacker News*
  - 📌 **内容**：该条目指向语言模型修改自身权重的能力，涉及模型自演化、更新机制与潜在安全风险。
  - 💡 **学习**：可学习参数更新、LoRA/元学习思路以及自我修改场景下的稳定性评测和隔离策略。
  - 🧭 **拓展**：可在沙箱中尝试仅更新少量权重或适配器，并比较修改前后的输出可靠性。
- [FFmpeg 9.1 'Gauss'](https://ffmpeg.org/download.html#release_9.1) — *Hacker News*
  - 📌 **内容**：该条目是FFmpeg版本更新，作为多媒体处理基础工具，版本迭代通常影响编解码、滤镜与开发链路。
  - 💡 **学习**：可关注多媒体框架升级对编解码能力、硬件加速、跨平台构建和依赖兼容的影响。
  - 🧭 **拓展**：可拉取新版本进行构建和基准测试，验证常用音视频格式的处理行为。
- [Show HN: I made an audio-to-MIDI model 60x more efficient](https://kobi.music/research/dettato) — *Hacker News*
  - 📌 **内容**：该条目展示一个更高效的audio-to-MIDI模型，重点在音频转符号音乐任务上的性能优化。
  - 💡 **学习**：可学习音频特征提取、模型量化/蒸馏、低延迟推理优化和MIDI转换评测方法。
  - 🧭 **拓展**：可用公开音频数据集对比基线模型，评估准确率、延迟与资源占用之间的取舍。

## 🌟 GitHub 热门开源项目

- [morluto/rea（Agent Skills）](https://github.com/morluto/rea) — *GitHub · TypeScript · +123.9k/周 · 总 471.4k star*
  - 📌 **是什么**：面向逆向工程场景的智能代理或 Agent Skills 集合，关注从应用行为分析到二进制逆向的工作流。
  - 💡 **学习点**：学习如何把复杂安全/逆向任务拆解成可编排的 Agent Skill，并理解多语言工具链接入方式。
  - 🧭 **上手**：阅读 README 和示例目录，对比其 Skill 如何定义输入、调用 CLI/模型工具以及输出逆向结果。
- [Leonxlnx/taste-skill（Agent Skills）](https://github.com/Leonxlnx/taste-skill) — *GitHub · JavaScript · +19.5k/周 · 总 119.3k star*
  - 📌 **是什么**：给 AI 编码代理注入设计审美与浏览器验证能力的 Agent Skills，强调艺术方向、参考图和反 AI 味页面生成。
  - 💡 **学习点**：学习如何把前端设计规范、视觉参考和浏览器检查变成可复用的代理技能，而不是只写 prompt。
  - 🧭 **上手**：运行一个页面生成示例，对比启用 skill 前后在布局、样式、参考图引用和浏览器检查环节的差异。
- [Panniantong/Agent-Reach（MCP 工具）](https://github.com/Panniantong/Agent-Reach) — *GitHub · Python · +13.3k/周 · 总 113.6k star*
  - 📌 **是什么**：为 AI Agent 提供实时互联网信息接入能力的 CLI/工具集合，可能覆盖社交、视频、代码和中文平台的信息读取与搜索。
  - 💡 **学习点**：学习如何通过 CLI 或 MCP 风格接口给 LLM 增加外部信息源，解决知识过时和无法查看实时网页的问题。
  - 🧭 **上手**：按 README 配置一个社交平台只读示例，观察工具 schema、认证方式、返回分页字段和上下文压缩策略。
- [addyosmani/agent-skills（Agent Skills）](https://github.com/addyosmani/agent-skills) — *GitHub · JavaScript · +10.4k/周 · 总 119.4k star*
  - 📌 **是什么**：面向 AI 编码代理的生产级工程技能库，可能覆盖代码审查、测试、调试、部署等软件工程工作流。
  - 💡 **学习点**：学习如何把真实工程实践沉淀成 Agent Skill，而不是停留在一次性聊天提示词。
  - 🧭 **上手**：阅读 examples 或 skills 目录，选择一个代码审查技能，分析其检查项、输入上下文和输出格式约定。
- [pingdotgg/t3code（开发工具）](https://github.com/pingdotgg/t3code) — *GitHub · TypeScript · +9.5k/周 · 总 67.8k star*
  - 📌 **是什么**：用于从任意设备远程控制 Claude Code、Codex 等编码代理的编排/开发工具。
  - 💡 **学习点**：学习如何把本地 CLI Agent 抽象成可远程交互的会话、任务队列和状态同步系统。
  - 🧭 **上手**：跑通连接现有 Claude Code 的最小示例，重点检查会话创建、命令路由、日志回流和手机控制链路。

## 🚀 技能提升点（工作总结汇总）

### 1. 状态同步防递归
- **技能点**：掌握在 Vue/React 多源状态同步中识别递归更新根因的能力，能用相等性守卫和单向派生替代裸双向 watch。
- **坑点**：状态 A 与数组 B 相互 watch，数组每次生成新引用且 deep watch，即使最终值未变也会互相唤醒，触发 Maximum recursive updates。
- **解决方案**：写入前逐项比较当前值与推导值，值相同直接 return；优先使用 computed 或显式事件把同步改成单向链。
```text
if (!isSameSize(localPanelSizes.value, next)) {
  localPanelSizes.value = next;
}
watch(localPanelSizes, sizes => {
  const derived = deriveFromSizes(sizes);
  if (derived.leftFolded !== leftFolded.value) {
    leftFolded.value = derived.leftFolded;
  }
}, { deep: true });
```
- **拓展**：可沉淀为团队规范：双向同步必须设置相等性守卫，避免在 watcher 中返回新数组/对象制造无意义变化。
- *来源：admin-workspace-new | 2026-08-11*

### 2. 事件冒泡与交互边界
- **技能点**：能定位“点击小组件误触发父级交互”的 DOM 事件冒泡问题，并区分业务适配与通用组件的职责边界。
- **坑点**：CopyText 在卡片 footer 点击后冒泡到外层容器的 handleQuestionClick，导致复制触发解析展开/收起。
- **解决方案**：在业务调用点使用 @click.stop 隔离事件；共享组件保持通用，不擅自内置 stopPropagation。
```text
<CopyText :text="题目编号: xxx" :copyText="data.questionId" @click.stop />
```
- **拓展**：可延伸到按钮、链接、菜单等嵌套可交互元素，统一做事件边界检查。
- *来源：admin-workspace-new | 2026-08-11*

### 3. 组件配置类型化注册
- **技能点**：能把编辑器工具栏这类隐式字符串配置升级为可枚举常量、联合类型和自动注册 handler，降低业务接入成本。
- **坑点**：业务不知道 toolbar-order 可填哪些名称，特殊按钮还需 customButtons 才能生效，配置容易漏且难校验。
- **解决方案**：导出 TOOLBAR_BUTTONS 常量与类型，内置常用 handler 无条件注册，业务只传声明式数组。
```text
export const TOOLBAR_BUTTONS = ['undo', 'redo', 'bold', 'formula', 'image'] as const;
type ToolbarButtonName = typeof TOOLBAR_BUTTONS[number];
const toolbarOrder: ToolbarButtonName[] = ['bold', 'formula'];
```
- **拓展**：可延伸到主题、表格、表单等复杂组件配置，建立“类型化配置 + 默认能力自动注册”模式。
- *来源：admin-workspace-new | 2026-08-11*

### 4. 通用组件插槽透传
- **技能点**：掌握为通用布局组件增加可定制插槽透传，同时保留原生交互和状态管理边界的能力。
- **坑点**：直接隐藏或替换官方折叠图标容易破坏组件通用行为；状态由 sizes/watch 维护时，业务只应替换视觉层。
- **解决方案**：通用组件按条件透传 #start-collapsible 插槽并隐藏默认图标；业务只在插槽里切换图标方向，折叠动作交给原生组件。
```text
<slot v-if="$slots['start-collapsible']" :name="`panel-${index}-start-collapsible`" />
<style scoped>
.el-splitter__collapse-button {
  display: none;
}
</style>
<template #panel-0-start-collapsible="{ folded }">
  <SvgIcon :name="folded ? 'arrow-right' : 'arrow-left'" />
</template>
```
- **拓展**：可延伸到弹窗、表格、分栏等通用组件，沉淀“原生行为保留 + 插槽只扩展视觉”的边界规范。
- *来源：admin-workspace | 2026-08-10*

### 5. React Hooks 依赖治理
- **技能点**：掌握把 TypeScript/Hooks lint 存量分批清零的方法，能区分规则误报、结构性副作用和真正依赖错误。
- **坑点**：依赖可变对象或 effect 内 setState 与 memoization 规则冲突，导致 exhaustive-deps / set-state-in-effect / preserve-manual-memoization 反复报错。
- **解决方案**：先把可变对象解构为局部变量，初始化/fetch 类 effect 用 useAsyncEffect，弹窗状态复位通过 key 重挂载并同步 useState 初值。
```text
const userId = user?.id;
useEffect(() => {
  if (!userId) return;
  loadDetail(userId);
}, [userId]);

// 弹窗打开时复位：调用方用 key 重挂载
<Dialog key={open ? 'mounted' : 'unmounted'}>
```
- **拓展**：可沉淀为 lint 升 error 清单：单行可独立替换的 any 先批量清，牵出签名链的逐条评估。
- *来源：admin-workspace-hr | 2026-10-20*

### 6. 运行版本与落盘核验
- **技能点**：能用部署分支、磁盘文件、静态检查作为事实源，避免把未生效代码误判为业务逻辑 bug。
- **坑点**：线上或测试环境可能跑 master 旧码，本地也可能出现编辑未落盘、多副本结论不适用等问题。
- **解决方案**：先 git show/diff 对比部署基线，再 grep 关键符号确认代码存在，最后用 lint/tsc 验证改动落地。
```text
git show origin/master:src/service/upload.ts
git diff origin/master -- src/service/upload.ts
npm run lint
rg 'ResizablePanels|isFilePreviewFolded' src -n
```
- **拓展**：可沉淀为故障排查 SOP：先确认分支、构建产物、环境变量和落盘状态，再排查业务逻辑。
- *来源：admin-workspace-hr | 2026-09-24*

