---
title: "工作台日报 · 2026-09-05"
date: 2026-09-05 07:14:02
categories: [工作日记]
tags: ["日报", "AI", "AI搜索", "Agent", "Rails"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-05

## 🔥 行业热点

- [Discovery of a new OpenAI agent message board](https://collusion.wiki/) — *Hacker News*
  - 📌 **内容**：发现了一个新的 OpenAI Agent 留言板/讨论区，可能是社区和产品信息的聚合入口。
  - 💡 **学习**：可以关注 Agent 生态社区，了解开发者对 OpenAI Agent 产品的讨论和最新实践。
  - 🧭 **拓展**：可以订阅该信息源并筛选高质量线索，跟踪 Agent 工具链演进。
- [Google AI Mode shows same products 21.6% more expensive than traditional search](https://productrise.app/blog/google-ai-mode-prefers-more-expensive-products) — *Hacker News*
  - 📌 **内容**：一项对比显示 Google AI Mode 展示的同一商品价格比传统搜索更贵，引发对 AI 搜索结果可靠性及商业模式的讨论。
  - 💡 **学习**：在 AI 搜索或推荐系统中需要对结构化商品数据进行一致性校验，避免输出偏差。
  - 🧭 **拓展**：可以设计实验对比不同搜索引擎的商品抽取与排序质量。
- [Corporate America is getting hooked on open-source AI](https://www.nytimes.com/2026/09/04/technology/open-source-ai-anthropic-openai.html) — *Hacker News*
  - 📌 **内容**：文章讨论美国企业越来越依赖开源 AI 模型，反映开源模型在企业应用中的主流化趋势。
  - 💡 **学习**：掌握开源模型的部署、微调与合规评估已成为企业 AI 工程师的重要技能。
  - 🧭 **拓展**：可以研究主流开源模型在企业场景中的授权条款与落地模式。
- [Government Rails Site Hit Hours After CVE Patch](https://rietta.com/blog/ruby-on-rails-cve-exploited-hours-after-patch/) — *Hacker News*
  - 📌 **内容**：一个政府 Rails 网站在补丁发布后数小时即遭攻击，说明漏洞利用速度远超预期。
  - 💡 **学习**：需要建立自动化补丁验证和应急响应流程，尤其要优先修复已知被利用漏洞。
  - 🧭 **拓展**：可以搭建 Rails 安全监控和 WAF 规则进行演练。
- [Formalizing Fermat's Last Theorem](https://www.anthropic.com/research/formalizing-fermats-last-theorem) — *Hacker News*
  - 📌 **内容**：内容围绕用形式化方法证明费马大定理，体现机器证明在纯数学中的进展。
  - 💡 **学习**：可以了解 Lean/Coq 等证明助手如何把复杂证明转化为可验证的形式化代码。
  - 🧭 **拓展**：可以尝试用 Lean 形式化一个简单的数论定理。
- [Solving the Jane Street reverse engineering challenge](https://jestoph.com/2026/09/04/jane-street-challenge.html) — *Hacker News*
  - 📌 **内容**：一篇攻克 Jane Street 逆向工程挑战的解题分享，涉及二进制分析、调试与协议还原。
  - 💡 **学习**：可以学习逆向工程与调试技能，提升对底层系统和漏洞机制的理解。
  - 🧭 **拓展**：可尝试复现该挑战并编写自己的解题脚本。
- [Show HN: Open-Source eInk Bike Computer](https://opentrailpaper.com) — *Hacker News*
  - 📌 **内容**：一个开源电子墨水自行车电脑，主打低功耗、高可读性和可定制性。
  - 💡 **学习**：可以学习 eInk 屏幕驱动、低功耗嵌入式开发和开源硬件设计。
  - 🧭 **拓展**：可参考其源码移植到不同开发板。
- [The Rust React Compiler is now native in Vite](https://blog.master.dev/react-now-rusted-all-the-way-out/) — *Hacker News*
  - 📌 **内容**：React 编译器的 Rust 实现已原生集成到 Vite，有望显著优化前端构建性能。
  - 💡 **学习**：可以关注 Rust 在前端工具链中的应用，学习 Vite 插件与编译器集成方式。
  - 🧭 **拓展**：可在现有 Vite 项目中开启该能力并对比构建耗时。
- [The Two Abstractions of System Design: Hide or Reduce](http://muratbuffalo.blogspot.com/2026/05/the-two-abstractions-of-system-design.html) — *Hacker News*
  - 📌 **内容**：文章提出系统设计中的两种抽象策略：隐藏复杂度或减少复杂度，并讨论其取舍。
  - 💡 **学习**：设计系统时既可以用接口隐藏内部细节，也可以从源头降低组件间的复杂度。
  - 🧭 **拓展**：可复盘当前项目的抽象层次，识别需要隐藏或消除的复杂度。
- [Show HN: TERMy – A fast terminal assistant that does not use LLMs](https://github.com/gioblu/NPC-Forge/blob/main/docs/development.md) — *Hacker News*
  - 📌 **内容**：TERMy 是一款不使用 LLM 的快速终端助手，说明基于规则或脚本的轻量自动化仍有价值。
  - 💡 **学习**：可学习如何以非 AI 方式构建即时响应的 CLI 工具，降低资源消耗。
  - 🧭 **拓展**：可阅读其源码，研究终端 UI 与命令补全实现。

## 🚀 技能提升点（工作总结汇总）

### 1. watch 双向同步防循环
- **技能点**：掌握 Vue watch 双向同步（状态→数组→状态）每步做值比较守卫，杜绝无限递归。
- **坑点**：两个 watch 互相唤醒：即使最终值未变，新数组引用 + deep watch 也会持续触发，直至报 Maximum recursive updates。
- **解决方案**：同步函数先计算 next 再与当前值全等比较，无变化直接 return；回写前同样先比较当前值与推导值。
```text
const syncSizesFromFlags = () => {
  const next = [leftWidth, 'auto', rightWidth];
  if (next.every((v, i) => v === localPanelSizes.value[i])) return;
  localPanelSizes.value = next;
};
watch(localPanelSizes, (v) => {
  const nextFlag = deriveFlag(v);
  if (nextFlag !== isFolded.value) isFolded.value = nextFlag;
}, { deep: true });
```
- **拓展**：可抽公共 shallowEqual 守卫，用于所有 watch 同步链。
- *来源：admin-workspace | 2026-08-11*

### 2. Quill 内容同步用 getHTML
- **技能点**：掌握 vue-quill 中 Delta 与 HTML 的区分：同步 v-model 必须取真实 HTML。
- **坑点**：getContents() 返回 Delta 对象，被当 HTML 字符串交给 DOMParser 后 modelValue 被污染成 Delta JSON，导致高亮/替换全部失配。
- **解决方案**：改用 quillRef.value?.getHTML?.() ?? '' 取 Quill innerHTML 再处理。
```text
const html = quillRef.value?.getHTML?.() ?? '';
processHtmlContent(html);
```
- **拓展**：镜像、回显、快照等所有 Quill 内容提取统一用 getHTML。

### 3. LaTeX 尖括号实体化
- **技能点**：掌握 HTML 内嵌 LaTeX 时 < > 必须实体化，避免被 HTML5 解析器误判为标签。
- **坑点**：latex 中裸 <（如 0<b<1）会破坏 question-latex 闭合，v-katex 渲染时大量内容被吞入公式节点。
- **解决方案**：innerHTML 之前把 question-latex 内容里的 < > 替换为 &lt; &gt;，用 textContent 读取时自动还原。
```text
html = html.replace(/(<question-latex\b[^>]*>)([\s\S]*?)(<\/question-latex>)/gi,
  (_, open, body, close) => open + body.replace(/</g, '&lt;').replace(/>/g, '&gt;') + close);
```
- **拓展**：data-formula 属性值同理；拍平解析需引号感知扫描标签结束。
- *来源：admin-workspace-new | MEMORY.md*

### 4. 文本匹配容空白差异
- **技能点**：掌握用容空白正则匹配 AI 生成文本与正文的空白差异，保证高亮/替换命中。
- **坑点**：AI 返回的 latex 无空格、正文 latex 带空格，严格字面匹配永远失配且无明显报错。
- **解决方案**：先严格匹配，未命中再用字符间允许 \s* 的正则重试，并剔除首尾空白。
```text
const parts = original.trim().split(/\s+/).map(escapeRegExp);
const tolerantRegex = new RegExp(parts.join('\\s*'));
if (!strictMatched) applyHighlight(tolerantRegex);
```
- **拓展**：可扩展为忽略 LaTeX 定界符/转义差异的归一匹配。

### 5. antd Sticky 配 scroll.x
- **技能点**：掌握 antd Table sticky 必须配合 scroll.x，避免表头与 body 错位。
- **坑点**：只开启 sticky 时，大容器下 body 右侧补占位列宽但 sticky 表头不补，导致右列错位。
- **解决方案**：启用 sticky 时同时声明 scroll={{ x: 'max-content' }} 或固定数值。
```text
<Table sticky scroll={{ x: 'max-content' }} />
```
- **拓展**：可作为 Table 使用规范写入 Code Review 检查清单。
- *来源：admin-workspace-hr | MEMORY.md*

### 6. HTML 实体解码与区间映射
- **技能点**：掌握富文本拍平时对 HTML 实体整体解码并维护区间映射，保证高亮/替换不拆实体。
- **坑点**：逐字符解码实体失败导致选项漏标；区间命中实体时直接截断会把 &lt; 切成 & 和 lt;。
- **解决方案**：累积文本节点后整体 decodeHtmlEntities，返回 entityRanges；命中实体时片段边界延伸到实体末尾。
```text
const { text, entityRanges } = decodeHtmlEntitiesWithRanges(rawText);
const segments = mapPlainRangeToTextSegments(start, end, entityRanges);
// 命中实体时 end 对齐 entityRanges[i].end
```
- **拓展**：该模式可复用于选区、替换、富文本 diff 的偏移映射。

