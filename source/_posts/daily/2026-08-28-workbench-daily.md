---
title: "工作台日报 · 2026-08-28"
date: 2026-08-28 06:55:48
categories: [工作日记]
tags: ["日报", "AI工具", "大模型", "AI Agent", "开源"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-08-28

## 🔥 行业热点

- [AI Engineer Notebooks – free, framework-free RAG/agents/evals on Colab](https://github.com/calmrocks/ai-engineer-notebooks) — *Hacker News*
  - 📌 **内容**：一套面向AI工程师的免费Notebook，提供无需框架的RAG、Agent与评估示例，可直接在Colab运行。
  - 💡 **学习**：学习不依赖框架构建RAG/Agent的方法，便于理解底层原理并自定义实现。
  - 🧭 **拓展**：可在Colab中逐个运行Notebook，再用自己的数据和模型验证效果。
- [CEO fired developers to make room for AI. Developers create open source AI CEO](https://github.com/SenteLabsAI/OpenExecutive) — *Hacker News*
  - 📌 **内容**：面对以AI替代开发者的公司，开发者用开源方式做了一个“AI CEO”作为回应。
  - 💡 **学习**：可以看到开发者如何把职位与工作流抽象为AI Agent，并以开源方式进行协作。
  - 🧭 **拓展**：可研究其Agent工作流设计，尝试复刻一个自动化的“老板”角色。
- [CMS with AI, Not AI CMS: Wagtail 8.0's New API](https://wagtail.org/blog/cms-with-ai-not-ai-cms-wagtail-80s-new-api/) — *Hacker News*
  - 📌 **内容**：Wagtail 8.0通过新的API把AI能力嵌入CMS，而不是把CMS变成“AI CMS”。
  - 💡 **学习**：关注CMS如何以可插拔方式集成AI能力，同时保持内容系统的稳定。
  - 🧭 **拓展**：可以阅读Wagtail 8.0 API文档，尝试在项目中接入其AI扩展点。
- [Show HN: A lightweight, stateless database for agent memory](https://polign.com/blog-edge-agent-memory) — *Hacker News*
  - 📌 **内容**：这是一个面向Agent记忆的轻量级、无状态数据库，重点突出简单与可移植。
  - 💡 **学习**：了解无状态设计如何简化Agent记忆的读写与水平扩展。
  - 🧭 **拓展**：可对比其他向量记忆库，测试它在不同Agent场景下的表现。
- [Nvidia agrees to acquire Hugging Face for $13B](https://www.businessinsider.com/nvidia-in-talks-to-buy-hugging-face-13-billion-dollars-2026-8) — *Hacker News*
  - 📌 **内容**：NVIDIA宣布以约130亿美元收购Hugging Face，AI基础设施与开源模型社区将进一步整合。
  - 💡 **学习**：关注大厂并购后对模型分发、推理平台和开发者工具链的影响。
  - 🧭 **拓展**：可跟踪Hugging Face平台后续的GPU集成和模型服务变化。
- [Tell HN: PayPal blocks GrapheneOS](https://news.ycombinator.com/item?id=49462253) — *Hacker News*
  - 📌 **内容**：用户在HN上反馈PayPal屏蔽GrapheneOS用户，涉及支付平台对定制系统的识别策略。
  - 💡 **学习**：了解移动支付风控中的设备指纹与兼容性机制，以及开源系统遇阻后的应对方案。
  - 🧭 **拓展**：可尝试从Web端或测试环境评估类似风控规则的影响。
- [Saving 100 terabytes of memory by optimizing 1.1.1.1's DNS cache](https://blog.cloudflare.com/dns-cache-memory-optimization-1111/) — *Hacker News*
  - 📌 **内容**：团队通过优化1.1.1.1的DNS缓存实现节省约100TB内存。
  - 💡 **学习**：学习DNS缓存的数据结构设计、TTL策略与内存占用优化技巧。
  - 🧭 **拓展**：可参考其思路分析自己服务的缓存命中率与内存模型。
- [Small Models Have Arrived](https://calv.info/small-models-have-arrived) — *Hacker News*
  - 📌 **内容**：文章指出小模型已经达到可用水平，成为大模型之外的重要选择。
  - 💡 **学习**：了解小模型在部署成本、延迟和隐私上的优势，以及如何匹配任务。
  - 🧭 **拓展**：可尝试用小型模型跑本地应用，对比与API大模型的效果。
- [Show HN: The load-bearing vocabulary of Claude](https://louisabraham.github.io/load-bearing/) — *Hacker News*
  - 📌 **内容**：项目分析Claude输出中的“承重词汇”，即对生成结果至关重要的高频或关键表达。
  - 💡 **学习**：可通过词汇统计与消融实验理解模型输出的关键语义依赖。
  - 🧭 **拓展**：用自己的提示词样本复现分析，找出影响输出质量的关键词。
- [Gemini Omni 1.1 Flash](https://blog.google/innovation-and-ai/technology/developers-tools/build-with-gemini-omni-1-1-flash/) — *Hacker News*
  - 📌 **内容**：这是Gemini系列中名为Omni 1.1 Flash的新模型/版本发布。
  - 💡 **学习**：关注新模型在速度、多模态和价格上的变化，评估其在应用中的可行性。
  - 🧭 **拓展**：可申请试用并测试与现有模型的替换成本。

## 🚀 技能提升点（工作总结汇总）

### 1. Quill 内容同步：getHTML 而非 getContents
- **技能点**：掌握编辑器内容序列化 API 差异：vue-quill 的 getContents() 返回 Delta 对象，getHTML() 才返回 HTML 字符串；同步 v-model 必须用 HTML。
- **坑点**：原代码 quillRef.getContents().trim() 将 Delta JSON 当 HTML 传给 DOMParser，导致 v-model 被 Delta JSON 串污染，后续公式/高亮匹配全部失配。
- **解决方案**：改为 getHTML() ?? ''（或 quill.root.innerHTML）取真实 Quill innerHTML，包含 data-formula 等属性；如必须用 Delta，则显式序列化再同步。
```text
const html = quillRef.value?.getHTML?.() ?? quillRef.value?.root?.innerHTML ?? '';
```
- **拓展**：可抽象为“编辑器与外部数据统一走 HTML 或统一走 Delta，不混用”的约定，在富文本组件文档中沉淀。
- *来源：admin-workspace 2026-08-12*

### 2. HTML 属性中裸 < > 破坏标签解析
- **技能点**：学会处理 HTML 属性内包含 < > 的解析问题：标签结束不能简单按 indexOf('>') 定位，需引号感知；注入属性前应先实体化。
- **坑点**：data-formula 属性值含原始 > 时，html.indexOf('>') 会把属性值内部的 > 当成标签结束，导致公式内容在解析后丢失；latex 中裸 < 在 innerHTML 赋值时还被浏览器当作标签开始，吞并后续内容。
- **解决方案**：标签结束定位改为引号感知扫描（< 后引号内跳过，直到引号外遇到真正的 >）；写入 data-formula 前先将 latex 中 < > 转换为 &lt;/&gt; 实体。
```text
function findTagEnd(html, start) {
  let inQuote = false, quote = '';
  for (let i = start; i < html.length; i++) {
    const c = html[i];
    if (inQuote) { if (c === quote) inQuote = false; }
    else if (c === '"' || c === "'") { inQuote = true; quote = c; }
    else if (c === '>') return i;
  }
  return -1;
}
```
- **拓展**：可沉淀为 HTML 与文本互转的公共工具函数，覆盖属性值提取、实体化与解析，供所有 HTML 拍平/匹配场景复用。
- *来源：admin-workspace 2026-08-12*

### 3. Vue watch 双向同步防递归循环
- **技能点**：巩固 Vue watch 双向同步防循环能力：每次回写前比较值是否变化，避免“写入新引用→deep watch→再写”的死循环。
- **坑点**：两个 watch（flags→数组、数组→flags）互相触发，即使最终值未变也写新数组引用/回写标志，最终触发 Maximum recursive updates exceeded。
- **解决方案**：syncSizesFromFlags 计算 next 后逐项比较相同则直接 return；数组 watch 内对每个标志先比较再决定是否回写，双向同步每一步加值守卫。
```text
const next = [leftWidth, 'auto', rightWidth];
if (next.every((v, i) => v === localPanelSizes.value[i])) return;
localPanelSizes.value = next;
```
- **拓展**：可总结为“跨 ref 状态同步规则”：所有 watch 写入前必须比较新旧值，杜绝无变化回写。
- *来源：admin-workspace 2026-08-11*

### 4. 工具栏配置数组需 flatMap 展平
- **技能点**：掌握配置映射时的展平原则：当映射函数返回数组时调用方应使用 flatMap，避免嵌套数组被下游误当对象处理。
- **坑点**：getDefaultButtonConfig 的 list/indent 返回数组，order.map 后数组项被 Quill 的 addControls 当作对象，生成 class="ql-0" 的空按钮。
- **解决方案**：将 order.map(...) 改为 order.flatMap(...)，多按钮数组展平为一维 controls，每个 {list:'ordered'} 独立注册，Quill 才能正常识别。
```text
const controls = order.flatMap(name => {
  const cfg = getDefaultButtonConfig(name);
  return Array.isArray(cfg) ? cfg : [cfg];
});
```
- **拓展**：可推广到任何“配置映射产生嵌套数组”的前端配置流水线，加一个 flatMap 规约并配单测。
- *来源：admin-workspace-new 2026-08-12*

### 5. 跨版本克隆文件先审目标内容
- **技能点**：提升跨版本/跨目录文件操作的前置检查意识：克隆文件前先确认目标文件已有导出与语义，而非直接覆盖。
- **坑点**：直接用旧版 type.ts 覆盖新版同名文件，导致新版本依赖的 QuillEditorProps 等完整类型声明全部丢失，编译报 Unresolvable type reference。
- **解决方案**：用 git show 对比目标文件，保留新版原始类型声明，只追加旧版需要的工具类型/函数；注意 named export 差异（如 Quill 在 quill-2 下需 default import）。
```text
git show HEAD:new/type.ts   # 先读目标文件已有结构
git show HEAD:old/type.ts   # 再决定追加还是覆盖
```
- **拓展**：可推广到目录合并/文件迁移场景：先 diff 目标文件语义，必要时用 patch 合并而非整体覆盖。
- *来源：admin-workspace-new 2026-08-12*

### 6. 覆盖 clipboard 需补齐默认 matcher
- **技能点**：掌握 Quill clipboard 覆盖的完整模式：自定义 clipboard 模块不会继承默认 matcher，必须显式补齐所有非表格 embed 的解析。
- **坑点**：注册 TableClipboard 后 clipboard.convert({html}) 遇到 img/divider 没有 matcher，图片与分割线被丢弃，只有基础 block 保留。
- **解决方案**：在 registerClipboardMatchers 中显式 addMatcher 处理 img[data-type="ql-image"] 与 divider.ql-divider，并让返回 Delta 与对应 Blot.value 字段对齐。
```text
clipboard.addMatcher('img[data-type="ql-image"]', node => {
  const { url, alt, title, width, height, style } = node.dataset;
  return new Delta().insert({ image: { url, alt, title, width, height, style } });
});
clipboard.addMatcher('divider.ql-divider', node =>
  new Delta().insert({ divider: { dataType: node.dataset.dataType, style: node.getAttribute('style') } })
);
```
- **拓展**：可延伸为“自定义 Quill 模块/Blot 与 clipboard 的注册清单”，新增 embed 时同步补 matcher。
- *来源：admin-workspace-new MEMORY.md*

