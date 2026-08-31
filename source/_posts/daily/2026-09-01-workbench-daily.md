---
title: "工作台日报 · 2026-09-01"
date: 2026-09-01 07:20:30
categories: [工作日记]
tags: ["日报", "AI", "AI应用", "大模型", "安全"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-01

## 🔥 行业热点

- [Apple caught off guard by AI demand for Mac Mini and Mac Studio](https://www.macrumors.com/2026/08/30/apple-unexpected-mac-mini-and-studio-demand/) — *Hacker News*
  - 📌 **内容**：苹果对 AI 工作负载带来的 Mac Mini/Mac Studio 需求增长估计不足。
  - 💡 **学习**：可关注 AI 本地推理对终端算力与内存的硬件需求。
  - 🧭 **拓展**：对比不同 Mac 配置跑大模型的性价比。
- [Agent memory as a file format](https://calpaterson.com/memoryfields.html) — *Hacker News*
  - 📌 **内容**：提出把 AI agent 的记忆以文件格式持久化的设计思路。
  - 💡 **学习**：可学习如何用结构化文件管理 agent 上下文与状态。
  - 🧭 **拓展**：尝试与向量数据库记忆方案做对比实验。
- [Smartphone LED detects hidden cameras with AI](https://www.chosun.com/english/industry-en/2026/08/30/SBFXUIJQYZEARKP5T4FBAY25HQ/) — *Hacker News*
  - 📌 **内容**：利用智能手机 LED 与 AI 识别隐藏摄像头。
  - 💡 **学习**：了解光电反射检测结合图像分类的安防思路。
  - 🧭 **拓展**：可复现一个用手机闪光灯加 CV 模型的检测 demo。
- [Show HN: We built the smallest dual-band aircraft tracker](https://pantsforbirds.com/the-worlds-smallest-dual-band-ads-b-receiver-module/) — *Hacker News*
  - 📌 **内容**：团队展示了体积最小的双频飞机追踪器。
  - 💡 **学习**：可学习射频接收、天线设计与嵌入式低功耗实现。
  - 🧭 **拓展**：结合 ADS-B 数据协议做飞行轨迹可视化。
- [Launch HN: Almanac (YC S26) – AI that knows your company](https://usealmanac.com/) — *Hacker News*
  - 📌 **内容**：YC 项目 Almanac 推出“了解你公司”的 AI 助手。
  - 💡 **学习**：可学习企业知识库检索、RAG 与权限集成的产品设计。
  - 🧭 **拓展**：体验后与 Notion AI 等企业知识产品对比。
- [How we configured OpenTelemetry logs in Rails](https://www.sixpatterns.com/blog/how-we-configured-opentelemetry-logs-in-rails) — *Hacker News*
  - 📌 **内容**：介绍了在 Rails 应用中配置 OpenTelemetry 日志的实践方法。
  - 💡 **学习**：可掌握 OTel 日志采集、管道配置与 Rails 集成。
  - 🧭 **拓展**：把日志与 trace/metric 关联做全链路可观测。
- [OpenShot 4.0 – Open-source video editor](https://www.openshot.org/blog/2026/08/30/openshot-40-record-edit-color-like-never-before/) — *Hacker News*
  - 📌 **内容**：开源视频编辑器 OpenShot 发布 4.0 版本。
  - 💡 **学习**：可了解开源视频编辑器的功能迭代与跨平台实现。
  - 🧭 **拓展**：试用新版并参与插件或特效开发。
- [Breaking Claude Code Opus 5 Auto Mode](https://embracethered.com/blog/posts/2026/breaking-claude-code-opus-5-and-automode/) — *Hacker News*
  - 📌 **内容**：有人演示了攻破 Claude Code Opus 5 的 Auto 模式。
  - 💡 **学习**：可学习 AI Agent 的工具调用边界与提示注入防护。
  - 🧭 **拓展**：用类似方法评估其他 Agent 框架的授权安全。
- [I turned my security cameras into an automatic bird identification system](https://jasontucker.blog/how-i-turned-my-security-cameras-into-an-automatic-bird-identification-system-with-birdnet-go/) — *Hacker News*
  - 📌 **内容**：作者把安防摄像头改造成自动鸟类识别系统。
  - 💡 **学习**：可学习视频流目标检测与模型部署的完整流程。
  - 🧭 **拓展**：将识别模型迁移到其他动物或物体检测场景。
- [Google Has Removed MV2 Extensions from the Chrome Web Store, Including UBO](https://webiterate.dev/google-removed-extensions-ublock-origin-108/) — *Hacker News*
  - 📌 **内容**：Google 已从 Chrome 应用商店移除 MV2 扩展，包含 uBlock Origin。
  - 💡 **学习**：需理解 Manifest V3 对扩展能力与隐私模型的限制。
  - 🧭 **拓展**：调研 MV3 下拦截类扩展的替代方案。

## 🚀 技能提升点（工作总结汇总）

### 1. LaTeX 特殊字符实体化
- **技能点**：掌握把 latex 文本嵌入 HTML 前后必须实体化的原则，并能用转义与引号感知解析双保险兜底。
- **坑点**：latex 里的裸 < > 会被 HTML5 解析器误判为标签开始，破坏 <question-latex> 闭合，后续内容全被吞进同一公式；data-formula 属性值里的原始 > 也会让按 > 定位标签结束的解析失败。
- **解决方案**：写入 innerHTML 前把公式体内的 < > 转成 &lt; &gt;；解析拍平时用引号感知扫描定位标签结束，latex 读取用 textContent 自动解码。
```text
html.replace(/(<question-latex\b[^>]*>)([\s\S]*?)(<\/question-latex>)/gi, (_, open, body, close) => open + body.replace(/</g, '&lt;').replace(/>/g, '&gt;') + close)
```
- **拓展**：可沉淀为统一的 latexToSafeHtml / htmlToLatex 工具，覆盖 innerHTML、data-formula 与拍平解析三处。
- *来源：admin-workspace / MEMORY*

### 2. Quill API 与 Clipboard matcher
- **技能点**：掌握 Quill 的 Delta/HTML 取值差异，以及自定义 clipboard 模块需显式补齐默认 matcher 的能力。
- **坑点**：watch Quill 内容时误用 getContents() 返回 Delta JSON 串，被当 HTML 写入 v-model 后替换/匹配全失败；自定义 clipboard 覆盖默认后 image/divider matcher 丢失，回填时图片与分割线被丢弃。
- **解决方案**：同步 HTML 改用 getHTML?.() ?? quill.root.innerHTML；在 registerClipboardMatchers 中显式 addMatcher 注册 image/divider，属性提取与 Blot.value 返回结构对齐。
```text
const html = quillRef.value?.getHTML?.() ?? "";
clipboard.addMatcher('img[data-type="ql-image"]', node => new Delta().insert({ image: { url: node.getAttribute('src') } }));
```
- **拓展**：可封装统一的 Quill 内容读写与粘贴 matcher 集合工具，避免每个编辑器重复踩坑。
- *来源：admin-workspace-new 2026-08-12 / MEMORY*

### 3. 文本匹配容错与归一化
- **技能点**：掌握先严格后容错的匹配策略，以及 typo 文本去壳归一的处理方式。
- **坑点**：AI 返回的 original 常带 <question-latex> 壳或 $ 定界符，且与正文 latex 空格不一致，严格 escapeRegExp 字面匹配会全部失配。
- **解决方案**：新增 normalizeTypoOriginal 去壳/去定界符；用 buildWhitespaceTolerantRegex 允许 \s* 容空白；严格匹配未命中再走容错匹配。
```text
const normalizeTypoOriginal = s => s.replace(/<\/?question-latex>/gi, '').replace(/^\s*\$([\s\S]*?)\$\s*$/, '$1');
const buildWhitespaceTolerantRegex = str => new RegExp(str.split(/\s+/).map(escapeRegExp).join('\\s*'));
```
- **拓展**：可推广到任何来源文本与目标文本格式不一致的匹配/高亮/替换场景。
- *来源：admin-workspace 2026-08-11 / questionCheck.ts*

### 4. Vue watch 双向同步循环防护
- **技能点**：掌握 Vue watch 双向同步（状态/数组互相写回）中防止无限递归的守卫写法。
- **坑点**：watch A 写数组新引用，watch 数组 deep 又写回 A，即使最终值未变仍互相唤醒，直至 Maximum recursive updates exceeded。
- **解决方案**：每次写回前先比较新旧值，无变化直接 return；推导数组时逐项相等就不再赋新引用。
```text
const next = [leftWidth, "auto", rightWidth];
if (next.every((v, i) => v === localPanelSizes.value[i])) return;
localPanelSizes.value = next;
```
- **拓展**：可沉淀为 Vue3 状态同步通用模式：所有 watch 回调先守卫再赋值。
- *来源：admin-workspace 2026-08-11 batchInput.vue*

### 5. 按需注册组件须显式引 CSS
- **技能点**：明确自动按需注册组件不等于自动加载 CSS，能迅速定位组件库新组件样式缺失导致的布局问题。
- **坑点**：unplugin-vue-components + ElementPlusResolver 未补齐 el-splitter/split-bar 等较新组件 CSS，组件退化为普通 block，三栏竖向堆叠。
- **解决方案**：在组件文件显式 import el-splitter.css 与 el-splitter-panel.css，逐个组件按需引 CSS 才稳妥。
```text
import "element-plus/theme-chalk/el-splitter.css";
import "element-plus/theme-chalk/el-splitter-panel.css";
```
- **拓展**：可将需显式引 CSS 的组件清单写入团队文档或集中到公共入口统一引入。
- *来源：admin-workspace MEMORY / ResizablePanels*

### 6. 受控 value 的 falsy 陷阱
- **技能点**：掌握受控组件值判断不应依赖 || 等 falsy 写法，避免丢失合法值 0。
- **坑点**：Select 用 value={config.value || undefined} 判断受控值，数字 0 被当 falsy 丢掉，含 0 的选项永远无法回显。
- **解决方案**：选项 value 使用字符串（如 '0'），onChange 时再 Number() 转回数字；判断有无值用 === undefined/null 显式比较。
```text
const val = config.value === undefined || config.value === null ? undefined : String(config.value);
// onChange: onChange(Number(value))
```
- **拓展**：适用于所有受控表单/筛选器，可封装 hasValue 工具统一处理。
- *来源：admin-workspace-hr MEMORY 2026-08-31*

