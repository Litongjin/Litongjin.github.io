---
title: "工作台日报 · 2026-09-25"
date: 2026-09-25 07:19:37
categories: [工作日记]
tags: ["日报", "AI Agent", "AI安全", "AI编程", "AI视频"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-25

## 🔥 行业热点

- [Early rogue AI agent activity and attempts to hack found on urlquery.net](https://transluce.org/agent-activity) — *Hacker News*
  - 📌 **内容**：报道在 urlquery.net 上观察到早期恶意 AI Agent 活动与黑客尝试，提示 AI 驱动的攻击已开始出现在真实网络基础设施中。
  - 💡 **学习**：可关注恶意 Agent 的行为特征与检测规则，在安全监控中增设对自动化 AI 流量的识别能力。
  - 🧭 **拓展**：可在自己的蜜罐或日志平台模拟类似异常访问，验证检测规则。
- [SkillOpt: Training Loop for Agent Skills](https://microsoft.github.io/SkillOpt/) — *Hacker News*
  - 📌 **内容**：介绍 SkillOpt，一种面向 Agent 技能的训练循环机制，让智能体通过迭代优化掌握更稳定的执行能力。
  - 💡 **学习**：学习如何把 Agent 行为拆解为可训练技能，并用闭环反馈持续优化 Prompt 或策略。
  - 🧭 **拓展**：可尝试在 Agent 框架中实现一个小型技能回放与优化循环，对比训练前后效果。
- [Rails World 2026 Opening Keynote [video]](https://www.youtube.com/watch?v=vDjW_dRyKXY) — *Hacker News*
  - 📌 **内容**：发布 Rails World 2026 开幕主题演讲视频，涵盖 Rails 社区最新方向与框架进展。
  - 💡 **学习**：可借此了解 Rails 生态的新特性、设计哲学与 Web 开发趋势。
  - 🧭 **拓展**：观看后挑选新特性在个人项目里做一次升级体验。
- [Opus 5.5 is good at explainer videos](https://launchvideo.io) — *Hacker News*
  - 📌 **内容**：评测认为 Opus 5.5 在制作讲解类视频方面表现优秀，说明新一代视频生成模型在知识传递场景更可用。
  - 💡 **学习**：可研究如何用视频生成模型快速产出教程或说明内容，并设计提示词控制节奏与视觉逻辑。
  - 🧭 **拓展**：用同主题内容对比 Opus 5.5 与其它视频模型的成片效果。
- [Security auditing in the age of (good enough) AI](https://blog.trailofbits.com/2026/09/18/auditing-in-the-age-of-good-enough-ai/) — *Hacker News*
  - 📌 **内容**：讨论在 AI 已“足够好”的时代如何进行安全审计，强调人机协作和验证流程的必要性。
  - 💡 **学习**：可以学习把 AI 生成的安全建议纳入现有审计管线，通过人工复核和自动化测试降低误报。
  - 🧭 **拓展**：可让 AI 对一份安全审计报告做初筛、人工把关，评估效率与质量。
- [Show HN: AgentRun: DSL to turn agents into workflows](https://github.com/Parcha-ai/agentrun) — *Hacker News*
  - 📌 **内容**：展示 AgentRun，一个用 DSL 把 AI Agent 编排成工作流的工具，提升复杂 Agent 任务的可控性。
  - 💡 **学习**：可以学习 DSL 设计在 Agent 流程编排中的优势，比如显式状态、分支与复用。
  - 🧭 **拓展**：尝试用 AgentRun 重构一个原有的多步 Prompt 调用流程。
- [Show HN: Air-gapped file encryption as self-decrypting HTML page](https://cms-sfx-demo.apeleg.com/) — *Hacker News*
  - 📌 **内容**：展示一种离线文件加密方案，生成自解密 HTML 页面，让文件在隔离环境中加密和解密。
  - 💡 **学习**：可了解纯前端密码学实现的边界，特别是密钥管理、流加密和浏览器安全模型。
  - 🧭 **拓展**：可对生成页面做代码审计，验证其加密算法与密钥派生实现。
- [A Million Agents Is a Distributed System Problem](https://www.instacloud.com/blogs/a-million-agents-is-a-distributed-systems-problem) — *Hacker News*
  - 📌 **内容**：文章指出当 Agent 规模达到百万级时，核心挑战变成分布式系统问题，而非单个模型能力。
  - 💡 **学习**：需要掌握任务调度、通信、一致性与故障恢复等分布式系统技术来支撑大规模 Agent。
  - 🧭 **拓展**：可结合消息队列或 Actor 模型设计一个 Agent 集群的架构草图。
- [Show HN: Critic – Review code with the agent that wrote it](https://www.critic.run/) — *Hacker News*
  - 📌 **内容**：展示 Critic 工具，让编写代码的同一 Agent 参与代码审查，形成生成-批评-修改闭环。
  - 💡 **学习**：可以借鉴“让模型自评+多轮修订”的流程，提高生成代码的正确性和可维护性。
  - 🧭 **拓展**：可把 Critic 接入 CI，在 PR 中自动生成审查意见。
- [F-Droid 2.0](https://f-droid.org/2026/09/24/f-droid-2.0-a-new-chapter-for-android-freedom.html) — *Hacker News*
  - 📌 **内容**：F-Droid 2.0 发布，开源 Android 应用商店带来新的版本与功能改进。
  - 💡 **学习**：可关注开源分发渠道的更新机制、签名策略与用户隐私保护方式。
  - 🧭 **拓展**：尝试在模拟器上安装体验 F-Droid 2.0，并对比其应用发现流程。

## 🌟 GitHub 热门开源项目

- [thedotmack/claude-mem（Agent Skills）](https://github.com/thedotmack/claude-mem) — *GitHub · TypeScript · +965/周 · 总 94.6k star*
  - 📌 **是什么**：为Claude等agent提供跨会话持久记忆的工具，捕捉会话内容并用AI压缩后注入未来上下文。
  - 💡 **学习点**：学习如何设计基于AI压缩与检索的长期记忆层。
  - 🧭 **上手**：阅读源码中会话捕获与压缩的核心模块，或运行一个对话示例观察记忆如何注入。
- [langchain-ai/langchain（Agent 框架）](https://github.com/langchain-ai/langchain) — *GitHub · Python · +901/周 · 总 147.0k star*
  - 📌 **是什么**：一个用于构建agent工程的开源平台，提供模块化组件与工作流编排能力。
  - 💡 **学习点**：掌握LangChain的抽象概念（工具调用、会话状态、智能体编排）是构建LLM应用的基础。
  - 🧭 **上手**：在官方文档跑一个快速开始的链或智能体示例，理解其核心抽象。
- [mem0ai/mem0（Agent 框架）](https://github.com/mem0ai/mem0) — *GitHub · Python · +842/周 · 总 66.0k star*
  - 📌 **是什么**：为AI agent提供可落地的持久记忆基础设施，可直接嵌入应用生产环境。
  - 💡 **学习点**：学习生产级记忆层设计：如何存储、检索和更新agent的长期记忆。
  - 🧭 **上手**：运行其快速开始示例，观察记忆如何随对话累积和检索。
- [langchain-ai/langgraph（Agent 框架）](https://github.com/langchain-ai/langgraph) — *GitHub · Python · +783/周 · 总 42.2k star*
  - 📌 **是什么**：一个用于构建可恢复、状态化agent的框架，强调工作流控制与容错。
  - 💡 **学习点**：理解状态图模型如何编排agent步骤和工具调用。
  - 🧭 **上手**：对照其状态图示例，亲手实现一个带循环与分支的agent。
- [infiniflow/ragflow（Agent 框架）](https://github.com/infiniflow/ragflow) — *GitHub · Go · +760/周 · 总 91.3k star*
  - 📌 **是什么**：开源RAG引擎，融合检索增强生成与agent能力，为LLM构建上下文层。
  - 💡 **学习点**：学习RAG与agent如何结合，以及如何设计多阶段检索流程。
  - 🧭 **上手**：运行其自带的网页问答示例，观察检索与生成如何协同。

## 🚀 技能提升点（工作总结汇总）

### 1. Quill blot 值读取契约
- **技能点**：掌握 Quill/Parchment 的静态契约：blot 真实值必须通过 `blot.statics.value(domNode)` 获取；Delta/Op 等类型用 `Quill.import('delta')` 拿运行时的类，而非 ES import。
- **坑点**：用实例 `blot.value()` 取到的是 `{ [blotName]: value }` 包装，弹窗回显变成 `[object Object]`；`import { Delta } from 'quill'` 在 Vite 预构建后不是构造函数，`new Delta()` 抛 TypeError；embed blot 的 create() 收到 `{blotName: value}` 而非常量，未守卫会写入脏数据。
- **解决方案**：取真值统一走静态契约 `statics.value(domNode)`；运行时类型一律 `Quill.import('delta')`；所有 create() 入口加 `typeof value === 'string' ? value : (value?.[name] ?? '')` 守卫。
```text
// 取真值走静态契约
const raw = blot.statics.value(blot.domNode);
// Delta 别用 ES import
const Delta = Quill.import('delta');
// create() 守卫
const v = typeof value === 'string' ? value : (value?.[name] ?? '');
```
- **拓展**：排查「框架运行时与构建产物不一致」的坑时，先确认 import 的对象是不是真的构造函数。
- *来源：admin-workspace-new*

### 2. Upload customRequest 回调参数
- **技能点**：掌握 antd Upload 的 customRequest 中 `onSuccess` 参数会被原样写入 `file.response` 的透传机制，能正确设计自定义上传链路。
- **坑点**：把上传结果包一层 file-like 对象再传给 onSuccess，导致 file.response 多套一层，后续取 key/url/type/control 全部落空，fileName 退化成本地文件名、预览下载失效。
- **解决方案**：onSuccess 直接透传上传接口原始返回值；依赖 file.response 的代码先查 antd/es/upload/Upload.js 确认参数流再决定是否包装。
```text
customRequest: async ({ file, onSuccess }) => {
  const result = await uploadV2(file, UploadScene.X);
  onSuccess(result); // 原始结果直传，勿包一层
}
// 消费端：file.response.key / file.response.url 直接可用
```
- **拓展**：所有「回调参数会被框架透传」的场景，都应先查框架源码确认参数流。
- *来源：admin-workspace-hr-talent*

### 3. 权限过滤边界值防护
- **技能点**：掌握写权限/配置过滤函数时对缺省值的防御：非 string 入参短路返回可见，避免拖垮整个列表渲染。
- **坑点**：`list.filter(btn => authTag(btn.authKey))` 只要有按钮没配 authKey（undefined），authTag 就走 `.every()` 分支抛 TypeError，整个 computed 崩溃、按钮区全部消失。
- **解决方案**：过滤前缀加 `!btn.authKey ||` 短路；过滤函数对非 string 入参返回默认可见；配置完整性校验前置。
```text
// 崩：authTag(undefined) 抛 TypeError
const list = buttons.filter(btn => authTag(btn.authKey));
// 修：缺省 key 直接放行
const list = buttons.filter(btn => !btn.authKey || authTag(btn.authKey));
```
- **拓展**：配置项驱动渲染的过滤逻辑先验证配置完整性，再进入判断逻辑。
- *来源：admin-workspace-test*

### 4. flex 滚动区 minHeight 陷阱
- **技能点**：掌握 flex column 容器内「仅列表区域滚动」的标准布局：根容器 height:100% + flex column + overflow:hidden；滚动区 flex:1 + minHeight:0 + overflow:auto；非滚动区 flexShrink:0。
- **坑点**：漏写 minHeight:0 时 flex 子项 min-height:auto 会撑破容器导致滚动失效；根容器用 minHeight:'100%' 而非 height:'100%' 会露灰底；祖先若是 content-box 带 padding，height:100% 还会被额外裁剪。
- **解决方案**：统一该布局模式：分页条放滚动容器外并 flexShrink:0；表头吸顶按有无 scroll.x 二选一（有则 sticky prop，无则 CSS th{position:sticky, top:0}）。
```text
<div style={{ height: '100%', display: 'flex', flexDirection: 'column', overflow: 'hidden' }}>
  <header style={{ flexShrink: 0 }}>标题/筛选</header>
  <main style={{ flex: 1, minHeight: 0, overflow: 'auto' }}>表格</main>
  <footer style={{ flexShrink: 0 }}>分页条</footer>
</div>
```
- **拓展**：沉淀为页面布局模板/组件，新页面直接套用避免重复踩坑。
- *来源：admin-workspace-hr-talent*

### 5. 聚合接口空值归一化
- **技能点**：掌握聚合/统计类接口在 service 层统一做空值归一的契约意识：分页接口 records || []、全量接口 ?? []，UI 不重复防御。
- **坑点**：后端无数据时 code=0 但不下发 data 字段，service 返回 undefined；UI 里 rows.reduce() 直接崩整页白屏（实测：报表选未来月份再切换 tab 触发）。
- **解决方案**：service 层兜底空数组；率值分母为 0 返回 null，渲染先判目标值再显 '-'；约定「UI 只消费非空数组」写进团队规范。
```text
// service 层兜底（分页）
return { records: data?.records ?? [], total: data?.total ?? 0 };
// service 层兜底（全量）
return data ?? [];
```
- **拓展**：把「后端欠字段由 service 兜底、UI 只消费契约」作为分层约定推广到所有接口。
- *来源：admin-workspace-hr-talent*

### 6. 会话内换账号权限残留
- **技能点**：掌握内存态权限缓存在「会话内换账号」场景的重置时机：清外部存储不够，还要清状态层（pinia/vuex）的授权对象。
- **坑点**：authSet() 纯累加、从不重置 authPage/authTag，logout 只清 DataStore.map 不清 pinia；同一标签页换账号时新账号继承旧账号按钮/列级授权，路由级有重拉 menuPath 拦住、按钮级漏光。
- **解决方案**：authSet() 开头重置两个授权对象；logout/登录拉权限前清空内存缓存；涉及列级权限的表格组件加数据源守卫。
```text
function authSet(pages, tags) {
  authPage = {}; // 先清空再累加
  authTag  = {};
  Object.assign(authPage, pages);
  Object.assign(authTag, tags);
}
```
- **拓展**：排查权限问题先定位是路由级还是按钮级缓存，再决定从哪一层清。
- *来源：admin-workspace-test*

