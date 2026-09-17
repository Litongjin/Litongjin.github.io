---
title: "工作台日报 · 2026-09-18"
date: 2026-09-18 07:02:44
categories: [工作日记]
tags: ["日报", "AI Agent", "大模型", "开发工具", "开源项目"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-18

## 🔥 行业热点

- [Launch HN: Skillsync (YC W26) – AI chat sessions made portable across agents](https://news.ycombinator.com/item?id=49743049) — *Hacker News*
  - 📌 **内容**：面向 AI Agent 的会话可移植方案，试图让同一段对话与上下文能在不同 Agent 之间迁移复用。
  - 💡 **学习**：理解 Agent 记忆/上下文的可移植性问题，以及跨 Agent 的上下文与格式标准化思路。
  - 🧭 **拓展**：可动手尝试把某个 Agent 的会话导出为通用结构，再导入另一客户端验证兼容性。
- [Show HN: Craigslist for agent skills, curated by a human](https://skillbay.sh/) — *Hacker News*
  - 📌 **内容**：以类似分类广告的形式、由人工策展的 Agent 技能市场，聚合可复用的 Agent 能力。
  - 💡 **学习**：了解 Agent 技能（工具/插件）如何被发现、描述与复用，以及技能注册表的设计。
  - 🧭 **拓展**：可浏览同类技能仓库，并把其中一个技能接入自己的 Agent 做端到端试验。
- [Bend – A language that blocks AI mistakes via proof, on CPU and GPU](https://bend-lang.com/) — *Hacker News*
  - 📌 **内容**：一门通过证明/形式化手段减少 AI 生成代码错误、并同时支持 CPU 与 GPU 执行的语言。
  - 💡 **学习**：形式化验证与并行计算结合的语言设计思路，以及如何用类型/证明约束程序行为。
  - 🧭 **拓展**：跑一遍官方示例，把同一任务与 Python/CUDA 实现做对比，观察验证成本。
- [Bonsai 2 27B: Near-Lossless Compression in a 9x Smaller Footprint](https://prismml.com/news/bonsai-2-27b) — *Hacker News*
  - 📌 **内容**：一个 27B 规模模型通过压缩技术大幅缩小体积，同时保持接近无损的效果，指向模型压缩方向的新进展。
  - 💡 **学习**：关注量化、蒸馏、低比特压缩等如何在部署成本与模型质量之间做权衡。
  - 🧭 **拓展**：在本地推理框架中对比压缩前后的显存占用与输出质量。
- [TSMC revealing details about next gen A14 node](https://iedm26.mapyourshow.com/8_0/sessions/session-details.cfm?scheduleid=331) — *Hacker News*
  - 📌 **内容**：台积电披露下一代 A14 制程节点细节，涉及晶体管密度与能效演进路线。
  - 💡 **学习**：了解先进制程节点演进对芯片性能、功耗与成本的影响逻辑。
  - 🧭 **拓展**：对照官方技术路线图与后续量产产品的实测数据来验证宣传指标。
- [How Uber Protects Against Retry Storms](https://www.uber.com/us/en/blog/protecting-against-retry-storms/) — *Hacker News*
  - 📌 **内容**：Uber 分享在大规模微服务中防止重试风暴的稳定性实践，涉及限流、退避与熔断等手段。
  - 💡 **学习**：重试放大效应、指数退避加抖动、幂等设计与熔断降级的工程经验。
  - 🧭 **拓展**：在自己的服务里注入故障做压测，观察重试放大并调整退避与限流策略。
- [Fujitsu launches made-in-Japan next-generation CPU FUJITSU-MONAKA](https://global.fujitsu/en-global/pr/news/2026/09/14-02) — *Hacker News*
  - 📌 **内容**：富士通发布日本本土自研的下一代数据中心 CPU，主打能效与云/AI 场景。
  - 💡 **学习**：了解非 x86（Arm 系）服务器 CPU 的架构选择与软件生态迁移考量。
  - 🧭 **拓展**：关注其性能功耗比基准与主流框架的适配进展，评估可迁移性。
- [Hister: A private search engine for the pages you visit and the files you keep](https://github.com/asciimoo/hister) — *Hacker News*
  - 📌 **内容**：一个隐私优先的本地搜索引擎，为浏览过的网页与本地文件建立可检索索引。
  - 💡 **学习**：本地内容索引、全文检索与隐私保护优先的架构设计。
  - 🧭 **拓展**：自建类似方案（本地全文索引或向量检索）并与现有工具做效果对比。
- [How GLM built its own inference infrastructure](https://z.ai/blog/glm-built-its-inference-infrastructure) — *Hacker News*
  - 📌 **内容**：GLM 团队分享自建大模型推理基础设施的实践，涉及吞吐、延迟与成本之间的权衡。
  - 💡 **学习**：推理服务中的批处理、KV Cache、并行与请求调度优化思路。
  - 🧭 **拓展**：用 vLLM/SGLang 等框架做压测复现，绘制吞吐-延迟曲线并调参。
- [One year of sponsored Servo development](https://servo.org/blog/2026/09/15/one-year-of-sponsorship/) — *Hacker News*
  - 📌 **内容**：Servo 浏览器引擎在赞助开发一年后的进展回顾，涉及并行渲染与嵌入能力。
  - 💡 **学习**：浏览器引擎架构与 Rust 在系统级软件中的应用实践。
  - 🧭 **拓展**：编译并运行 Servo 的嵌入示例，了解其在嵌入式渲染场景中的可用性。

## 🌟 GitHub 热门开源项目

- [tt-a1i/archify（Agent Skills）](https://github.com/tt-a1i/archify) — *GitHub · JavaScript · +7.8k/周 · 总 65.8k star*
  - 📌 **是什么**：一个 Agent Skill，让编码代理生成自包含 HTML 的架构图、工作流图、时序图、数据流图与生命周期图，带动效并支持清晰导出。
  - 💡 **学习点**：可以学到如何把「可视化输出」封装成代理可调用的技能，而不是让模型在对话里用 Mermaid 凑合画图。
  - 🧭 **上手**：先读仓库里 SKILL.md 一类的技能描述文件，跑一次它给出的示例提示词，观察生成的 HTML 与默认 Mermaid 输出在可读性上的差别。
- [reactive-resume/reactive-resume（Agent Skills）](https://github.com/reactive-resume/reactive-resume) — *GitHub · TypeScript · 周增量统计中 · 总 43.1k star*
  - 📌 **是什么**：一个注重隐私、可自托管、可定制的开源简历生成器，TypeScript 前端项目，同时带有 AI 与 MCP 相关标签。
  - 💡 **学习点**：对前端转型者来说，它是研究成熟 React/TypeScript 工程结构、以及如何把 AI/MCP 能力接入既有产品的好样本。
  - 🧭 **上手**：先跑本地自托管启动流程，再翻看它接入 MCP 或 AI 能力的相关代码，思考如果由你来加一个「AI 润色简历段落」功能会改哪些文件。
- [herdrdev/herdr（Agent 框架）](https://github.com/herdrdev/herdr) — *GitHub · Rust · 周增量统计中 · 总 39.2k star*
  - 📌 **是什么**：用 Rust 写的编码代理运行时/编排层，把多个编码代理与终端复用器统一在一个 CLI 环境里管理。
  - 💡 **学习点**：理解「代理运行时」这层抽象——会话、终端与代理进程如何被调度，是前端工程师从界面思维转向系统思维的好切入点。
  - 🧭 **上手**：从仓库的 CLI 入口与命令定义文件读起，本地跑一遍列出/切换代理会话的命令，体会它与直接开终端窗口的差别。
- [CopilotKit/CopilotKit（Agent 框架）](https://github.com/CopilotKit/CopilotKit) — *GitHub · TypeScript · 周增量统计中 · 总 37.4k star*
  - 📌 **是什么**：面向 React、Angular、移动端与 Slack 等平台的前端代理与生成式 UI 技术栈，并维护 AG-UI 协议。
  - 💡 **学习点**：这是前端工程师最自然的 AI 转型入口：把组件状态与代理状态双向打通，做出真正在界面里可交互的 AI 功能。
  - 🧭 **上手**：跑一遍官方 React 快速开始示例，重点看代理状态如何通过 hook 映射到组件 props，再对比 AG-UI 协议文档理解前后端消息约定。
- [alibaba/open-code-review](https://github.com/alibaba/open-code-review) — *GitHub · Go · 周增量统计中 · 总 34.6k star*
  - 📌 **是什么**：大规模生产环境验证过的代码审查工具，采用确定性流水线加 LLM Agent 的混合架构，输出精确到行的多语言规则审查意见。
  - 💡 **学习点**：可以学到「规则引擎兜底 + LLM 补充」的工程折中思路，以及如何把模型输出约束成结构化的行级注释。
  - 🧭 **上手**：先读它的规则定义与流水线编排部分，理解确定性检查与 LLM 检查的分工，再拿一个小仓库跑一次看评论格式。

## 🚀 技能提升点（工作总结汇总）

### 1. 自定义标签内容需转义再 innerHTML
- **技能点**：理解浏览器 HTML5 解析器对裸尖括号的容错行为，掌握在注入 innerHTML 前对自定义标签内容做实体转义、再由 textContent 无损取回的安全套路。
- **坑点**：把含 LaTeX（如 <b<、<a<）的 <question-latex> 内容直接塞进 el.innerHTML，浏览器把它当成 HTML 标签开始，破坏闭合标签，后续内容被吞进公式节点。
- **解决方案**：先用正则匹配 <question-latex>...</question-latex>，只对内部内容把 < > 转成 &lt; &gt; 再赋给 innerHTML；读取时用 textContent 自动解码还原原 latex。
```text
const safe = html.replace(/(<question-latex\b[^>]*>)([\s\S]*?)(<\/question-latex>)/gi,
  (_, o, c, e) => o + c.replace(/</g,'&lt;').replace(/>/g,'&gt;') + e);
el.innerHTML = safe;
```
- **拓展**：所有把用户或接口文本注入 innerHTML 的场景都适用，可沉淀为通用 escapeInnerHtml 工具。

### 2. 格式不一致数据先归一匹配
- **技能点**：掌握当接口返回值与页面已转换数据存在「壳/定界符」差异时，先做归一化再字面匹配的通用调试与修复能力。
- **坑点**：接口返回的 original 带 <question-latex> 标签壳或 $ 定界符，而正文已被转成无壳 <formula data-formula>，escapeRegExp 后字面匹配全部失败（matchCount=0）。
- **解决方案**：新增 normalizeTypoOriginal 剥掉 <question-latex> 标签和首尾 $，所有 doMark 与 fallback 替换调用统一改用归一串匹配。
```text
const normalizeTypoOriginal = (original) =>
  original.replace(/<\/?question-latex[^>]*>/gi, '').replace(/^\$|\$$/g, '');
```
- **拓展**：凡涉及多来源文本比对的场景都可先建「归一化层」，再分别处理仍无法匹配的脏数据。

### 3. antd v6 废弃 API 迁移映射
- **技能点**：建立组件库大版本升级时的 API 映射清单，并能识别「全局替换会误伤」的例外属性。
- **坑点**：沿用过时属性（bodyStyle、destroyOnClose、Drawer width）导致告警或行为异常；若把 direction 等合法属性也全局替换，会改坏 Anchor 等组件。
- **解决方案**：按官方映射逐项替换：bodyStyle→styles={{body}}、destroyOnClose/destroyInactiveTabPane→destroyOnHidden、Drawer width→size，同时标注 Modal width 未废弃、Anchor direction 合法。
```text
// bodyStyle -> styles={{ body }}
// destroyOnClose / destroyInactiveTabPane -> destroyOnHidden
// Drawer width -> size（Modal width 未废弃）
// ⚠️ Anchor 的 direction 合法，勿全局替换
```
- **拓展**：每次升级后沉淀一张项目级迁移对照表，配 lint 规则自动拦截旧属性。
- *来源：admin-workspace-hr-talent*

### 4. 仅列表区域滚动 flex 布局
- **技能点**：掌握「头部固定 + 表格区滚动 + 分页条常驻」的三层 flex 布局定稿模式，理解 min-height:0 对 flex 子项收缩的必要性。
- **坑点**：只设 height:100% 与 overflow:auto，flex 子项默认 min-height:auto 无法收缩，导致表格不滚动或分页条被挤出视口。
- **解决方案**：根 div height:100% + flex column + overflow:hidden，头部/分页 flexShrink:0，表格包裹层 flex:1 + minHeight:0 + overflow:auto 作为唯一滚动容器。
```text
<div style={{ height:'100%', display:'flex', flexDirection:'column', overflow:'hidden' }}>
  <div style={{ flexShrink:0 }}>头部</div>
  <div style={{ flex:1, minHeight:0, overflow:'auto' }}>表格</div>
  <div style={{ flexShrink:0 }}>分页条</div>
</div>
```
- **拓展**：可作为通用 PageLayout 组件封装，避免每页重复拼装。
- *来源：admin-workspace-hr-talent*

### 5. Table dataIndex 类型盲区
- **技能点**：意识到 TS 对字符串型 dataIndex 不做键名校验，改字段名后必须逐列人工核对渲染列。
- **坑点**：dataIndex 类型为 string（联合类型退化为 string），字段名写错不报编译错，只静默渲染空白，排查成本高。
- **解决方案**：改行类型字段名后逐列核对 dataIndex 与 rowKey，必要时用 satisfies 或常量映射收敛列定义。
```text
// dataIndex 不受类型约束，写错不报编译错，只静默渲染空白
{ title: '姓名', dataIndex: 'userNmae' } // 改行类型字段名后必须逐列核对
```
- **拓展**：可写自定义 eslint 规则或类型工具，从行类型反查 dataIndex 合法性。
- *来源：admin-workspace-hr*

### 6. 替换第三方模块补默认 matcher
- **技能点**：掌握「自定义模块覆盖第三方默认实现后，必须显式补齐被覆盖的默认行为」这一通用排查思路。
- **坑点**：quill-table-up 的 TableClipboard 只加表格 matcher，不继承默认 clipboard 的 image/divider matcher，导致粘贴/回显时图片与分割线被静默丢弃。
- **解决方案**：在 registerClipboardMatchers 中显式 addMatcher img[data-type=ql-image] 与 divider，属性提取与对应 Blot.value 结构对齐。
```text
clipboard.addMatcher('img[data-type="ql-image"]', node =>
  new Delta().insert({ image: { url: node.getAttribute('src') } }));
clipboard.addMatcher('hr.ql-divider, div.ql-divider', node =>
  new Delta().insert({ divider: { dataType: 'divider' } }));
```
- **拓展**：任何「接管/覆盖第三方模块」的场景都要列出原默认能力清单，逐项确认是否需恢复。
- *来源：admin-workspace-new*

### 7. 异步初始化 v-model 时序缓存
- **技能点**：掌握封装第三方库时处理 v-model 首次传值与组件异步初始化顺序矛盾的能力。
- **坑点**：watch modelValue 加 immediate 在 setup 阶段触发，但此时 quillInstance 尚未创建，直接 return 会把首次传入的初始 HTML 丢掉，弹窗里显示空白。
- **解决方案**：加 pendingModelValue 缓存，watch 在实例未就绪时暂存值；initQuill 完成后 nextTick 调 flushPendingModelValue 消费。
```text
watch(() => props.modelValue, v => {
  if (!quillInstance) { pendingModelValue = v; return; }
  setContent(v);
}, { immediate: true });
// onMounted 里 initQuill 完成后 flushPendingModelValue()
```
- **拓展**：该模式可推广到 ECharts、地图等「挂载后才初始化」的封装组件。
- *来源：admin-workspace-new*

