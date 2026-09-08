---
title: "工作台日报 · 2026-09-09"
date: 2026-09-09 07:02:15
categories: [工作日记]
tags: ["日报", "AI工具", "隐私", "AI", "Web"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-09

## 🔥 行业热点

- [Muse: Meta's personal AI agent, features and capabilities](https://ai.meta.com/muse/) — *Hacker News*
  - 📌 **内容**：Meta推出个人AI代理Muse，重点展示其功能与应用场景。
  - 💡 **学习**：可关注个人AI代理的交互设计、记忆机制与工具调用能力。
  - 🧭 **拓展**：可将Muse的能力与ChatGPT、Claude等代理功能做对比评测。
- [Mistral raises €3B](https://mistral.ai/news/mistral-makes-sovereign-open-weight-ai-to-frontier/) — *Hacker News*
  - 📌 **内容**：法国AI公司Mistral完成新一轮约30亿欧元融资，继续加码大模型研发。
  - 💡 **学习**：关注欧洲AI头部公司的融资节奏与开源/闭源产品策略。
  - 🧭 **拓展**：可跟踪其模型发布并对比性能。
- [I-have-ADHD: A skill to stop coding agents from burying the answer](https://github.com/ayghri/i-have-adhd) — *Hacker News*
  - 📌 **内容**：一个名为I-have-ADHD的skill，旨在让编码Agent避免把关键答案埋在长篇输出里。
  - 💡 **学习**：可以通过定制技能/提示词约束AI代理的输出结构，提升信息获取效率。
  - 🧭 **拓展**：可自行实现类似规则并在日常Agent工作流中验证。
- [Cognition (Devin) raises $2B at $48B valuation](https://cognition.com/blog/series-e) — *Hacker News*
  - 📌 **内容**：AI编程代理公司Cognition完成20亿美元融资，估值达480亿美元（标题所示）。
  - 💡 **学习**：关注AI编程代理的商业模式与估值逻辑，理解Agent产品如何形成护城河。
  - 🧭 **拓展**：可试用Devin并观察其自动化编程任务的实际表现。
- [GrapheneOS on AI Usage](https://grapheneos.social/@GrapheneOS/117236529351603001) — *Hacker News*
  - 📌 **内容**：GrapheneOS针对AI使用发表观点，强调移动端隐私与安全考量。
  - 💡 **学习**：了解在Android生态中如何隔离AI功能并保护用户数据。
  - 🧭 **拓展**：可参考其隐私建议配置自己的设备。
- [Google DeepMind Releases AlphaGenome Atlas](https://blog.google/innovation-and-ai/models-and-research/google-deepmind/alphagenome-atlas/) — *Hacker News*
  - 📌 **内容**：Google DeepMind发布AlphaGenome Atlas，提供人类基因组预测图谱。
  - 💡 **学习**：学习AI在基因组数据建模中的方法，以及大规模生物学数据集的构建思路。
  - 🧭 **拓展**：可访问Atlas探索变异数据并了解其建模方式。
- [LG TVs caught spying even when offline or on standby](https://www.theverge.com/tech/991190/lg-tv-spying-standby-recording-wi-fi-scanning-gamers-nexus) — *Hacker News*
  - 📌 **内容**：报道称LG电视即使在离线或待机状态下也会收集数据，引发隐私争议。
  - 💡 **学习**：关注IoT设备后台数据采集的实现方式与隐私合规设计。
  - 🧭 **拓展**：可用网络抓包工具验证自己家智能设备的上传行为。
- [DaVinci Resolve 21.1](https://www.blackmagicdesign.com/media/release/20260908-03) — *Hacker News*
  - 📌 **内容**：专业视频剪辑软件DaVinci Resolve发布21.1版本更新。
  - 💡 **学习**：可了解专业视频软件的功能迭代方向，学习其中涉及的色彩/渲染技术。
  - 🧭 **拓展**：可下载更新体验新流程。
- [ChatGPT Images 2.5](https://openai.com/index/introducing-chatgpt-images-2-5/) — *Hacker News*
  - 📌 **内容**：ChatGPT的图像生成能力更新至2.5版本，带来新的出图体验。
  - 💡 **学习**：关注多模态生成模型的版本演进，学习图像编辑/生成提示词技巧。
  - 🧭 **拓展**：可对比测试其与Midjourney、DALL·E等的差异。
- [Antiquated HTML Snippets and Artefacts](https://vale.rocks/posts/html-relics) — *Hacker News*
  - 📌 **内容**：收集展示早期HTML代码片段与Web历史遗物。
  - 💡 **学习**：从历史代码中理解Web标准演进与浏览器兼容性设计。
  - 🧭 **拓展**：可查看对应代码仓库并尝试在旧浏览器模拟器中运行。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步循环守卫
- **技能点**：掌握 watch 双向同步必须做值比较守卫，避免数组引用变化 + deep watch 造成无限递归。
- **坑点**：状态 A → 数组 → 状态 B → 状态 A 的同步链中，即使最终值未变，新数组引用仍会触发 deep watch 回写标志，互相唤醒直至 Maximum recursive updates exceeded。
- **解决方案**：每次写入前比较新值与当前值，相等则直接 return；标志回写同样先比较，确保无变化不触发。
```text
watch([foldA, foldB], () => {
  const next = [left, 'auto', right];
  if (next.every((v, i) => v === localPanelSizes.value[i])) return;
  localPanelSizes.value = next;
});
watch(localPanelSizes, (sizes) => {
  const nextFold = derive(sizes[0]);
  if (nextFold !== foldA.value) foldA.value = nextFold;
}, { deep: true });
```
- **拓展**：可推广到任何响应式双向同步场景，优先用派生状态或单向数据流减少同步链路。
- *来源：admin-workspace-new / 2026-08-11.md*

### 2. 按需注册组件需显式引入 CSS
- **技能点**：学会排查 UI 库按需引入导致样式缺失的问题，理解组件注册与样式加载是两回事。
- **坑点**：Element Plus 按需 resolver 自动注册组件但不自动补全较新/较冷组件的 CSS，导致 splitter 等退化为 block 流式布局，方向/拖拽全部失效。
- **解决方案**：显式 import 所需组件 CSS（如 el-splitter.css、el-splitter-panel.css）；排查时先看根容器方向，再看 CSS 是否真的被加载。
```text
import 'element-plus/theme-chalk/el-splitter.css';
import 'element-plus/theme-chalk/el-splitter-panel.css';
```
- **拓展**：任何按需加载场景（antd 按需 CSS 等）都要注意样式缺失，可将所需 CSS 清单沉淀到组件文档。
- *来源：admin-workspace / MEMORY.md*

### 3. innerHTML 中裸 `<` 的解析陷阱
- **技能点**：学会在把含特殊字符的文本注入 innerHTML 前进行转义，理解浏览器 HTML5 解析器的容错行为。
- **坑点**：v-katex 渲染时 LaTeX 含裸 `<b<` 等，直接 el.innerHTML = html 使浏览器把 `<` 当标签开始，破坏闭合导致后续内容被吞并进同一公式节点。
- **解决方案**：innerHTML 赋值前用正则匹配标签内容并转义 `<`/`>` 为实体；读取时用 textContent 自动解码回原始 latex。
```text
html = html.replace(/(<question-latex\b[^>]*>)([\s\S]*?)(<\/question-latex>)/gi,
  ($0, open, body, close) => open + body.replace(/</g, '&lt;').replace(/>/g, '&gt;') + close);
el.innerHTML = html;
```
- **拓展**：任何富文本/公式渲染场景都需考虑特殊字符注入，也可用 DOM API 构建或 textContent 赋值代替 innerHTML 拼接。
- *来源：admin-workspace-new / katex.js*

### 4. 匹配前统一归一化（去壳）
- **技能点**：掌握在字符串匹配/替换前对数据进行归一化（去标签壳、去定界符），提高匹配鲁棒性。
- **坑点**：后端返回的 original 带 `<question-latex>` 或 `$` 定界符，而正文已被转换为 `<formula data-formula>`，字面匹配全部失败（matchCount=0）。
- **解决方案**：编写 normalizeTypoOriginal() 剥掉标签壳与首尾 `$`，在 doMark 与 replace 中统一使用归一后的字符串匹配，并用临时脚本复用真实函数验证 matchCount。
```text
const normalizeTypoOriginal = (original: string) =>
  original.replace(/<\/?question-latex>/gi, '').replace(/^\$|\$$/g, '').trim();
```
- **拓展**：适合所有前后端数据格式不一致的场景，可沉淀为通用 normalize 工具，或用模糊匹配增强鲁棒性。
- *来源：admin-workspace-new / useTypoReplace.ts*

### 5. antd 版本升级 API 迁移
- **技能点**：掌握库升级时废弃 API 的系统替换方法，能根据官方类型定义和新属性迁移。
- **坑点**：v5→v6 中 bodyStyle、destroyOnClose、Drawer width、Space direction 等废弃，直接写旧属性无报错但功能不稳定；且部分属性名（如 Anchor 的 direction）仍合法，全局替换会误伤。
- **解决方案**：建立迁移清单逐项替换为 styles={{body}}、destroyOnHidden、size={500}、orientation="vertical"；全局替换前排除仍合法用法，用 npx tsc --noEmit 验证。
```text
<Modal centered destroyOnHidden styles={{ body: { maxHeight: 'calc(100vh - 160px)', overflowY: 'auto' } }} />
<Drawer size={500} destroyOnHidden styles={{ body: { ... } }} />
<Space orientation="vertical">...</Space>
```
- **拓展**：每次依赖大版本升级前先读 release notes + 类型定义，可沉淀为 codemod 脚本或 lint 规则。
- *来源：admin-workspace-hr / MEMORY.md*

### 6. 自定义模块覆盖后补齐默认能力
- **技能点**：学会扩展库默认模块时，识别并补全被覆盖的默认 matcher/handler，避免功能静默丢失。
- **坑点**：Quill 中用 TableClipboard 接管 modules/clipboard，但它未继承默认 clipboard 的 image/divider matcher，导致 convert({html}) 时图片/分割线被丢弃，编辑回显丢失内容。
- **解决方案**：在 registerClipboardMatchers 中显式添加 img[data-type="ql-image"]、divider.ql-divider 等 matcher，返回值结构须与对应 Blot.value 对齐。
```text
clipboard.addMatcher('img[data-type="ql-image"]', (node) => {
  const el = node as HTMLElement;
  return new Delta().insert({ image: { url: el.getAttribute('src'), alt: el.getAttribute('alt'), width: el.getAttribute('width'), height: el.getAttribute('height'), style: el.getAttribute('style') } });
});
clipboard.addMatcher('divider.ql-divider, p div hr.ql-divider, p div div.ql-divider', (node) => {
  const el = node as HTMLElement;
  return new Delta().insert({ divider: { dataType: el.getAttribute('data-type'), style: el.getAttribute('style') } });
});
```
- **拓展**：任何“继承并覆盖”默认实现（如自定义 paste 处理器、中间件）都要考虑默认行为清单，并做 round-trip 回归验证。
- *来源：admin-workspace-new / QuillEditorNew MEMORY.md*

