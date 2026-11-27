---
title: "工作台日报 · 2026-11-28"
date: 2026-11-28 07:03:07
categories: [工作日记]
tags: ["日报", "前端", "大模型", "网络安全", "AI"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-28

## 🔥 行业热点

- [Google Will Make AI Agents Use Its MCP Servers. The Gmail One Can't Send Email](https://techstackups.com/articles/google-workspace-ai-agents-mcp/) — *Hacker News*
  - 📌 **内容**：Google 正推动 AI Agent 接入其 MCP 服务器，但 Gmail 相关工具似乎不能直接发送邮件，反映出平台对 Agent 能力边界和权限控制的考量。
  - 💡 **学习**：可以学习 MCP 协议在 Agent 工具调用中的集成方式，以及平台 API 权限、安全策略与能力限制的设计。
  - 🧭 **拓展**：可尝试构建一个 MCP 客户端，测试 Gmail 等工具在不同权限下的实际可用能力。
- [AI made self-hosting fun again](https://janlukas.blog/posts/ai-self-hosting) — *Hacker News*
  - 📌 **内容**：AI 工具和本地模型让自托管重新变得有趣，可能降低了个人部署、自动化和私有服务搭建的门槛。
  - 💡 **学习**：适合了解 Docker、HomeLab、本地模型部署与个人自动化服务的组合玩法。
  - 🧭 **拓展**：可尝试用容器化方式部署一个本地 AI 助手或私人知识库。
- [US Air Force's cloud computing platform hacked](https://news.sky.com/story/us-air-forces-cloud-computing-platform-hacked-13617163) — *Hacker News*
  - 📌 **内容**：美国空军云计算平台被黑，提示关键云基础设施在身份认证、访问控制、供应链和日志审计方面存在持续安全风险。
  - 💡 **学习**：程序员可关注云安全基线、IAM 权限最小化、零信任架构和异常行为监控。
  - 🧭 **拓展**：可用云安全配置扫描工具检查自己项目中的错误暴露和权限配置。
- [The need for a formally verifiable mainstream programming language](https://blainehansen.me/post/mainstream-verifiable-language/) — *Hacker News*
  - 📌 **内容**：该话题讨论主流编程语言为何需要更强的形式化验证能力，以提升关键软件系统的可靠性和可证明正确性。
  - 💡 **学习**：可以理解类型系统、形式化证明、语言设计与高可靠软件开发之间的关系。
  - 🧭 **拓展**：可进一步了解 Rust、Ada SPARK、Lean 等语言或工具在形式化验证上的实践。
- [Prospero: Bloat-free, native AI coding harness with ImGui](https://github.com/skeeto/prospero) — *Hacker News*
  - 📌 **内容**：Prospero 是一个基于 ImGui 的原生 AI 编码工具，强调轻量、无臃肿和本地化工作流。
  - 💡 **学习**：可以学习如何用 ImGui 构建桌面开发工具，以及如何集成 AI 编码能力。
  - 🧭 **拓展**：可试用或参考其项目结构，探索轻量级 AI 编码工作流。
- [SessionGraph: Visualize all your coding agent sessions in one interactive graph](https://sessiongraph.ai/) — *Hacker News*
  - 📌 **内容**：SessionGraph 将多个编码 Agent 会话汇总成一个可交互图，帮助开发者理解任务分支、上下文关系和会话历史。
  - 💡 **学习**：可以学习 Agent 日志解析、会话建模、图数据结构和交互式可视化实现。
  - 🧭 **拓展**：可尝试接入 Claude Code、Cursor 或本地编码 Agent 的会话数据做可视化。
- [YouTube Ad Punisher – Auto-reloads YouTube pages when an ad plays](https://github.com/antonpictures/youtube-punisher) — *Hacker News*
  - 📌 **内容**：该工具在 YouTube 广告播放时自动刷新页面，是一种通过浏览器自动化对抗广告体验的脚本式方案。
  - 💡 **学习**：可以学习油猴脚本、DOM 事件监听、页面状态检测和浏览器自动化技巧。
  - 🧭 **拓展**：可尝试写一个浏览器扩展或用户脚本，测试广告检测和自动重载策略。
- [Claude sabotaging ML research tasks?](https://twitter.com/RealAstropulse/status/2126119408997466389) — *Hacker News*
  - 📌 **内容**：该讨论关注 Claude 在 ML 研究任务中是否会出现偏离目标或“破坏任务”的行为，涉及 Agent 对齐和评测可信度。
  - 💡 **学习**：可以学习大模型行为评估、任务基准设计、提示工程和安全护栏。
  - 🧭 **拓展**：可设计受控实验，观察模型在复杂任务中是否会偏离用户目标。
- [AMD-specific fork of llama.cpp](https://devnull.land/amd-specific-llama-cpp) — *Hacker News*
  - 📌 **内容**：这是 llama.cpp 的 AMD 专用分支，可能面向 AMD GPU 或特定平台做本地大模型推理优化。
  - 💡 **学习**：可以学习本地大模型推理、AMD 硬件适配、ROCm/Vulkan 等后端以及量化优化思路。
  - 🧭 **拓展**：可在 AMD 显卡上与原版 llama.cpp 做吞吐、延迟和显存占用对比。
- [Credit card skimming scripts using WebRTC found on popular shopping sites](https://www.infohawk.com/news/2026-11-27-black-friday-webrtc-skimmer) — *Hacker News*
  - 📌 **内容**：恶意脚本被发现利用 WebRTC 在购物网站窃取信用卡信息，说明前端第三方脚本和实时 API 正成为攻击面。
  - 💡 **学习**：可以学习 Web 前端安全、CSP、子资源完整性、第三方脚本审计和 WebRTC 风险。
  - 🧭 **拓展**：可对自身网站做第三方脚本清单和 CSP 策略检查。

## 🌟 GitHub 热门开源项目

- [Snailclimb/JavaGuide](https://github.com/Snailclimb/JavaGuide) — *GitHub · JavaScript · +392/周 · 总 159.7k star*
  - 📌 **是什么**：面向 Java、后端与 AI 应用开发的面试与知识体系指南，覆盖计算机基础、数据库、分布式与系统设计。
  - 💡 **学习点**：帮助前端工程师补齐后端与 AI 工程常见面试语境，理解数据库、缓存、高并发和系统设计如何支撑 LLM 应用。
  - 🧭 **上手**：从 README 进入，选择数据库、系统设计和 AI 应用开发章节，整理成自己的问答卡片。
- [HKUDS/DeepTutor（Agent 框架）](https://github.com/HKUDS/DeepTutor) — *GitHub · Python · +18.0k/周 · 总 63.8k star*
  - 📌 **是什么**：面向长期个性化辅导的 AI 教学系统，结合多智能体、RAG 和交互式学习流程。
  - 💡 **学习点**：可学习如何把检索增强、学习记忆和对话式辅导组织成可持续演进的 Agent 应用。
  - 🧭 **上手**：先跑通官方示例，再观察一次问答中检索资料、上下文组织和回复生成如何串联。
- [Graphify-Labs/graphify（Agent Skills）](https://github.com/Graphify-Labs/graphify) — *GitHub · Python · +17.3k/周 · 总 154.3k star*
  - 📌 **是什么**：将代码库、文档、SQL 结构、配置和 PDF 转换为可查询知识图谱的 Agent 技能，服务 Claude Code、Cursor 等编码工具。
  - 💡 **学习点**：适合前端工程师学习 AST、代码结构提取和 GraphRAG 思路，让 Agent 更准确地理解项目。
  - 🧭 **上手**：在一个小型前端仓库中调用 /graphify，尝试询问组件依赖、接口来源或文档位置。
- [heygen-com/hyperframes（Agent Skills）](https://github.com/heygen-com/hyperframes) — *GitHub · TypeScript · +15.5k/周 · 总 91.5k star*
  - 📌 **是什么**：开源 AI 视频编辑工具，支持用 HTML 描述视频内容并渲染为 MP4，也可通过 Claude Code、Codex 等方式创作。
  - 💡 **学习点**：前端可学习把 DOM、动画和时间轴思维迁移到视频生成，理解 HTML-to-video 渲染管线。
  - 🧭 **上手**：运行一个最小 HTML 到 MP4 示例，然后修改布局、动画或文案并比较成片差异。
- [blader/humanizer（Agent Skills）](https://github.com/blader/humanizer) — *GitHub · Python · +12.8k/周 · 总 74.3k star*
  - 📌 **是什么**：用于降低文本 AI 生成痕迹的 Agent 技能，通过改写策略让输出更自然。
  - 💡 **学习点**：可学习提示词设计、风格迁移和后处理策略，理解 LLM 输出的语气与结构控制。
  - 🧭 **上手**：拿一段明显的 AI 文本运行该技能，对比改写前后的句式、口吻和信息密度。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue 双向 watch 守卫
- **技能点**：掌握 Vue 多状态双向同步时防止递归更新的守卫设计能力。
- **坑点**：折叠标志、面板尺寸数组和 deep watch 互相回写，新数组引用即使值未变也会触发循环，最终报 Maximum recursive updates。
- **解决方案**：每次写状态前先做值比较，相等不写；回写标志前比较当前值与推导值。更彻底是保持单一数据源，减少双向同步。
```text
if (next.some((v, i) => v !== localPanelSizes.value[i])) localPanelSizes.value = next;
if (folded.value !== nextFolded) folded.value = nextFolded;
```
- **拓展**：可封装 useSyncedState 或 computed + emit 模式，统一处理同步守卫。
- *来源：admin-workspace-new 2026-08-11*

### 2. 组件工具栏配置类型化
- **技能点**：掌握通过常量与联合类型暴露组件可配置项，降低业务接入成本。
- **坑点**：业务侧不知道工具栏按钮名能传什么，部分能力还强制依赖 customButtons，造成重复配置和误用。
- **解决方案**：导出 TOOLBAR_BUTTONS 常量和 ToolbarButtonName 类型，并让默认按钮内置注册，业务只通过 :toolbar-order 选用。
```text
export const TOOLBAR_BUTTONS = { bold: 'bold', formula: 'formula' } as const;
export type ToolbarButtonName = keyof typeof TOOLBAR_BUTTONS;
```
- **拓展**：可继续为按钮提供元数据：图标、标题、默认行为和禁用规则。
- *来源：admin-workspace-new 2026-08-11*

### 3. 事件冒泡与组件边界
- **技能点**：掌握在复合交互中划分通用组件与业务组件的事件责任边界。
- **坑点**：复制按钮点击冒泡到题目卡片外层，误触发解析展开/收起；若在共享组件内 stop，会污染通用行为。
- **解决方案**：保持 CopyText 通用，不内置 stopPropagation，在业务调用处对特定场景使用 @click.stop 阻断冒泡。
```text
<CopyText :copyText='data.questionId' @click.stop />
```
- **拓展**：可为可交互子组件约定默认不冒泡的业务事件或 emit 明确动作。
- *来源：admin-workspace-new 2026-08-11*

### 4. 插槽透传扩展组件定制
- **技能点**：掌握通用组件通过插槽开放定制点，避免业务逻辑侵入核心实现。
- **坑点**：为满足折叠图标定制直接改组件内部结构，会让通用组件承载业务差异，后续复用困难。
- **解决方案**：组件用 $slots 检测并透传插槽，隐藏默认图标；业务侧注入自定义图标，用样式表达展开/收起状态。
```text
<slot v-if="$slots['start-collapsible']" name="start-collapsible" />
```
- **拓展**：沉淀组件扩展点清单：插槽、props、事件，明确哪些允许业务覆盖。
- *来源：admin-workspace-new 2026-08-10*

### 5. 请求实例与路径前缀
- **技能点**：掌握根据请求实例的 baseURL 决定接口路径写法，避免网关前缀丢失。
- **坑点**：不同封装实例的 baseURL 不同，有的已含业务网关前缀，有的不含；按文档照抄容易 404 或路径错配。
- **解决方案**：写 service 前先看使用的是哪个 http 实例，若 baseURL 不含网关前缀则路径写全 /teach、/admin 等标识。
```text
// baseURL 仅 /api 时，路径写全网关前缀
http.get('/teach/question/list')
```
- **拓展**：可在 service 层用环境/实例说明注释或封装路径生成器降低出错概率。
- *来源：admin-workspace-new*

### 6. 格式化代码合并主干
- **技能点**：掌握在已有全量格式化的分支上合并主干时，控制冲突范围的工程策略。
- **坑点**：用本地镜像格式化分支先格式化主干再合并，会改变合并基线，把单侧修改变成双侧修改，显著放大冲突。
- **解决方案**：直接 merge 主干，冲突处优先采纳主干基线，再统一执行格式化，并按清单重放语义改动。
```text
git merge master
npm run format
```
- **拓展**：可将格式化拆成独立提交或 CI 检查，避免与功能修改混在一起。
- *来源：admin-workspace-hr 2026-10-14*

