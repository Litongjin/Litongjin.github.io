---
title: "工作台日报 · 2026-09-10"
date: 2026-09-10 07:01:19
categories: [工作日记]
tags: ["日报", "AI Agent", "大模型", "网络安全", "AI安全"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-10

## 🔥 行业热点

- [Muse – Meta’s personal AI agent](https://ai.meta.com/muse/) — *Hacker News*
  - 📌 **内容**：Meta推出个人AI代理Muse，主打利用对话与记忆完成个性化日常任务。
  - 💡 **学习**：可以研究其记忆管理、工具调用和任务规划机制，借鉴到Agent类应用。
  - 🧭 **拓展**：尝试用开源框架复现一个轻量的个人助理流程。
- [Show HN: Geiger – See every AI agent on your machine and what it can touch](https://github.com/Atomburstofficial/geiger) — *Hacker News*
  - 📌 **内容**：一个可视化工具，让用户看到本机每个AI代理可访问的资源和操作权限。
  - 💡 **学习**：学习如何对本地Agent做系统调用追踪、配置审计与最小权限评估。
  - 🧭 **拓展**：可部署到自己的Agent开发环境中验证沙箱是否合理。
- [Understanding the recent DDoS attack against Read the Docs](https://about.readthedocs.com/blog/2026/09/2026-ddos-attack/) — *Hacker News*
  - 📌 **内容**：分析Read the Docs近期遭受的DDoS攻击，复盘攻击特征与防护过程。
  - 💡 **学习**：理解DDoS攻击常见放大手段及CDN、速率限制等缓解策略。
  - 🧭 **拓展**：可结合自身服务日志做容量与限流压测。
- [Procedural Graphs: Self-Evolving Execution Structures for LLM Agents](https://arxiv.org/abs/2609.09153) — *Hacker News*
  - 📌 **内容**：一种让LLM Agent执行结构随任务自演化的程序化图方法，替代固定流程。
  - 💡 **学习**：学习图式任务拆解与动态调整执行顺序的Agent框架设计。
  - 🧭 **拓展**：可在复杂工具调用场景中对比链式与图式Agent效果。
- [Better AI code comment detector](https://entropicthoughts.com/better-ai-comment-classifier) — *Hacker News*
  - 📌 **内容**：一个改进的AI代码注释检测器，可识别注释是否由AI生成或质量不高。
  - 💡 **学习**：了解如何用文本分类/启发式规则检测AI注释并集成到代码评审。
  - 🧭 **拓展**：可将其作为CI检查，自动过滤无信息量注释。
- [Show HN: Self-hosted company OS, Claude Code and Codex agents in departments](https://github.com/OtoDock/oto-dock) — *Hacker News*
  - 📌 **内容**：一个自托管的公司操作系统，按部门编排Claude Code和Codex等AI代理。
  - 💡 **学习**：学习多Agent协同、权限隔离与部门级任务路由的实现思路。
  - 🧭 **拓展**：可参考其架构搭建内部Agent工作台。
- [Microsoft says email spammers are adopting ASCII smuggling](https://arstechnica.com/security/2026/09/once-popular-for-attacking-ai-ascii-smuggling-is-embraced-by-spammers/) — *Hacker News*
  - 📌 **内容**：微软指出垃圾邮件正利用ASCII走私（Ascii Smuggling）绕过文本安全检测。
  - 💡 **学习**：了解Unicode/同形字符混淆的原理，以及邮件解析时的规范化处理。
  - 🧭 **拓展**：可在邮件网关中加入字符标准化与隐藏文本扫描。
- [Claude, change the “Add to Cart” button to blue](https://opusfived.dev/) — *Hacker News*
  - 📌 **内容**：一个用自然语言让Claude直接修改前端按钮颜色的AI编程实操案例。
  - 💡 **学习**：掌握向AI编程助手描述UI改动、约束与验证结果的方式。
  - 🧭 **拓展**：可在实际前端项目中尝试“对话驱动”的小改动流程。
- [Desert Ant Labs: local, fast models that run on device](https://desertant.com/blog/introducing-desert-ant-labs/) — *Hacker News*
  - 📌 **内容**：Desert Ant Labs推出可在设备端运行的快速本地模型，强调低延迟和隐私。
  - 💡 **学习**：关注端侧小模型的量化和推理优化，以及离线部署经验。
  - 🧭 **拓展**：可比较其与云端大模型在简单Agent任务上的表现。
- [How I advertise malicious software on Google Ads](https://xlii.space/eng/malicious-software-on-google-ads/) — *Hacker News*
  - 📌 **内容**：作者披露如何通过在Google Ads投放广告来传播恶意软件并绕过审核。
  - 💡 **学习**：认识恶意广告常用的落地页混淆、域名轮换与企业资质滥用手法。
  - 🧭 **拓展**：可据此优化企业广告品牌保护与落地页检测规则。

## 🚀 技能提升点（工作总结汇总）

### 1. innerHTML 裸 < 转义
- **技能点**：掌握将含特殊字符的文本安全插入 HTML 的必要性，理解浏览器 HTML5 解析器对裸 < 的误判。
- **坑点**：LaTeX 内裸 < 被当作标签开始，破坏闭合结构，导致后续内容被吞并进同一公式节点。
- **解决方案**：innerHTML 赋值前，用正则将 <question-latex> 内容中的 < > 转义为 &lt; &gt;，解析时统一转义。
```text
const safe = raw.replace(/(<question-latex\b[^>]*>)([\s\S]*?)(<\/question-latex>)/gi, (_, a, body, b) => a + body.replace(/</g, '&lt;').replace(/>/g, '&gt;') + b);
```
- **拓展**：适用于所有富文本/公式/代码片段注入场景，可沉淀为通用 escapeHtml 工具。
- *来源：admin-workspace 2026-08-10*

### 2. 按需组件库 CSS 缺失
- **技能点**：排查组件样式失效时，先确认样式文件是否真的被加载，而非盲目调整布局属性。
- **坑点**：unplugin-vue-components 注册了组件但未自动引入对应 CSS，el-splitter 退化为普通 block，三栏堆叠。
- **解决方案**：显式 import 'element-plus/theme-chalk/el-splitter.css' 和 'el-splitter-panel.css'，不依赖全量样式。
```text
import "element-plus/theme-chalk/el-splitter.css";
import "element-plus/theme-chalk/el-splitter-panel.css";
```
- **拓展**：其他按需引入的组件库同理，遇到样式异常先查 CSS 是否缺失。
- *来源：admin-workspace MEMORY*

### 3. antd v6 API 迁移
- **技能点**：掌握 antd 大版本升级的废弃属性映射，写出符合新版本的代码。
- **坑点**：bodyStyle、destroyOnClose、Drawer width、Space direction 等旧 API 在 v6 已废弃，直接使用会失效或警告。
- **解决方案**：按新 API 替换：bodyStyle→styles={{body}}，destroyOnClose→destroyOnHidden，Drawer width→size，Space direction→orientation。
```text
<Modal styles={{ body: { maxHeight: 'calc(100vh - 160px)', overflowY: 'auto' } }} destroyOnHidden />
```
- **拓展**：升级任何 UI 库时先查 migrations，避免历史遗留 API 继续扩散。
- *来源：admin-workspace-hr MEMORY*

### 4. Table sticky 与 scroll.x
- **技能点**：安全使用 antd Table 粘性表头，理解 sticky 与滚动容器的交互。
- **坑点**：sticky 不配 scroll.x 会导致表头与 body 列错位；sticky 还会启用自带横向滚动条，产生无用横滚。
- **解决方案**：需要 sticky 时配合 scroll={{ x: 'max-content' }}；仅表头吸顶时改用 CSS th { position: sticky; top: 0 }。
```text
<Table sticky scroll={{ x: 'max-content' }} />
// 或
<Table className="xxx-table" scroll={{ y: 400 }} />
/* .xxx-table .ant-table-thead > tr > th { position: sticky; top: 0; z-index: 2 } */
```
- **拓展**：其他组件库同理，先弄清 sticky 的滚动上下文副作用。
- *来源：admin-workspace-hr MEMORY*

### 5. 跨字段校验 dependencies
- **技能点**：理解表单校验的触发机制，避免跨字段校验的错误提示残留。
- **坑点**：antd validator 只在自身 validateTrigger 触发，另一字段变化不会自动重跑，导致错误提示残留。
- **解决方案**：跨字段校验使用 dependencies 声明关联字段，或确认后端是否真的要求配对，避免前端过度约束。
```text
<Form.Item dependencies={['endDate']} rules={[{ validator: (_, v) => v && startDate > v ? Promise.reject() : Promise.resolve() }]}>
  <DatePicker />
</Form.Item>
```
- **拓展**：react-hook-form 等表单库也有类似 watch + trigger 机制，可迁移。
- *来源：admin-workspace-hr MEMORY*

### 6. 数字 0 的 falsy 陷阱
- **技能点**：处理受控组件取值时，避免将合法值 0 当作空值丢失。
- **坑点**：value={config.value || undefined} 会把数字 0 丢掉，导致筛选选项无法回显。
- **解决方案**：改用 ?? 判断 null/undefined，或使用字符串 '0'/'1' 作为 option 值，onChange 再 Number() 转换。
```text
value={config.value ?? undefined}
// 或 options value 用 '0'/'1'，onChange 时 Number(value)
```
- **拓展**：所有 falsy 值（0、''、false）场景都适用，是 JS 判断的通用注意点。
- *来源：admin-workspace-hr MEMORY*

### 7. Quill clipboard matcher 覆盖
- **技能点**：定制 Quill 编辑器时，理解自定义 Clipboard 模块不会继承默认 matcher，需显式补齐。
- **坑点**：注册 TableClipboard 覆盖默认 clipboard 后，img/divider 等默认 matcher 失效，粘贴/回显时丢内容。
- **解决方案**：在 registerClipboardMatchers 里为 img[data-type=ql-image] 和 divider 显式添加 matcher，与对应 blot 结构对齐。
```text
clipboard.addMatcher('img[data-type="ql-image"]', node => new Delta().insert({ image: extractImage(node) }));
clipboard.addMatcher('divider.ql-divider', node => new Delta().insert({ divider: extractDivider(node) }));
```
- **拓展**：新增任何自定义 embed blot 接 clipboard 输入时，都要显式注册 matcher，不能依赖默认兜底。
- *来源：admin-workspace-new MEMORY*

