---
title: "工作台日报 · 2026-11-14"
date: 2026-11-14 07:01:47
categories: [工作日记]
tags: ["日报", "大模型", "AI工具", "开发工具", "网络安全"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-14

## 🔥 行业热点

- [Reverse engineering my Coway air purifier to ditch the cloud](https://scottjg.com/posts/2026-11-11-reverse-engineering-my-coway-air-purifier-to-ditch-the-cloud/) — *Hacker News*
  - 📌 **内容**：围绕空气净化器的云端控制做逆向，目标是通过本地或开源方式替代官方服务。
  - 💡 **学习**：可关注嵌入式协议、网络抓包和 IoT 设备去云化改造的基本思路。
  - 🧭 **拓展**：可尝试在隔离网络中抓包，并用开源固件或本地网关验证控制链路。
- [Elevated errors on claude.ai, Claude Code, and platform.claude.com](https://status.claude.com/incidents/nn11wk485zfn) — *Hacker News*
  - 📌 **内容**：Claude 网页、Claude Code 和平台侧服务错误率上升，可能影响开发者使用 AI 工具的工作流。
  - 💡 **学习**：可思考 AI 开发工具依赖外部平台时的故障监控、降级和重试策略。
  - 🧭 **拓展**：可在 CI 或本地开发中记录调用失败率，并设计备用模型或离线回退方案。
- [AI unlocks Gödel's notebooks, said to contain hundreds of proved theorems](https://www.theguardian.com/science/2026/nov/13/kurt-godel-notebooks-ai-helps-unlock-hidden-discoveries-man-who-transformed-maths) — *Hacker News*
  - 📌 **内容**：AI 被用于挖掘哥德尔手稿，可能把非结构化文献转成可检索、可验证的定理知识。
  - 💡 **学习**：可了解大模型在手写识别、文献结构化和知识图谱构建中的应用。
  - 🧭 **拓展**：可用开源 OCR 与模型对历史手稿或论文数据集做小规模抽取和人工校验。
- [Brazil's Caixa insurer breach exposes names and tax IDs of all 7.6M customers](https://filemanager-cdn.mziq.com/published/3972906b-e50b-4f74-ab74-4d0d32125d11/9567a559-d20a-486e-a7ae-5c6bbc030c5e) — *Hacker News*
  - 📌 **内容**：保险机构大规模客户身份数据泄露，暴露姓名和税号等敏感信息。
  - 💡 **学习**：可学习数据分类分级、最小化存储、加密、密钥管理和泄露响应。
  - 🧭 **拓展**：可用威胁建模检查自有系统中的 PII 流转、权限边界和日志告警。
- [OpenAI disbands its robotics team as its general models start running robots](https://fortune.com/2026/11/13/openai-disbands-robotics-team-as-its-general-models-start-running-robots/) — *Hacker News*
  - 📌 **内容**：OpenAI 调整机器人团队可能反映通用模型正在替代传统机器人专用模块。
  - 💡 **学习**：可关注模型平台化对机器人感知、控制和工作流解耦的影响。
  - 🧭 **拓展**：可在仿真环境或小型机器人上测试通用模型驱动的任务编排能力。
- [Perplexity Decider v1.3: Training, eval, and use cases](https://www.perplexity.ai/hub/blog/perplexity-decider) — *Hacker News*
  - 📌 **内容**：Perplexity 发布 Decider 新版训练与评测信息，可能用于搜索或问答中的决策能力。
  - 💡 **学习**：可学习检索增强产品中模型选择、意图判断和离线评测设计。
  - 🧭 **拓展**：可构建小型查询集，对比不同模型在检索、摘要和拒答上的表现。
- [Ask HN: What is the state of self-hosting email in 2026?](https://news.ycombinator.com/item?id=50488230) — *Hacker News*
  - 📌 **内容**：讨论自托管邮件在送达、反垃圾和身份验证上的现状，偏向开发者运维选择。
  - 💡 **学习**：可了解 SMTP、SPF、DKIM、DMARC、IP 信誉等邮件基础设施要点。
  - 🧭 **拓展**：可在独立域名下部署邮件服务，并用主流邮箱测试送达率。
- [Rust 1.100](https://blog.rust-lang.org/2026/11/12/Rust-1.100.0/) — *Hacker News*
  - 📌 **内容**：Rust 发布新版本，语言特性、标准库或工具链可能带来更新。
  - 💡 **学习**：可关注升级对现有项目、依赖兼容性和性能特性的影响。
  - 🧭 **拓展**：可在测试分支运行 rustup 升级、clippy 和集成测试，验证迁移成本。
- [It Takes 16 Clicks for a Student to Open a PowerPoint on an iPad](https://www.reddit.com/r/Teachers/comments/1xuup7a/technology_sucks_it_literally_takes_16_clicks_for/) — *Hacker News*
  - 📌 **内容**：通过教学设备中打开 PPT 的繁琐点击，反映移动端产品与身份权限链路的体验问题。
  - 💡 **学习**：可学习用户任务分析、认证授权链路梳理和多端界面流程优化。
  - 🧭 **拓展**：可对一个关键任务做计时和点击数统计，找出可简化的登录、授权或加载步骤。
- [After 600 experiments, here's what I discovered about JEPA world models](https://haddyalchaer.com/jepa) — *Hacker News*
  - 📌 **内容**：分享大量 JEPA 世界模型实验结果，关注表征学习和世界模型训练经验。
  - 💡 **学习**：可学习自监督表征、世界模型训练目标、消融实验和评测指标。
  - 🧭 **拓展**：可基于开源实现复现小规模实验，对比不同数据规模和损失函数效果。

## 🌟 GitHub 热门开源项目

- [AstrBotDevs/AstrBot（Agent 框架）](https://github.com/AstrBotDevs/AstrBot) — *GitHub · Python · +5.0k/周 · 总 46.9k star*
  - 📌 **是什么**：把 IM 平台、LLM、插件与工具网关整合在一起的多平台 AI Agent 开发框架。
  - 💡 **学习点**：前端工程师可学习消息接入、插件编排与 LLM 调用如何组合成可部署助手。
  - 🧭 **上手**：从 README 的部署示例切入，跟踪一条消息从接收、模型处理到回复的完整链路。
- [headroomlabs-ai/headroom（MCP 工具）](https://github.com/headroomlabs-ai/headroom) — *GitHub · Python · +4.3k/周 · 总 79.6k star*
  - 📌 **是什么**：面向 LLM 上下文成本的压缩层，可在工具输出、日志、文件和 RAG 片段进入模型前做裁剪，并提供库、代理或 MCP server 形态。
  - 💡 **学习点**：学习上下文工程：如何保留关键信息同时减少 token 消耗。
  - 🧭 **上手**：选一段工具日志或 JSON，对比压缩前后模型回答与 prompt 体积。
- [CopilotKit/CopilotKit（Agent 框架）](https://github.com/CopilotKit/CopilotKit) — *GitHub · TypeScript · 周增量统计中 · 总 38.7k star*
  - 📌 **是什么**：面向 Agent 与生成式 UI 的前端框架，把 React、移动端、Slack 等界面接入智能体能力。
  - 💡 **学习点**：前端工程师可学习 Agent 事件流、工具调用状态与生成式 UI 的渲染方式。
  - 🧭 **上手**：打开示例项目，观察前端组件如何订阅 Agent 事件并把返回内容渲染为界面。
- [alibaba/open-code-review（开发工具）](https://github.com/alibaba/open-code-review) — *GitHub · Go · +3.7k/周 · 总 50.6k star*
  - 📌 **是什么**：结合确定性审查流水线与 LLM Agent 的代码审查工具，强调多语言规则、行级评论和仓库级上下文。
  - 💡 **学习点**：学习把静态规则、上下文定位和模型推理接入工程化质量检查流程。
  - 🧭 **上手**：在一个小型 PR 示例上运行审查，重点比较规则输出与 LLM 生成评论的差异。
- [patchy631/ai-engineering-hub](https://github.com/patchy631/ai-engineering-hub) — *GitHub · Jupyter Notebook · +3.5k/周 · 总 41.8k star*
  - 📌 **是什么**：覆盖 LLM、RAG 与 AI Agent 应用教程的学习型开源仓库。
  - 💡 **学习点**：适合从可复现的教程样例理解 LLM 应用构建与工具链组合。
  - 🧭 **上手**：挑一个 RAG 或 Agent 教程示例，逐步运行并记录每个节点的输入输出。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue双向watch防递归
- **技能点**：掌握状态与派生数据同步时设置值比较守卫，避免对象/数组引用变化触发无限递归。
- **坑点**：折叠标志和面板尺寸数组互相 watch，写入新数组后 deep watch 回写标志，持续唤醒对方。
- **解决方案**：写入前逐项比较，值不变就不写；回写标志前比较当前值与推导值，切断同步环。
```text
watch([leftFlag, rightFlag], () => {
  const next = [left, "auto", right];
  if (next.every((v, i) => v === sizes.value[i])) return;
  sizes.value = next;
});

watch(sizes, (v) => {
  const derived = v[0] === 0;
  if (leftFlag.value !== derived) leftFlag.value = derived;
}, { deep: true });
```
- **拓展**：可抽象 useSyncedValue/useGuardedWatch 工具，统一处理同步、比较与防抖。
- *来源：admin-workspace-new/test | 2026-08-11*

### 2. 事件冒泡修复边界
- **技能点**：巩固事件流定位与业务边界修复意识：共享组件保持通用，冲突在调用处拦截。
- **坑点**：复制按钮点击冒泡到外层卡片点击处理器，误触发解析区展开/收起。
- **解决方案**：在业务组件调用处加 @click.stop，避免把业务交互逻辑写进共享组件。
```text
<CopyText :text="题目编号: xxx" :copyText="data.questionId" @click.stop />
```
- **拓展**：可沉淀 UI 事件冲突清单：stop、prevent、pointer-events、target 判断、父级动作隔离。
- *来源：admin-workspace-new/test | 2026-08-11*

### 3. 工具栏API常量与类型
- **技能点**：掌握组件配置可发现性设计：用常量暴露合法项，用联合类型约束调用。
- **坑点**：业务方不知道 toolbar-order 可传哪些名称，内置按钮仍需通过 customButtons 才能生效。
- **解决方案**：导出 TOOLBAR_BUTTONS 常量和类型，并让默认 handler 无条件注册，减少隐式依赖。
```text
export const TOOLBAR_BUTTONS = {
  undo: 'undo',
  formula: 'formula',
  image: 'image'
} as const;

export type ToolbarButtonName = typeof TOOLBAR_BUTTONS[keyof typeof TOOLBAR_BUTTONS];
```
- **拓展**：可进一步生成 IDE 提示、校验枚举与输出组件文档表格。
- *来源：admin-workspace-new | 2026-08-11*

### 4. 单一状态源去同步
- **技能点**：提升状态建模能力：区分用户意图、显示条件与派生尺寸，避免多状态互相同步。
- **坑点**：持久化标志、折叠状态、尺寸数组和显隐条件混在一起，形成隐式依赖与循环风险。
- **解决方案**：删除尺寸持久化与折叠标志双向同步，只保留拖拽状态，用 v-if 控制面板显隐。
```text
// before: flag -> sizes -> flag
// after:
// <template v-if="isAiReviewer" #panel-2>...</template>
localPanelSizes.value = [760, "auto", 400];
```
- **拓展**：复杂 UI 先画状态归属表：谁产生、谁消费、谁派生，再写代码。
- *来源：admin-workspace-new/test | 2026-08-11*

### 5. 跨副本分支事实校验
- **技能点**：巩固多仓库/多分支环境下的事实校验习惯：先确认代码落盘、部署分支和文件存在，再排查现象。
- **坑点**：同项目多副本停在不同分支/commit，本地已改但测试环境仍跑旧代码，造成“线上现象与本地代码不一致”。
- **解决方案**：引用排坑结论前检查目标副本文件是否存在；用 git show master:<file> 对比部署分支真实代码。
```text
git -C <repo> branch --show-current
git -C <repo> status --short
git -C <repo> show master:path/to/File.vue
```
- **拓展**：可沉淀标准检查流：branch、status、diff、关键文件存在性、构建产物来源。
- *来源：admin-workspace-hr-talent | 2026-09-24*

### 6. 顺序编辑与落盘校验
- **技能点**：掌握安全改码流程：改动前检索引用，同文件编辑串行应用，完成后以磁盘和 lint 验证。
- **坑点**：并行编辑同一文件可能基于同一快照互相覆盖；会话切换后部分编辑未真正落盘，磁盘仍是旧版。
- **解决方案**：同文件多处修改逐条顺序执行；修改后 grep 关键符号并重跑 lint，确认真实生效。
```text
// same-file edits: sequential, not parallel
grep -n "TOOLBAR_BUTTONS" src/components/QuillEditorNew/toolbar/toolbarButtons.ts
npm run lint
```
- **拓展**：可固化为 agent 规则：编辑后校验 diff、mtime、目标字符串存在和 lint 结果。
- *来源：admin-workspace-new | 2026-08-11; admin-workspace-hr-talent | 2026-09-08*

