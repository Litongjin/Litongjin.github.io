---
title: "工作台日报 · 2026-10-18"
date: 2026-10-18 07:05:15
categories: [工作日记]
tags: ["日报", "大模型", "AI", "安全", "AI工具"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-10-18

## 🔥 行业热点

- [Llama.cpp on a jailbroken PS5: run any GGUF model natively on the console](https://github.com/cobanov/PS5LM) — *Hacker News*
  - 📌 **内容**：有开发者在越狱的 PS5 上运行 llama.cpp，可原生加载任意 GGUF 格式模型，把游戏主机变成本地推理设备。
  - 💡 **学习**：了解 llama.cpp 的跨平台可移植性思路：通过适配 CPU 指令集与内存布局，可在非传统设备上跑本地大模型。
  - 🧭 **拓展**：可尝试在闲置主机/掌机上部署量化模型，实测 token 速度与内存占用，评估边缘推理的可行性。
- [Pele – Mermaid diagrams as themeable SVG](https://pele.run/) — *Hacker News*
  - 📌 **内容**：Pele 将 Mermaid 图表渲染为可主题化的 SVG，便于在文档、博客中统一图表风格并摆脱运行时 JS 依赖。
  - 💡 **学习**：学习将图表 DSL 编译为静态 SVG 的工具链思路，以及如何用主题变量统一视觉输出。
  - 🧭 **拓展**：可在自己的技术博客/文档站接入类似渲染管线，验证构建期生成图表对加载性能的改善。
- [Tell HN: Remember to migrate away from .name domain](https://news.ycombinator.com/item?id=50089577) — *Hacker News*
  - 📌 **内容**：社区提醒 .name 顶级域服务/支持存在风险，建议用户尽早迁移域名以避免解析与续费问题。
  - 💡 **学习**：了解小众 TLD 的运营风险，以及域名迁移中 DNS 配置、邮箱与证书联动的注意事项。
  - 🧭 **拓展**：检查自己项目的域名与 DNS 依赖，做一次迁移演练与回滚预案。
- [Microsoft and CMU team up to train early career software engineers](https://www.cs.cmu.edu/news/2026/microsoft-cmu-praise) — *Hacker News*
  - 📌 **内容**：微软与卡内基梅隆大学合作培养初级软件工程师，探索产业界与高校联合的新培养模式。
  - 💡 **学习**：关注企业在 AI 时代对工程师能力要求的变化：更强调系统设计、协作与 AI 工具运用。
  - 🧭 **拓展**：可对照此类课程体系梳理自己的技能缺口，制定学习路线。
- [The math that OpenAI keeps for itself](https://newsletter.danielpaleka.com/p/the-math-that-openai-keeps-for-itself) — *Hacker News*
  - 📌 **内容**：文章讨论 OpenAI 在模型训练中未公开的关键数学与工程细节，如缩放规律、数据配比等核心 know-how。
  - 💡 **学习**：认识大模型训练中'公开论文 vs 内部实践'的差距，重视超参、数据混合等隐性工程知识。
  - 🧭 **拓展**：可阅读公开的 scaling law 论文并做小模型复现实验，体会其中的经验性成分。
- [Claude reveals ECDSA curve origins](https://www.anthropic.com/research/ecdsa-curve-origins) — *Hacker News*
  - 📌 **内容**：Claude 输出了关于 ECDSA 椭圆曲线参数起源的解释，引发关于加密标准历史与 AI 知识准确性的讨论。
  - 💡 **学习**：了解 ECDSA 曲线参数（如 secp256k1/P-256）来源的密码学背景，同时警惕 LLM 在技术史实上的幻觉。
  - 🧭 **拓展**：可让多个模型回答同一密码学史实问题并与权威资料比对，练习事实核查。
- [OpenCode Hacked: User credentials leaked](https://twitter.com/opencode/status/2111516673953935833) — *Hacker News*
  - 📌 **内容**：开源 AI 编码工具 OpenCode 被入侵，用户凭据泄露，涉及大量开发者的 API key 与账号安全。
  - 💡 **学习**：重视 AI 编码工具链的供应链安全：最小化权限、定期轮换密钥、审计第三方工具的数据访问。
  - 🧭 **拓展**：检查自己使用的 AI 编程插件的权限范围，启用密钥轮换与使用审计。
- [DeepSeek-v4.1-Flash-1017](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash-1017) — *Hacker News*
  - 📌 **内容**：DeepSeek 发布 v4.1-Flash 新版本模型，延续其高性价比路线，面向低延迟推理场景。
  - 💡 **学习**：关注国产开源模型的迭代方向：在推理速度、成本与能力之间的权衡与优化手段。
  - 🧭 **拓展**：可在本地或 API 上对比该模型与同类模型在代码任务上的表现与延迟。
- [You could have invented transformers from memory](https://alexwang.ai/posts/associative-memory/) — *Hacker News*
  - 📌 **内容**：文章以'你本可以从记忆中发明 Transformer'为题，拆解注意力机制的思想演进，降低理解门槛。
  - 💡 **学习**：通过第一性原理重建 Transformer：从序列建模的需求出发理解注意力、位置编码与多头机制的设计动机。
  - 🧭 **拓展**：尝试从零手写一个最小 Transformer（如 nanoGPT 风格），加深对架构的理解。
- [Ultramarine 1 Flash](https://www.paperinstruments.com/ultramarine) — *Hacker News*

## 🌟 GitHub 热门开源项目

- [Panniantong/Agent-Reach（MCP 工具）](https://github.com/Panniantong/Agent-Reach) — *GitHub · Python · +16.2k/周 · 总 98.7k star*
  - 📌 **是什么**：一个给 AI Agent 装上网页“眼睛”的 CLI 工具，统一读取和搜索 Twitter、Reddit、YouTube、GitHub、B 站、小红书等平台内容，号称零 API 费用。
  - 💡 **学习点**：学习如何用统一的工具接口封装异构数据源，让 Agent 通过命令行/MCP 获得实时互联网信息获取能力。
  - 🧭 **上手**：跑通它的 CLI 安装流程，先对 GitHub 或 Reddit 做一次搜索，观察返回的数据结构再接入自己的 Agent。
- [addyosmani/agent-skills（Agent Skills）](https://github.com/addyosmani/agent-skills) — *GitHub · JavaScript · +12.4k/周 · 总 107.8k star*
  - 📌 **是什么**：面向 AI 编程 Agent 的生产级工程技能集合，把资深工程师的实践沉淀为可被 Claude Code、Cursor 等调用的 skill 文件。
  - 💡 **学习点**：学习如何写出结构清晰、约束明确的 Agent Skill 提示词，把个人工程经验转化为 Agent 可复用的能力。
  - 🧭 **上手**：挑一个与你日常开发相关的 skill 文件精读，再把它装进 Claude Code 里实际跑一次任务对比效果。
- [farion1231/cc-switch（开发工具）](https://github.com/farion1231/cc-switch) — *GitHub · Rust · +12.3k/周 · 总 145.5k star*
  - 📌 **是什么**：用 Rust 写的跨平台桌面工具，统一管理 Claude Code、Codex、OpenCode 等多种 AI 编程助手的配置与切换。
  - 💡 **学习点**：了解多 CLI Agent 时代的工具链整合思路：配置管理、账号切换与桌面端产品设计。
  - 🧭 **上手**：下载安装后配置一个备用 API 端点，体验一键切换不同编程 Agent 后端的流程。
- [firecrawl/firecrawl（开发工具）](https://github.com/firecrawl/firecrawl) — *GitHub · TypeScript · +11.2k/周 · 总 192.5k star*
  - 📌 **是什么**：把任意网站抓取并转成干净的 LLM 可用 Markdown 数据的 Web 数据引擎，为 Agent 提供结构化网页输入。
  - 💡 **学习点**：学习网页抓取、清洗与 HTML→Markdown 转换的工程链路，这是构建联网 Agent 的数据基础设施。
  - 🧭 **上手**：用其 playground 或 API 抓一个复杂文档站点，对比原始 HTML 与输出 Markdown 的差异。
- [Leonxlnx/taste-skill（Agent Skills）](https://github.com/Leonxlnx/taste-skill) — *GitHub · JavaScript · +10.8k/周 · 总 98.4k star*
  - 📌 **是什么**：一组让编程 Agent 具备设计品味的 skill，通过美术指导、参考图和浏览器检查来避免生成千篇一律的 AI 风格页面。
  - 💡 **学习点**：前端工程师可重点学习如何把设计审美规范编码成 Agent 指令，让 LLM 生成高质量 UI 而非模板化页面。
  - 🧭 **上手**：把该 skill 接入你的 Cursor/Claude Code，生成一个 landing page 并与默认输出做视觉对比。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步防递归
- **技能点**：掌握了 Vue 多状态经数组中转双向同步时防死循环的守卫手法，能独立定位并修复 'Maximum recursive updates exceeded' 类报错。
- **坑点**：标志位→sizes 数组（新引用）→deep watch 回写标志位，即使值未变，引用变化也会互相唤醒 watch，形成无限递归。
- **解决方案**：每个同步方向都先做值比较再写：源侧算出 next 与当前逐项相等则 return；deep watch 内对派生值先判 !== 再回写。
```text
const syncSizesFromFlags = () => {
  const next = [leftWidth, 'auto', rightWidth]
  const cur = localPanelSizes.value
  if (next[0] === cur[0] && next[1] === cur[1] && next[2] === cur[2]) return
  localPanelSizes.value = next
}
watch(localPanelSizes, (sizes) => {
  const folded = sizes[0] === 0
  if (isFilePreviewFolded.value !== folded) isFilePreviewFolded.value = folded
}, { deep: true })
```
- **拓展**：可沉淀为 useSyncedState 之类的通用 composable；更优解是消除双向同步、单一数据源。
- *来源：admin-workspace-new 2026-08-11*

### 2. 共享组件保持通用，特例在业务侧处理
- **技能点**：学会在改共享组件 vs 改业务调用点之间做正确取舍：共享组件保持最小通用契约，业务特需（如阻止冒泡）在调用方用修饰符解决。
- **坑点**：CopyText 点击冒泡触发外层卡片展开/收起；若给共享 copyText.vue 加 stopPropagation 会污染所有使用方。
- **解决方案**：回退共享组件改动，仅在业务组件模板加 @click.stop；同时用 copyText prop 区分显示文本与实际复制内容。
```text
<CopyText :text="`题目编号: ${data.questionId}`" :copyText="data.questionId" @click.stop />
```
- **拓展**：评估改动先问『这是通用能力还是业务特例』，特例一律下沉到调用方。
- *来源：admin-workspace-new 2026-08-11*

### 3. Quill 工具栏按钮统一注册
- **技能点**：掌握了把编辑器工具栏按钮收敛为『常量 + 联合类型 + 无条件内置 handler』的注册机制设计，业务侧仅凭名字即可选用。
- **坑点**：公式/填空等按钮此前必须配 customButtons 才能用，业务组件不知道 toolbar-order 可传哪些名，易漏配。
- **解决方案**：导出 TOOLBAR_BUTTONS 常量与 ToolbarButtonName 类型；buildToolbarHandlers 对 formula/image 无条件注册内置 handler；样式层自动写 SVG 图标与 title。
```text
export const TOOLBAR_BUTTONS = { undo: 'undo', formula: 'formula', fillInInsert: 'fillInInsert' } as const
export type ToolbarButtonName = typeof TOOLBAR_BUTTONS[keyof typeof TOOLBAR_BUTTONS]
// toolbarOrder: (ToolbarButtonName | string)[]
```
- **拓展**：同类可配置项（菜单、操作列）都适用『常量注册表 + 类型约束』模式。
- *来源：admin-workspace-new 2026-08-11*

### 4. 面板显隐用 v-if 而非尺寸魔法值
- **技能点**：巩固了『结构显隐走声明式渲染、尺寸归尺寸』的职责分离意识，精简掉持久化/双向同步等过度设计。
- **坑点**：此前用 sizes 数组里的 "hidden" 字符串控制右栏显隐，把显隐语义混进尺寸状态，直接催生双向 watch 同步与死循环。
- **解决方案**：右栏改用 <template v-if="isAiReviewer" #panel-2>；删除 storageKey 持久化、折叠标志 ref 与全部同步 watch，localPanelSizes 仅作拖拽内部状态。
```text
<ResizablePanels v-model:sizes="localPanelSizes">
  <template v-if="isAiReviewer" #panel-2>
    <AiCheckContent />
  </template>
</ResizablePanels>
```
- **拓展**：状态设计先问『这个值的单一职责是什么』，一个字段承载两种语义往往是后续 bug 之源。
- *来源：admin-workspace-new 2026-08-11*

### 5. 重构分支合并主干策略
- **技能点**：掌握了「一方做过全量 prettier 的分支合并主干」的正确路径，避免镜像格式化分支法放大冲突。
- **坑点**：把主干也格式化后再 merge（镜像分支法）实测冲突反而从 51 文件/118 块涨到 117 文件/414 块：合并 base 是未格式化版，两侧格式化基于不同代码行布局错位，还把『仅我方改』的文件变成双方都改。
- **解决方案**：直接 git merge 主干 → 冲突处整体采纳主干版 → npm run format 统一排版 → 按清单 grep 驱动地在主干基线上重做语义成果。
```text
git merge master
# 冲突文件整体采纳主干版后
npm run format
# 再按语义改动清单逐文件 grep 核对、重做
```
- **拓展**：全量格式化这类噪声变更应与语义变更严格分离提交，否则合并、review、回溯都付出倍数代价。
- *来源：admin-workspace-hr MEMORY.md（2026-10-14 实战）*

### 6. 线上现象≠本地代码的排查
- **技能点**：建立了『现象与代码对不上时先怀疑部署版本』的排查直觉，会用 git show 直接对比部署分支代码。
- **坑点**：手机扫码上传报 No static resource，本地怎么改都无效；实因 v2 迁移全在未提交工作区，测试环境跑的是 master 旧码。
- **解决方案**：用 git show master:<文件> 与本地对比，确认部署分支实际内容后再排根因；修复先提交合并，不在工作区空转。
```text
git show master:src/service/upload.ts
```
- **拓展**：同类套路：环境变量、构建缓存、CDN 缓存也应列入『现象≠代码』的排查清单。
- *来源：admin-workspace-hr MEMORY.md（2026-09-24）*

