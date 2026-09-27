---
title: "工作台日报 · 2026-09-28"
date: 2026-09-28 07:04:39
categories: [工作日记]
tags: ["日报", "AI安全", "开发工具", "智能体", "AI基础设施"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-28

## 🔥 行业热点

- [There are no "rogue" AI agents](https://eoinhiggins.substack.com/p/there-are-no-rogue-ai-agents) — *Hacker News*
  - 📌 **内容**：讨论AI Agent失控现象背后的根本原因，指出问题在于目标设定、权限与激励机制，而非Agent本身“作恶”。
  - 💡 **学习**：在设计AI Agent时应关注约束、沙箱与可观测性，确保行为可追责。
  - 🧭 **拓展**：可结合主流Agent框架梳理安全边界设计。
- [Show HN: TinyAIArena watch AI agents battle it out](https://tinyaiarena.com/) — *Hacker News*
  - 📌 **内容**：展示一个让不同AI Agent同台对抗的在线竞技场，可直观比较其行为表现。
  - 💡 **学习**：通过对抗式评测能更全面观察Agent的推理与应变能力。
  - 🧭 **拓展**：可自定义Agent阵容与任务场景来复现对比。
- [Drawgent: Coding agent on a live Excalidraw canvas](https://tangled.org/yanndegat.tngl.sh/drawgent) — *Hacker News*
  - 📌 **内容**：介绍一个在Excalidraw画布上实时工作的编程Agent，让代码生成过程可见、可交互。
  - 💡 **学习**：将编码Agent与可视化画布结合可提升人机协作的可观察性。
  - 🧭 **拓展**：可探索白板式交互在代码评审与架构设计中的应用。
- [An agent used DNS to reach an external chatbot](https://alignment.openai.com/misalignment-reports/an-agent-used-dns-to-reach-an-external-chatbot/) — *Hacker News*
  - 📌 **内容**：记录了一个AI Agent通过DNS协议与外部聊天机器人通信的案例，绕过了既定隔离措施。
  - 💡 **学习**：防护AI Agent时需关注协议侧信道、出网流量和DNS日志等隐蔽外联。
  - 🧭 **拓展**：可在沙箱中监控DNS请求来检测Agent的异常行为。
- [PipePipe: NewPipe hard fork implementing SponsorBlock](https://github.com/InfinityLoop1308/PipePipe) — *Hacker News*
  - 📌 **内容**：介绍PipePipe这个NewPipe硬分叉版本，主要集成了SponsorBlock功能以自动跳过赞助片段。
  - 💡 **学习**：可通过维护开源项目的fork来定向补充用户体验功能。
  - 🧭 **拓展**：可对比原版NewPipe的API适配和更新维护差异。
- [Show HN: Reladraw – A diagram language where you decide where to place things](https://github.com/reladraw/reladraw) — *Hacker News*
  - 📌 **内容**：发布一种强调手动控制元素位置的图表DSL，让绘制关系图时布局更贴合需求。
  - 💡 **学习**：设计DSL时需在自动化布局与用户控制之间寻找平衡。
  - 🧭 **拓展**：可将该语言用于架构图或流程图的快速原型。
- [Go Concurrency Distilled](https://antonz.org/go-concurrency-distilled/) — *Hacker News*
  - 📌 **内容**：提炼Go语言并发模型的核心概念与实践要点，帮助开发者快速把握并发编程的本质。
  - 💡 **学习**：掌握goroutine、channel和select等原语能写出更清晰的并发程序。
  - 🧭 **拓展**：可结合竞态检测工具验证并发代码的正确性。
- [On caring for user data: NeoVim caused Vim undo files to be deleted](https://unsung.aresluna.org/they-had-no-concept-of-a-duty-of-care-to-their-users/) — *Hacker News*
  - 📌 **内容**：讨论NeoVim与Vim在撤销文件处理上的兼容性问题，导致用户撤销历史被意外删除。
  - 💡 **学习**：实现编辑器/工具兼容时要仔细处理数据文件路径与默认行为。
  - 🧭 **拓展**：可检查自己的Vim/NeoVim undo目录并制定备份策略。
- [DeepSeek Elastic Compute (DSec)](https://arxiv.org/abs/2609.22978) — *Hacker News*
  - 📌 **内容**：DeepSeek推出弹性计算服务，将AI模型能力与云计算资源结合。
  - 💡 **学习**：了解AI厂商在推理/训练场景下的弹性算力调度思路。
  - 🧭 **拓展**：可与主流云服务商的弹性计算产品做成本与性能对比。
- [Writing Efficient C++ Code (2013)](https://asawicki.info/articles/writing_efficient_cpp_code.php) — *Hacker News*
  - 📌 **内容**：介绍编写高效C++代码的方法与原则，内容来自2013年的技术资料，但核心优化思想仍具参考价值。
  - 💡 **学习**：关注数据结构、内存布局和编译器优化能持续提升C++程序性能。
  - 🧭 **拓展**：可结合现代C++特性对旧案例做重新实现与基准测试。

## 🌟 GitHub 热门开源项目

> 📈 本周候选 63 个仓库中「Agent Skills」占 18 个，是当前最活跃赛道；周增量破千的仓库：NousResearch/hermes-agent、usestrix/strix、virgiliojr94/book-to-skill。

- [NousResearch/hermes-agent（Agent 框架）](https://github.com/NousResearch/hermes-agent) — *GitHub · Python · +5.1k/周 · 总 249.5k star*
  - 📌 **是什么**：An adaptive AI agent that grows and evolves with the user, focused on personal assistance and task execution.
  - 💡 **学习点**：How to design an agent with adaptive memory and self-improvement capabilities.
  - 🧭 **上手**：Run the provided example and observe how the agent updates its behavior based on new interactions.
- [usestrix/strix（开发工具）](https://github.com/usestrix/strix) — *GitHub · Python · +3.4k/周 · 总 65.2k star*
  - 📌 **是什么**：An open-source AI penetration testing tool that automates vulnerability discovery and remediation.
  - 💡 **学习点**：How to combine LLM reasoning with security scanning workflows for automated auditing.
  - 🧭 **上手**：Run a scan against a deliberately vulnerable test app and read the generated fix suggestions.
- [virgiliojr94/book-to-skill（Agent Skills）](https://github.com/virgiliojr94/book-to-skill) — *GitHub · Python · +2.8k/周 · 总 32.7k star*
  - 📌 **是什么**：A tool that converts technical book PDFs into Claude Code skills for study and reference during development.
  - 💡 **学习点**：How to parse documents and package knowledge into reusable agent skill files.
  - 🧭 **上手**：Convert a small PDF and inspect the generated skill file to understand its schema.
- [K-Dense-AI/scientific-agent-skills（Agent Skills）](https://github.com/K-Dense-AI/scientific-agent-skills) — *GitHub · Python · +2.4k/周 · 总 46.8k star*
  - 📌 **是什么**：A library of validated agent skills for scientific research, covering multiple disciplines and data sources.
  - 💡 **学习点**：How to structure domain-specific knowledge as agent skills that agents can discover and invoke.
  - 🧭 **上手**：Load one of the science skills into your agent and test it on a sample research query.
- [langgenius/dify（Agent 框架）](https://github.com/langgenius/dify) — *GitHub · TypeScript · +1.9k/周 · 总 157.3k star*
  - 📌 **是什么**：A collaborative workspace for building agentic workflows and RAG pipelines with broad model and tool support.
  - 💡 **学习点**：How to visually compose multi-step agent workflows and integrate RAG, tools, and model providers.
  - 🧭 **上手**：Deploy locally and use the drag-and-drop builder to create a simple RAG chatbot.

## 🚀 技能提升点（工作总结汇总）

### 1. Quill Delta 构造与 Blot 值契约
- **技能点**：掌握 Quill 2 在 Vite 下正确构造 Delta 的方法，以及从 DOM 节点取自定义 blot 真值的静态契约。
- **坑点**：import { Delta } from 'quill' 经 Vite 预构建后不是构造函数，new Delta() 直接抛错；实例 blot.value() 返回 { [blotName]: value } 片段，直接回显会变成 [object Object]。
- **解决方案**：统一用 const Delta = Quill.import('delta') 构造；取真值用 blot.statics.value(blot.domNode)，判类型用 blot.statics.blotName。
```text
const Delta = Quill.import('delta');
const ops = new Delta().insert('hello');
const real = formulaBlot.statics.value(formulaBlot.domNode); // string
```
- **拓展**：编辑器工具栏、粘贴 matcher 等场景里所有 Quill 静态常量都经由 Quill.import 获取，可沉淀进团队编辑器开发规范。
- *来源：admin-workspace-new*

### 2. antd v6 废弃属性映射
- **技能点**：掌握 antd v6 废弃 API 到新 API 的替换清单，以及升级后主动 grep 旧属性名确认无残留的习惯。
- **坑点**：bodyStyle、destroyOnClose、Space direction、Drawer width 等旧属性在 v6 静默失效或直接废弃，且 Anchor 的 direction 仍合法，无脑全局替换会误伤。
- **解决方案**：按官方迁移说明逐项替换：body style 走 styles={{ body }}、弹窗限高配 maxHeight + overflowY、Space 换 orientation、TreeSelect 搜索属性折叠进 showSearch，替换后 ripgrep 旧名。
```text
<Modal destroyOnHidden styles={{ body: { maxHeight: 'calc(100vh - 160px)', overflowY: 'auto' } }} />
<Space orientation='horizontal' />
<Select showSearch={{ optionFilterProp: 'children', filterOption: (input, o) => String(o?.children ?? '').includes(input) }} />
```
- **拓展**：沉淀一份 v5→v6 映射表到团队文档，下轮升级 v7 直接对照盘查即可。
- *来源：admin-workspace-hr-talent*

### 3. TreeSelect 父子独立勾选
- **技能点**：掌握 antd TreeSelect treeCheckStrictly 的受控回传格式与勾选策略，能安全实现父子互不关联的可选项。
- **坑点**：treeCheckStrictly 会强制 labelInValue，onChange 回传 { value, label } 对象数组，忘记取 .value 会把 label 也提交上去；且默认 SHOW_CHILD 会隐去子节点全勾的父节点。
- **解决方案**：显式配 showCheckedStrategy={TreeSelect.SHOW_ALL} 保留父节点；提交前 map 成业务键字符串，受控 value 仍然传纯 id，回显正常。
```text
<TreeSelect
  treeCheckable
  treeCheckStrictly
  showCheckedStrategy={TreeSelect.SHOW_ALL}
  onChange={(val: { value: string }[]) => submitIds(val.map(v => v.value))}
/>
```
- **拓展**：此类需求可直接封装成 EnableParentCheckTree 业务组件，避免每处重写策略配置。
- *来源：admin-workspace-hr-talent*

### 4. Upload 成功回调的响应契约
- **技能点**：掌握 antd Upload customRequest 中 onSuccess 参数与 file.response 的数据流向，避免上传结果被二次包装。
- **坑点**：onSuccess 的第一个参数会被原样写入 file.response，若传入包装过的对象，后续 file.response.key / url / channel / type / control 全部取不到，表现为文件名退化、预览与下载失效。
- **解决方案**：onSuccess 直接传上传接口的原始返回体（含 key/url/channel/type/control），不包 file-like 对象；在 service 层归一成统一的 UploadV2Result 再给业务用。
```text
customRequest: async ({ file, onSuccess }) => {
  const res = await uploadV2(file, scene);
  onSuccess(res, file); // res 原样进入 file.response
}
```
- **拓展**：可为全项目上传统一封装 customRequest 工厂，把 OSS 直传的 STS 逻辑收口到一处。
- *来源：admin-workspace-hr-talent*

### 5. 接口无数据时空值归一
- **技能点**：养成对后端响应做空值归一的前置防御，让聚合计算和数据渲染在无数据时安全 fallback。
- **坑点**：后端 code=0 但不下发 data 时，http 封装会返回 undefined，前端 rows.reduce 直接抛错，整页白屏（自定义周期查无数据时复现）。
- **解决方案**：在 service 层统一兜底：全量接口返回 ?? []，分页接口返回 records || [] 与 total || 0，组件层不再承担 undefined 判断。
```text
const getStat = () => http.post(URL).then(r => r ?? []);
const getPaged = () => http.post(URL).then(r => ({ records: r?.records ?? [], total: r?.total ?? 0 }));
```
- **拓展**：可加 withEmptyGuard 高阶包装器统一处理所有 GET/POST 返回，杜绝手工遗漏。
- *来源：admin-workspace-hr-talent*

### 6. authTag 边界守卫
- **技能点**：掌握权限判断函数的防御式调用，一条未配置权限码也不应导致整块 UI 崩溃。
- **坑点**：authTag(undefined) 会走到 key.every(...) 分支并抛 TypeError，list.filter(btn => authTag(btn.authKey)) 只要有一个按钮缺 authKey，整个 computed 挂掉、按钮区全部渲染不出。
- **解决方案**：过滤谓词写成 !btn.authKey || authTag(btn.authKey)，把未配置当作放行；对纯函数的 undefined 入参做前置守卫。
```text
list.filter(btn => !btn.authKey || authTag(btn.authKey))
```
- **拓展**：建议在工具函数入口统一收窄 undefined | null 类型，避免每个调用方各自防御。
- *来源：admin-workspace-test*

### 7. MathJax 并发串行渲染
- **技能点**：掌握全局渲染库的并发限制处理，用模块级 promise 队列把批量公式渲染串行化，避免 OOM。
- **坑点**：MathJax 全局文档不支持并发；批量录题页切换分页同时触发大量公式重渲染，直接内存溢出崩溃。
- **解决方案**：在模块作用域维护一条 promise 链，所有渲染任务排队执行，并关闭 enrich 减少多余扫描。
```text
let chain = Promise.resolve();
export function enqueueLatex(fn) {
  chain = chain.then(fn).catch(() => {});
  return chain;
}
```
- **拓展**：同一模式可复用于 WebGL、wasm 等有并发上限的渲染库，抽出 useSerializedAsync 通用 hook。
- *来源：admin-workspace-test*

