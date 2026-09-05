---
title: "工作台日报 · 2026-09-06"
date: 2026-09-06 07:01:43
categories: [工作日记]
tags: ["日报", "大模型", "AI工具", "AI应用", "AI运维"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-06

## 🔥 行业热点

- [Discovery of a new OpenAI agent message board](https://collusion.wiki/) — *Hacker News*
  - 📌 **内容**：发现了一个新的OpenAI智能体留言板，可能用于Agent间交流或社区分享。
  - 💡 **学习**：关注OpenAI Agent生态的新入口，了解如何接入或参与。
  - 🧭 **拓展**：可进一步调研该留言板的访问方式与用途。
- [Can AI design circuit boards yet?](https://eebench.org/blog/can-ai-design-circuit-boards-yet/) — *Hacker News*
  - 📌 **内容**：探讨AI在电路板设计上的能力边界，评估当前进展。
  - 💡 **学习**：了解AI辅助电子设计自动化（EDA）的现状与局限。
  - 🧭 **拓展**：可尝试用开源AI布线工具做简单实验。
- [AI handles incidents, engineers lose touch with their systems](https://www.sylvainkalache.com/blog/ai-handles-incidents-engineers-lose-touch-with-their-systems) — *Hacker News*
  - 📌 **内容**：文章反思AI接管故障处理会让工程师对系统逐渐陌生。
  - 💡 **学习**：在引入AI运维时需保留人工复盘和系统认知。
  - 🧭 **拓展**：可设计AI与人工协作的故障演练流程。
- [Visualizing Rust's Vtables: How dyn Trait Works In Memory](https://sofiabelen.github.io/projects/visualizing-rusts-vtables-how-dyn-trait-works-in-memory/) — *Hacker News*
  - 📌 **内容**：用可视化方式讲解Rust中dyn Trait的虚表（vtable）内存布局。
  - 💡 **学习**：深入理解Rust动态分发的实现细节，有助于性能优化。
  - 🧭 **拓展**：可编写代码打印trait object的地址和布局验证。
- [GPT-6 Astra](https://openai.com/index/gpt-6-astra/) — *Hacker News*
  - 📌 **内容**：关于新一代大模型GPT-6 Astra的消息，但具体细节尚不明确。
  - 💡 **学习**：跟进大模型迭代方向，了解能力升级的潜在影响。
  - 🧭 **拓展**：可等待API开放后做基准测试。
- [Actively exploited sandbox RCE in all Chromium versions](https://nvd.nist.gov/vuln/detail/cve-2026-85046) — *Hacker News*
  - 📌 **内容**：Chromium全版本存在正被利用的沙箱逃逸远程代码执行漏洞。
  - 💡 **学习**：重视浏览器供应链安全，及时更新Chromium内核。
  - 🧭 **拓展**：可检查自己的浏览器和Electron应用版本。
- [Nitter has more working instances than before the takedowns](https://codeberg.org/mv12star/shitter/wiki/Instances) — *Hacker News*
  - 📌 **内容**：Nitter在遭遇封杀后可用实例反而比之前更多。
  - 💡 **学习**：了解开源去中心化服务的抗打击能力。
  - 🧭 **拓展**：可自建Nitter实例体验部署流程。
- [Show HN: Open-Source eInk Bike Computer](https://opentrailpaper.com) — *Hacker News*
  - 📌 **内容**：有人开源了一款电子墨水屏自行车码表。
  - 💡 **学习**：学习低功耗嵌入式设备的硬件与固件设计。
  - 🧭 **拓展**：可查看开源原理图和代码改造复用。
- [GPT-6 Astra on OpenRouter](https://openrouter.ai/openai/gpt-6-astra) — *Hacker News*
  - 📌 **内容**：GPT-6 Astra已出现在OpenRouter平台，可便于统一调用。
  - 💡 **学习**：通过OpenRouter等聚合平台快速体验新模型。
  - 🧭 **拓展**：可比较不同渠道的延迟和成本。
- [Portal by Spotify cut my Claude Code token usage by 90%](https://engineering.atspotify.com/2026/9/portal-by-spotify-cut-my-claude-code-token-usage-by-90) — *Hacker News*
  - 📌 **内容**：Spotify的Portal工具大幅减少了Claude Code的token消耗。
  - 💡 **学习**：探索通过上下文压缩或缓存来优化LLM调用成本。
  - 🧭 **拓展**：可研究Portal的实现思路并应用到自己的工具链。

## 🚀 技能提升点（工作总结汇总）

### 1. Quill Delta 与 HTML 同步
- **技能点**：掌握 Quill 编辑器内容同步时区分 Delta 与 HTML，确保 modelValue 绑定的始终是真实 HTML 字符串。
- **坑点**：vue-quill 的 getContents() 返回 Delta 对象，误用后 modelValue 被 JSON 字符串污染，DOMParser 解析后 `<` 被转义导致匹配失败。
- **解决方案**：改用 getHTML() 或 quill.root.innerHTML 获取真实 innerHTML 作为 v-model 值。
```text
const html = quillRef.value?.getHTML?.() ?? quillRef.value?.root?.innerHTML ?? '';
```
- **拓展**：可沉淀为编辑器封装规范：所有 Quill 组件内容同步一律用 getHTML。

### 2. HTML 属性内裸尖括号解析
- **技能点**：掌握对含内嵌 LaTeX 的 HTML 属性值进行引号感知解析，并养成特殊字符实体化存储的习惯。
- **坑点**：data-formula 等属性值内含原始 `<`/`>`，flattenToPlain 用 indexOf('>') 定位标签结束，属性值里的 `>` 把标签拦腰截断，内容丢失。
- **解决方案**：扫描标签结束改为引号感知：引号内的 `>` 不结束标签；同时数据侧将 latex 中的 `<`/`>` 实体化存储。
```text
function findTagEnd(html, from) {
  let quote = null;
  for (let i = from; i < html.length; i++) {
    const c = html[i];
    if (quote) {
      if (c === quote) quote = null;
    } else if (c === '"' || c === "'") {
      quote = c;
    } else if (c === '>') {
      return i;
    }
  }
  return -1;
}
```
- **拓展**：可抽象为健壮 HTML 拍平/解析工具，并建议数据侧统一实体化。

### 3. Quill Clipboard 基础匹配补齐
- **技能点**：掌握 Quill 自定义 Clipboard 模块时显式注册基础 matcher 的扩展模式，保障图片/分割线等非文本内容不丢失。
- **坑点**：自定义 TableClipboard 接管 clipboard 后未继承默认 Quill clipboard 的 image/divider matcher，clipboard.convert 丢弃这些节点。
- **解决方案**：在 registerClipboardMatchers 中显式 addMatcher，为 img 和 .ql-divider 等补上 Delta 映射。
```text
clipboard.addMatcher('img', node => new Delta().insert({ image: extractImage(node) }));
clipboard.addMatcher('.ql-divider', node => new Delta().insert({ divider: extractDivider(node) }));
```
- **拓展**：自定义 embed blot 接 clipboard 时沿用同样模式显式加 matcher。
- *来源：admin-workspace-new；MEMORY.md*

### 4. Vue watch 双向同步循环守卫
- **技能点**：掌握 Vue ref 间双向同步的值比较守卫技巧，避免 watch 互相触发导致无限递归。
- **坑点**：A→数组→B→A 的同步链每次产生新数组引用触发 deep watch，即使最终值未变也互相唤醒，报 Maximum recursive updates。
- **解决方案**：每条同步路径先比较当前值与推导值，相等则 return；数组整体比对后再赋值。
```text
watch(localPanelSizes, sizes => {
  const next = deriveFlags(sizes);
  if (flagA.value !== next.a) flagA.value = next.a;
  if (flagB.value !== next.b) flagB.value = next.b;
});
```
- **拓展**：可沉淀为通用双向同步模式，适用于 panel 尺寸、折叠状态等联动场景。
- *来源：admin-workspace；2026-08-11*

### 5. LaTeX 空格差异容错匹配
- **技能点**：掌握“先严格匹配、再空白容错匹配”的文本校对策略，能处理 AI 生成文本与正文的空格差异。
- **坑点**：AI 生成的 latex 与正文 latex 空格不一致（original 无空格、正文有空格），用 escapeRegExp(original) 严格匹配永远失配。
- **解决方案**：新增 buildWhitespaceTolerantRegex 允许字符间任意空白，先严格匹配失败后再用容空白正则重试。
```text
function buildWhitespaceTolerantRegex(str) {
  if (!str.trim()) return null;
  return new RegExp(str.split('').map(escapeRegExp).join('\\s*'));
}
```
- **拓展**：可推广到任意用户/AI 文本对拍场景，但需限定只容忍空白、不跨标签。

### 6. Element Plus 按需样式显式引入
- **技能点**：掌握 Element Plus 按需注册（unplugin-vue-components）下组件样式不会自动补齐的问题，能快速定位并显式引入。
- **坑点**：Element Plus 按需注册组件时 el-splitter 等新组件 CSS 未自动加载，布局退化为普通 block 导致堆叠。
- **解决方案**：显式 import 'element-plus/theme-chalk/el-splitter.css' 与 el-splitter-panel.css，不依赖全量/自动引入。
```text
import 'element-plus/theme-chalk/el-splitter.css';
import 'element-plus/theme-chalk/el-splitter-panel.css';
```
- **拓展**：可作为组件库按需注册时的样式审计清单，遇到布局异常先查 CSS 是否被真实加载。
- *来源：admin-workspace；MEMORY.md*

