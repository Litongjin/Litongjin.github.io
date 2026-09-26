---
title: "工作台日报 · 2026-09-27"
date: 2026-09-27 07:03:48
categories: [工作日记]
tags: ["日报", "AI安全", "AI工具", "AI智能体", "云计算"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-27

## 🔥 行业热点

- [Revealing the details of how OpenAI agents hacked Hugging Face](https://swarmtraces.org/) — *Hacker News*
  - 📌 **内容**：披露了OpenAI智能体攻破Hugging Face平台的具体细节，展示AI代理在真实环境中执行复杂操作的能力与风险。
  - 💡 **学习**：学习AI代理的权限隔离、认证绕过等安全知识。
  - 🧭 **拓展**：可在隔离环境复现类似的代理攻击路径以验证防护策略。
- [OpenAI bots meddled with multiple US Government agency sites](https://www.bbc.com/news/articles/cw62jje658dlo) — *Hacker News*
  - 📌 **内容**：报道OpenAI的机器人在多个美国政府网站上产生干扰行为，引发对自动化代理合规性的讨论。
  - 💡 **学习**：了解AI自动化访问网络服务的合规边界与治理机制。
  - 🧭 **拓展**：可研究政府网站的robots协议与安全响应机制。
- [Drawgent: Coding agent on a live Excalidraw canvas](https://tangled.org/yanndegat.tngl.sh/drawgent) — *Hacker News*
  - 📌 **内容**：在实时Excalidraw画布上运行的编程智能体，将代码生成与可视化绘图结合在一起。
  - 💡 **学习**：学习将LLM代理接入画布API，实现实时交互式编程。
  - 🧭 **拓展**：可尝试用Excalidraw的实时API开发自定义编码代理。
- [OpenAI pauses training of its 'most capable models'](https://www.theverge.com/ai-artificial-intelligence/1001049/openai-training-pause) — *Hacker News*
  - 📌 **内容**：OpenAI暂停其最强大模型的训练，可能出于安全评估或战略调整。
  - 💡 **学习**：关注大模型训练暂停背后的安全评估与计算资源权衡。
  - 🧭 **拓展**：可跟踪OpenAI后续发布的安全报告与模型路线图。
- [Ollaya – Ollama for open-source, Jev-style decision models](https://ollaya.dev/) — *Hacker News*
  - 📌 **内容**：介绍Ollaya，一个开源决策模型运行工具，借鉴Ollama模式，支持Jev风格模型。
  - 💡 **学习**：掌握本地部署决策模型的方法，比较推理框架接口设计。
  - 🧭 **拓展**：可安装Ollaya并与Ollama对比测试模型推理效果。
- [What even is an OS now?](https://sockpuppet.org/blog/2026/09/25/what-even-is-an-os-now/) — *Hacker News*
  - 📌 **内容**：探讨现代技术栈中操作系统的定义演变，及其在容器、虚拟化和云环境下的角色。
  - 💡 **学习**：理解操作系统在云原生环境下的抽象层次变化。
  - 🧭 **拓展**：可对比传统操作系统与WebAssembly/容器OS的架构。
- [PipePipe: NewPipe hard fork implementing SponsorBlock](https://github.com/InfinityLoop1308/PipePipe) — *Hacker News*
  - 📌 **内容**：PipePipe是NewPipe的一个硬分叉，集成了SponsorBlock以跳过赞助段落。
  - 💡 **学习**：学习开源分叉流程及SponsorBlock等第三方集成方式。
  - 🧭 **拓展**：可阅读PipePipe源码了解NewPipe的扩展点。
- [Show HN: Jev Plays Pokémon Red](https://jev-pokemon.vercel.app/) — *Hacker News*
  - 📌 **内容**：一个名为Jev的AI智能体演示如何游玩《宝可梦红》，通过视觉识别屏幕并实时做出操作。
  - 💡 **学习**：学习视觉模型驱动的游戏AI构建，包括屏幕识别与动作映射。
  - 🧭 **拓展**：可用类似方法让智能体玩其他复古游戏进行验证。
- [Floci: Locally emulating any cloud service](https://floci.io) — *Hacker News*
  - 📌 **内容**：Floci能够在本地模拟各种云服务，便于开发和测试。
  - 💡 **学习**：学习云服务仿真器的设计思路，掌握本地集成测试技巧。
  - 🧭 **拓展**：可在CI流程中集成Floci模拟云服务依赖。
- [Modern Object Pascal Introduction for Programmers](https://castle-engine.io/modern_pascal) — *Hacker News*
  - 📌 **内容**：面向程序员的现代Object Pascal入门内容，涵盖语言基础与当代特性。
  - 💡 **学习**：学习Object Pascal语法与面向对象设计，拓展超越主流语言的视野。
  - 🧭 **拓展**：可尝试用Free Pascal编写命令行小工具。

## 🌟 GitHub 热门开源项目

- [firecrawl/firecrawl（开发工具）](https://github.com/firecrawl/firecrawl) — *GitHub · TypeScript · +6.1k/周 · 总 185.1k star*
  - 📌 **是什么**：A web data API for searching, scraping, and interacting with web content at scale, designed for AI and LLM pipelines.
  - 💡 **学习点**：Understand how to build robust crawling/scraping services that feed structured data into LLM workflows.
  - 🧭 **上手**：Read the API docs and run a quick scrape-to-markdown example to see how HTML is cleansed for LLM use.
- [blader/humanizer（Agent Skills）](https://github.com/blader/humanizer) — *GitHub · Python · +5.5k/周 · 总 52.2k star*
  - 📌 **是什么**：An agent skill that removes AI-generated writing patterns from text to make it sound more human.
  - 💡 **学习点**：Learn how to package prompt-engineering logic as a reusable agent skill for coding assistants.
  - 🧭 **上手**：Inspect the skill file to see how prompts are structured for Claude Code/Cursor compatibility.
- [calesthio/OpenMontage（Agent 框架）](https://github.com/calesthio/OpenMontage) — *GitHub · Python · +4.3k/周 · 总 61.4k star*
  - 📌 **是什么**：An open-source, agentic video production system with 12 production pipelines, 100+ tools, and 700+ agent skill files.
  - 💡 **学习点**：See how large agentic workflows are decomposed into tools, skills, and knowledge files.
  - 🧭 **上手**：Browse the skill directory and trace one pipeline from prompt to video output.
- [cathrynlavery/diagram-design（Agent Skills）](https://github.com/cathrynlavery/diagram-design) — *GitHub · HTML · +4.3k/周 · 总 42.5k star*
  - 📌 **是什么**：A self-contained editorial diagram design skill for coding agents, producing clean HTML+SVG diagrams without Mermaid.
  - 💡 **学习点**：Learn how to constrain agent output to create high-quality visual assets via detailed guidelines.
  - 🧭 **上手**：Open one of the skill files and try generating an SVG diagram in Claude Code.
- [herdrdev/herdr（开发工具）](https://github.com/herdrdev/herdr) — *GitHub · Rust · +1.7k/周 · 总 40.9k star*
  - 📌 **是什么**：A runtime for coding agents, providing orchestration, multiplexing, and CLI tools to manage multiple agent sessions.
  - 💡 **学习点**：Understand how to build infrastructure that supervises and coordinates multiple coding agents.
  - 🧭 **上手**：Run herdr with multiple agents to see how it multiplexes sessions and manages context.

## 🚀 技能提升点（工作总结汇总）

### 1. Quill Blot 取值与 Delta 构造
- **技能点**：掌握 Parchment blot 静态取值契约，以及 Vite 环境下安全获取 Delta 的方式，避免富文本回显和插入数据损坏。
- **坑点**：实例 blot.value() 返回的是 delta 片段而非原始值；import { Delta } from 'quill' 在 Vite 预构建后不是构造函数；embed blot 的 create(value) 收到的 value 常被包成 { blotName: value }。
- **解决方案**：统一用 blot.statics.value(blot.domNode) 取真值；用 Quill.import('delta') 获取 Delta；create 里加 typeof value === 'string' ? value : value?.[name] ?? '' 守卫。
```text
const Delta = Quill.import('delta');
class MyBlot extends BlockEmbed {
  static create(value) {
    const node = super.create();
    node.setAttribute('data-value', typeof value === 'string' ? value : value?.[name] ?? '');
    return node;
  }
  static value(domNode) { return domNode.getAttribute('data-value'); }
}
```
- **拓展**：自定义 embed blot、跨 Quill 版本迁移和 clipboard matcher 都遵循同一套契约，可沉淀为富文本自定义格式开发规范。
- *来源：admin-workspace-new | MEMORY.md*

### 2. antd Upload onSuccess 与 file.response
- **技能点**：正确使用 antd Upload customRequest 的回传参数，保证上传结果对象能被业务层稳定读取。
- **坑点**：antd 会把 onSuccess 的参数原样写进 file.response；若传的是包装过的对象，file.response.key/url/channel/type 等全部取不到，导致文件名退化、预览下载失效。
- **解决方案**：customRequest 里直接 onSuccess(上传服务返回的结果对象)，不要再包一层；按 UploadV2Result 固定返回结构。
```text
customRequest: async ({ file, onSuccess }) => {
  const result = await uploadV2(file, scene);
  onSuccess(result);
}
```
- **拓展**：统一封装上传服务与结果类型，各业务场景直接消费 file.response 即可复用。
- *来源：admin-workspace-hr-talent | MEMORY.md*

### 3. TreeSelect 受控值/回传陷阱
- **技能点**：分清 antd TreeSelect 的 treeCheckStrictly、showCheckedStrategy 对 value 形态和勾选提交值的影响。
- **坑点**：treeCheckStrictly 会强制 labelInValue，onChange 回传 { value, label }[] 而非纯 id；SHOW_CHILD 下用 maxCount 会被内部逻辑拦截勾选，不能用于标签折叠。
- **解决方案**：严格模式提交前取 item.value，受控回显仍可传纯 id；标签折叠用 tagRender/CSS，不用 maxCount。
```text
<TreeSelect
  treeCheckable
  treeCheckStrictly
  showCheckedStrategy={TreeSelect.SHOW_ALL}
  onChange={(items) => submit(items.map((item) => String(item.value)))}
/>
```
- **拓展**：部门树、权限树等需要父子独立勾选的场景都可复用，先读源码确认策略再封装组件。
- *来源：admin-workspace-hr-talent | MEMORY.md*

### 4. 列表页唯一滚动容器布局
- **技能点**：掌握 flex 布局下“仅列表区滚动 + 常驻分页条”的标准结构，避免整页滚动错位或底部被裁。
- **坑点**：根节点用 minHeight 而非 height 会撑不到底露灰底；滚动区漏设 minHeight:0 会导致 flex 子项溢出；外层 content-box padding 会让 height:100% 底部被裁。
- **解决方案**：根 div 用 height:100% + flex column + overflow:hidden；头部/分页条 flexShrink:0；滚动区 flex:1 + minHeight:0 + overflow:auto。
```text
.wrapper { height:100%; display:flex; flex-direction:column; overflow:hidden; }
.header, .pagination { flex-shrink:0; }
.scroll { flex:1; min-height:0; overflow:auto; }
```
- **拓展**：可作为内容区滚动通用模板，遇到 sticky 表头时再按有无 scroll.x 选择 sticky prop 或 CSS 方案。
- *来源：admin-workspace-hr-talent | MEMORY.md*

### 5. 接口空数据归一化
- **技能点**：在 service 层统一把后端缺省返回归为数组或默认值，从源头保证聚合计算安全。
- **坑点**：后端无数据时 code=0 但不下发 data，http 返回 undefined；若直接交给 useFullList，rows 变 undefined，sumBy/reduce 会崩成整页白屏。
- **解决方案**：service 层对全量列表和详情接口统一 ?? [] / records || []，分页/全量 hooks 不再兜底，契约由 service 保证。
```text
const list = (await http.post<Item[]>(url, body)) ?? [];
const records = data?.records || [];
```
- **拓展**：可抽一个 toArray 工具，对所有返回数组的业务接口做统一归一，并加接口返回类型校验。
- *来源：admin-workspace-hr-talent | MEMORY.md*

### 6. 权限 key 缺失防御
- **技能点**：编写菜单/按钮权限过滤时对可选 key 做防御，避免单个配置缺失拖垮整个渲染。
- **坑点**：自研 authTag(key) 对非字符串 key 走 key.every(...)，传 undefined 会抛 TypeError；filter(btn => authTag(btn.authKey)) 在任一按钮没配 authKey 时让 computed 崩溃、按钮区全部消失。
- **解决方案**：统一写成 !btn.authKey || authTag(btn.authKey)；在 authTag 实现层对非字符串输入直接返回安全值。
```text
list.filter((btn) => !btn.authKey || authTag(btn.authKey))
```
- **拓展**：可封装 hasAuth(key?) 工具并作为权限过滤唯一入口，配套 lint 规则强制可选 key 必须判空。
- *来源：admin-workspace-test | MEMORY.md*

