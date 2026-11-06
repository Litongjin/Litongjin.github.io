---
title: "工作台日报 · 2026-11-06"
date: 2026-11-06 10:52:17
categories: [工作日记]
tags: ["日报", "AI工具", "AI Agent", "大模型", "AI模型"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-06

## 🔥 行业热点

- [AI is forcing us to pay off 40 years of technical debt](https://spectrum.ieee.org/ai-in-cybersecurity-php2shell-vulnerability) — *Hacker News*
  - 📌 **内容**：讨论 AI 加速开发之后，组织不得不面对并偿还过去几十年积累的技术债。
  - 💡 **学习**：理解技术债的复利效应，以及在 AI 辅助编码时代更需要在架构、测试与文档上做长期投入。
  - 🧭 **拓展**：可以从自己项目的测试覆盖率与遗留模块入手，列一份技术债清单并排出偿还优先级。
- [Claude Code sends your email address to websites](https://github.com/anthropics/claude-code/issues/107757) — *Hacker News*
  - 📌 **内容**：讨论 Claude Code 这类编码 Agent 在访问网页时可能把用户邮箱等身份信息带出去，形成隐私泄露的风险面。
  - 💡 **学习**：使用 Agent/CLI 工具时要理解其网络请求携带的元数据，以及对外部服务的调用链。
  - 🧭 **拓展**：用代理或抓包工具审计自己常用 AI CLI 的请求头，验证外发的信息范围。
- [MS Paint Remote Code Execution Vulnerability](https://www.cve.org/CVERecord?id=CVE-2026-83844) — *Hacker News*
  - 📌 **内容**：看似简单的画图程序被曝可被利用执行远程代码，说明老旧桌面组件的攻击面不可忽视。
  - 💡 **学习**：了解 RCE 的常见成因（文件解析、内存破坏），以及最小权限和及时更新等防护思路。
  - 🧭 **拓展**：查阅对应公告与补丁说明，在隔离环境中理解漏洞的触发路径。
- [MCP Explained](https://mcp-explained.com/) — *Hacker News*
  - 📌 **内容**：面向开发者的 MCP（模型上下文协议）科普，说明它如何标准化模型与外部工具、数据源之间的连接。
  - 💡 **学习**：掌握 MCP 的 client/server 架构与工具描述方式，可用于给自建 Agent 接入外部能力。
  - 🧭 **拓展**：用官方 SDK 写一个最小 MCP server，并在支持 MCP 的客户端中验证调用。
- [Baidu released ERNIE 5.5 Flash Base](https://ernie.baidu.com/release/ernie-5-5-flash-base/) — *Hacker News*
  - 📌 **内容**：百度发布 ERNIE 的新版本开放模型，反映国内大模型在开放权重路线上的持续迭代。
  - 💡 **学习**：关注新模型的上下文长度、推理成本与中文能力，评估替换现有模型的可行性。
  - 🧭 **拓展**：在本地用推理框架部署，并用一组自有任务做对比评测。
- [Show HN: A decent text-to-image model trained for ~$20K (Apache 2.0 Weights)](https://www.linum.ai/field-notes/pjit-21k) — *Hacker News*
  - 📌 **内容**：有开发者展示以约两万美元成本训练出可用的文生图模型并开放权重，说明低成本复现成为可能。
  - 💡 **学习**：了解文生图训练中的数据准备、算力预算与效果取舍。
  - 🧭 **拓展**：对照开源扩散模型的训练脚本，估算自己的微调成本。
- [Tencent Launches Hy4, One of the Leading Open-Source AI Models](https://www.tencent.com/tencent-launches-hy4-joining-the-ranks-of-the-worlds-leading-open-source-ai-models/) — *Hacker News*
  - 📌 **内容**：腾讯发布开源大模型，开源模型阵营的竞争进一步加剧。
  - 💡 **学习**：学习从推理成本、许可协议与中文表现三方面评估开源模型。
  - 🧭 **拓展**：用 vLLM 等框架部署，做吞吐与延迟的基准测试。
- [Building a Multi-Agent System from Scratch: Architecture Patterns](https://kadhar.dev/blog/multi-agent-systems-from-scratch) — *Hacker News*
  - 📌 **内容**：从零搭建多智能体系统的架构模式总结，覆盖角色分工、通信与编排。
  - 💡 **学习**：理解多 Agent 中的规划、消息传递、状态管理与失败恢复模式。
  - 🧭 **拓展**：选一个框架实现最小多 Agent 流水线，并做不同编排方式的对比。
- [Launch HN: Rivet (YC W23) – Scale and Orchestrate Agents on S3](https://github.com/rivet-dev/rivet) — *Hacker News*
  - 📌 **内容**：面向 Agent 的编排与扩展平台，把 Agent 运行状态放到对象存储上，探索无服务器化的 Agent 基础设施。
  - 💡 **学习**：了解 Agent 编排中的持久化状态、任务队列与弹性伸缩设计。
  - 🧭 **拓展**：试用其 SDK，与自建队列加函数计算的方案做成本与复杂度对比。
- [Show HN: Stroq – a local audit of what your coding agent did and why](https://github.com/AGGIB/Stroq) — *Hacker News*
  - 📌 **内容**：在本地审计编码 Agent 做了什么、为什么这么做，让 AI 编程过程更可追溯。
  - 💡 **学习**：理解 Agent 可观测性的关键：工具调用轨迹、决策日志与操作回放。
  - 🧭 **拓展**：给自己的 Agent 加结构化日志，实现操作回放与审计。

## 🌟 GitHub 热门开源项目

- [OpenHands/OpenHands（Agent 框架）](https://github.com/OpenHands/OpenHands) — *GitHub · TypeScript · +2.6k/周 · 总 92.8k star*
  - 📌 **是什么**：一个由 AI 驱动开发的 agent 平台：agent 能在沙箱里读写代码、执行命令并与开发者协作完成任务。
  - 💡 **学习点**：看一个完整 agent 运行时如何编排工具调用、沙箱执行与会话事件流，这是理解 coding agent 全链路的好样本。
  - 🧭 **上手**：按 README 的本地启动说明起一次服务，跑一个简单任务，边看日志里边观察 agent 的每一步工具调用。
- [langgenius/dify（LLM 应用）](https://github.com/langgenius/dify) — *GitHub · Python · +2.5k/周 · 总 160.5k star*
  - 📌 **是什么**：低代码的 LLM 应用与 agentic 工作流编排平台，把 RAG、模型接入、工具调用和可视化编排放在同一个协作空间里，可云端或自托管部署。
  - 💡 **学习点**：前端工程师可直接对照它的可视化工作流编辑器，学节点式编排界面如何与后端执行引擎、变量与上下文传递对接。
  - 🧭 **上手**：用 docker compose 起服务后，在编辑器里搭一条“表单输入 → LLM → 输出”的最小 workflow 并跑通。
- [koala73/worldmonitor（MCP 工具）](https://github.com/koala73/worldmonitor) — *GitHub · TypeScript · +2.3k/周 · 总 90.2k star*
  - 📌 **是什么**：实时全球情报仪表盘，把 AI 新闻聚合、地缘监控与基础设施追踪汇到统一的态势感知界面，并提供 MCP server 形态的接口。
  - 💡 **学习点**：学 TypeScript 全栈项目如何把多源实时数据聚合成统一信息面板，并把能力以 MCP 工具的形式暴露给 agent。
  - 🧭 **上手**：先找到 MCP server 的入口文件，看它注册了哪些 tool、参数 schema 与数据来源。
- [hesreallyhim/awesome-claude-code（Agent Skills）](https://github.com/hesreallyhim/awesome-claude-code) — *GitHub · Python · +2.2k/周 · 总 57.3k star*
  - 📌 **是什么**：一份精选资源清单，汇总 Claude Code 的命令、钩子、工作流与配置等可复用素材。
  - 💡 **学习点**：从前端视角梳理 coding agent 的可扩展点（slash command、hooks、子代理），为自己的工具链设计插件面。
  - 🧭 **上手**：从清单里挑一个 hooks 或 slash command 条目，照说明放进自己的配置目录并实际触发一次。
- [unclecode/crawl4ai（MCP 工具）](https://github.com/unclecode/crawl4ai) — *GitHub · Python · +2.1k/周 · 总 87.0k star*
  - 📌 **是什么**：面向 AI 的开源网页抓取与抽取库，把网页清洗成适合 LLM 消费的 markdown，并配套托管的抽取与搜索 API。
  - 💡 **学习点**：学如何把任意网页转成结构化、低噪声的文本，直接决定 RAG 与 agent 的输入质量。
  - 🧭 **上手**：跑 AsyncWebCrawler 的最小示例，对比同一页面在开启与关闭 markdown 过滤时的输出差异。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue 双向 watch 递归循环
- **技能点**：能识别「状态 A → 数组 → 状态 B → 状态 A」的循环同步链路，并用值比较校验让同步过程幂等，杜绝无限更新。
- **坑点**：双向 watch（标志 ↔ 面板尺寸数组）互相回写：每次赋值都产生新数组引用，deep watch 必被唤醒，值没变也照样写，最终报 "Maximum recursive updates exceeded"。
- **解决方案**：在每一跳写入前先比对目标值，相等就直接 return；deep watch 内对每个标志先判断「当前值 !== 推导值」再回写。
```text
function syncSizesFromFlags() {
  const next = [leftWidth.value, "auto", rightWidth.value];
  const cur = localPanelSizes.value;
  if (next[0] === cur[0] && next[2] === cur[2]) return; // 值不变就不写，断掉循环
  localPanelSizes.value = next;
}
watch(localPanelSizes, (v) => {
  if (isFilePreviewFolded.value !== v[0] <= 0) isFilePreviewFolded.value = v[0] <= 0;
}, { deep: true });
```
- **拓展**：用 computed 或单一数据源（尺寸数组为唯一真相）替代双向同步，把标志降级为派生值。
- *来源：admin-workspace-new / 2026-08-11*

### 2. 事件冒泡误触发父级行为
- **技能点**：定位「点了子元素却引发外层行为」的冒泡根因，能在正确的边界（业务调用点 vs 共享组件）上放置阻止冒泡。
- **坑点**：footer 里 CopyText 的点击冒泡到外层容器的 @click，触发题目卡片展开/收起；若在共享组件里加 stopPropagation，会破坏其他调用方的冒泡预期。
- **解决方案**：共享组件保持中立（不拦事件），在业务调用点绑定 @click.stop；需要复制的值用独立 prop 与展示文本解耦。
```text
<!-- business component -->
<CopyText :text="`题目编号: ${id}`" :copyText="data.questionId" @click.stop />
```
- **拓展**：沉淀为约定：共享组件只发事件不吞事件，事件语义由调用方决定。
- *来源：admin-workspace-new / 2026-08-11*

### 3. 通用组件扩展用插槽透传
- **技能点**：扩展通用组件时不侵入其内在逻辑：新增插槽透传 + 隐藏默认 UI，业务侧自行渲染，能力保持向后兼容。
- **坑点**：想换官方折叠图标，直接改组件内部会污染所有调用方并让组件与业务样式耦合。
- **解决方案**：在组件内用 $slots 条件检测透传 #start-collapsible 插槽，同时 display:none 隐藏原生图标；折叠动作仍由原生面板处理，状态链不变。
```text
<!-- shared component -->
<template v-if="$slots.collapsible" #start-collapsible>
  <slot name="collapsible" />
</template>
<style>.official-collapse-icon { display: none; }</style>
```
- **拓展**：同类需求（自定义图标/空态/角标）统一走「插槽透传 + 默认隐藏」范式，组件只负责结构与事件。
- *来源：admin-workspace / 2026-08-10*

### 4. 去魔法字符串与冗余状态
- **技能点**：能区分「组件通用能力」与「调用方业务状态」，删除重复的状态副本，用语义化开关替代魔法值。
- **坑点**：用 sizes 数组里的 "hidden" 字符串控制右栏显隐，又额外维护折叠标志做双向同步，导致状态多份副本、四处回写，是递归更新和后续难维护的温床。
- **解决方案**：显隐改由语义化布尔值（v-if="isAiReviewer"）直接控制，删掉折叠标志、基础宽度常量、双向 watch 与 onMounted 同步；组件的 storageKey 能力保留，仅调用方不再传。
```text
<template v-if="isAiReviewer" #panel-2>
  <AiCheckPanel />
</template>
<!-- 不传 storageKey：ResizablePanels 内 saveStorage 自动跳过 -->
<ResizablePanels v-model:sizes="localPanelSizes" :collapsible="panelCollapsible" />
```
- **拓展**：组件可选能力用「不传即降级」实现（空 key 自动跳过持久化），避免为停用功能改组件内部。
- *来源：admin-workspace-new / 2026-08-11*

### 5. 编辑落盘校验与顺序写文件
- **技能点**：对「以为改了其实没落盘」和「并行编辑互相覆盖」两类事故建立操作纪律：逐条顺序编辑 + 改后回读校验。
- **坑点**：会话切换后磁盘仍是旧版，前期部分 edit 实际未落盘；同一文件的多处编辑若基于同一快照并行执行，会互相覆盖。
- **解决方案**：同一文件的多个编辑逐条顺序执行；改完立即回读目标文件确认内容，并在收尾时 lint 验证「无新增错误 + 已删符号 0 残留引用」。
- **拓展**：把「改后回读 + 残留引用 grep」固化为提交前自检清单。
- *来源：admin-workspace-new / 2026-08-11*

### 6. 结论复用前核对副本与部署码
- **技能点**：判断「现象与本地代码不一致」时的排查路径：先确认目标副本是否存在该文件、部署分支跑的是哪版代码，再谈结论。
- **坑点**：多 clone/多副本停在不同分支，历史排坑结论直接照搬会指向不存在的文件；线上问题常是因为改动还在未提交工作区，测试环境跑的是主干旧码。
- **解决方案**：引用记忆结论前先确认文件存在；线上问题先 git show master:<文件> 与部署分支对比，再定位。
```text
git show master:src/service/file.ts | head -50
git log --oneline -1 master
```
- **拓展**：在记忆索引里显式标注每条结论的适用副本与分支。
- *来源：admin-workspace-hr / 2026-09-24*

### 7. 合并主干的正确姿势
- **技能点**：评估格式化对 merge 冲突的影响，能选择冲突面最小的合并路径，而不是凭直觉先对齐格式。
- **坑点**：用「本地镜像格式化分支」把主干也 prettier 后再 merge，实测冲突被放大（117 文件/414 块 vs 直接 merge 的 51/118）：合并 base 未格式化，两侧行布局错位，还把「仅我方改」的文件变成双方都改。
- **解决方案**：直接 git merge <主干> → 冲突处整体采纳主干版 → npm run format 统一排版 → 按清单 grep 驱动地在主干基线上重做我方语义改动。
```text
git merge master
# 冲突处整体取主干版
git checkout --theirs .
npm run format
# 再按清单逐个重做语义改动
```
- **拓展**：合并前先小范围试算冲突规模，据此选路径；结论写入领域文档备查。
- *来源：admin-workspace-hr / 2026-10-14*

