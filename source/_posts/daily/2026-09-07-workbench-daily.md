---
title: "工作台日报 · 2026-09-07"
date: 2026-09-07 07:01:27
categories: [工作日记]
tags: ["日报", "大模型", "硬件", "隐私", "AI研究"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-07

## 🔥 行业热点

- [Chrome again exempts Google from user site data settings](https://lapcatsoftware.com/articles/2026/9/1.html) — *Hacker News*
  - 📌 **内容**：报道 Chrome 在用户网站数据设置上再次将 Google 自身站点排除在外，引发对浏览器默认信任边界的讨论。
  - 💡 **学习**：可关注浏览器存储分区、第三方 Cookie 与站点权限的默认策略差异，理解平台方特殊豁免对 Web 生态的影响。
  - 🧭 **拓展**：可通过 chrome://settings/content 或 DevTools Application 面板检查不同站点的数据隔离情况。
- [Research acceleration: The view inside OpenAI](https://openai.com/index/research-acceleration-view-inside-openai) — *Hacker News*
  - 📌 **内容**：从 OpenAI 内部视角探讨如何加速 AI 研究进程，涉及实验流程与团队协作的工程化改造。
  - 💡 **学习**：可借鉴其快速实验和自动化评估的流水线思路，优化自己的机器学习工作流。
  - 🧭 **拓展**：可阅读 OpenAI 公开论文与代码库，尝试复现其中的评估框架。
- [Cloud in a Bottle: making self-hosting accessible to everyone](https://cloudinabottle.org/blog/launch-post) — *Hacker News*
  - 📌 **内容**：以“瓶子里的云”为概念降低自托管门槛，让个人也能快速运行云服务。
  - 💡 **学习**：学习如何用容器、编排和配置模板封装复杂服务，提供一键部署体验。
  - 🧭 **拓展**：可实测其部署流程，并与 Docker Compose 方案对比运维成本。
- [LLMs as a Cognitive Virus](https://arxiv.org/abs/2609.03344) — *Hacker News*
  - 📌 **内容**：把大语言模型比作“认知病毒”，讨论 LLM 对思维方式和信息环境的潜在影响。
  - 💡 **学习**：使用大模型时需注意输出偏差与过度依赖，建立批判性验证和事实核查习惯。
  - 🧭 **拓展**：可在实际 prompt 场景中设计对抗性测试，观察模型对错误信息的坚持程度。
- [The "$60 Gaming PC" – AMD BC-250 (2025)](https://devquasar.com/hardware/the-60-gaming-pc-amd-bc-250/) — *Hacker News*
  - 📌 **内容**：围绕 AMD BC-250 构建 60 美元游戏 PC 的硬件玩法，展示低成本 DIY 可能性。
  - 💡 **学习**：了解矿卡或专用计算硬件的供电、散热与驱动改造，拓展硬件选型思路。
  - 🧭 **拓展**：可参考其功耗与性能数据，评估是否适合作为家庭实验室节点。
- [Nitter and XCancel resume service after legal advice](https://github.com/zedeus/nitter/commit/1428b4c2b4246f92a7e5b2673438e5fb39fcc4a3) — *Hacker News*
  - 📌 **内容**：Nitter 与 XCancel 在获得法律建议后恢复服务，开源替代前端继续为 Twitter 用户提供隐私访问。
  - 💡 **学习**：开源项目需要关注 API 条款与当地法律，提前设计合规边界。
  - 🧭 **拓展**：可查看项目仓库的法律声明与更新日志，了解具体合规调整。
- [Learn Programming with OCaml](https://usr.lmf.cnrs.fr/lpo/) — *Hacker News*
  - 📌 **内容**：面向入门者用 OCaml 讲解编程概念，强调函数式编程与类型系统。
  - 💡 **学习**：通过 OCaml 掌握模式匹配、不可变数据和代数类型，可迁移到 Rust、Scala 等语言。
  - 🧭 **拓展**：尝试用 OCaml 实现一个小型解释器或 JSON 解析器。
- [Asahi Linux on M3](https://asahilinux.org/2026/09/m2-episode-1/) — *Hacker News*
  - 📌 **内容**：Asahi Linux 适配 Apple M3 芯片，持续完善 Apple Silicon 上的 Linux 体验。
  - 💡 **学习**：可从中学到驱动移植、设备树和 ARM64 启动流程，理解硬件底层协作。
  - 🧭 **拓展**：如有 M3 设备可安装体验，并参与驱动排错社区。
- [GrapheneOS Overhauled Default Apps and Secure Clipboard](https://grapheneos.social/@GrapheneOS/117225539756835649) — *Hacker News*
  - 📌 **内容**：GrapheneOS 对默认应用和安全剪贴板进行重构，强化 Android 隐私保护。
  - 💡 **学习**：了解剪贴板隔离、权限最小化与应用沙箱在移动安全中的设计取舍。
  - 🧭 **拓展**：可对比 AOSP 与 GrapheneOS 的剪贴板实现，梳理攻击面。
- [AMD Based FreeBSD Desktop Reloaded](https://vermaden.wordpress.com/2026/09/06/amd-based-freebsd-desktop-reloaded/) — *Hacker News*
  - 📌 **内容**：介绍在 AMD 平台上重新构建 FreeBSD 桌面环境的过程与现状。
  - 💡 **学习**：可学习 FreeBSD 的图形栈、驱动兼容性和桌面包管理实践。
  - 🧭 **拓展**：在虚拟机或旧 AMD 设备上安装 FreeBSD，尝试复现桌面流程。

## 🚀 技能提升点（工作总结汇总）

### 1. HTML 实体解码拍平与区间映射
- **技能点**：掌握将 HTML 内含实体的文本节点整体解码并维护实体起始偏移映射的能力，适配高亮/替换的区间回算。
- **坑点**：逐字符解码实体必然失败（实体跨多字符），且按 mapIndex 映射可能把实体拦腰截断。
- **解决方案**：文本节点先累积再整体 decodeHtmlEntities，返回 entityRanges；匹配命中实体时片段边界延伸到实体末尾，避免截断。
- **拓展**：可沉淀为通用「带偏移映射的 HTML 拍平」工具，供搜索/标注/选区等场景复用。

### 2. 公式 latex 内裸 < > 破坏 HTML 解析
- **技能点**：识别 latex 特殊字符与 HTML 解析冲突，设计引号感知的标签扫描或预先实体化。
- **坑点**：data-formula 属性内若含原始 < 或 >，用 indexOf('>') 定位标签结束会把标签拦腰截断，公式内容丢失。
- **解决方案**：标签结束定位改为引号感知扫描：引号内的 > 不视为标签结束；或写入属性前统一转义 < > 为实体。
- **拓展**：可推广到所有「HTML 属性内嵌代码/数学公式/模板串」场景的序列化规范。

### 3. Quill getContents 返回 Delta 误当 HTML
- **技能点**：掌握 vue-quill 编辑器内容读取 API 语义，避免 v-model 被非 HTML 数据结构污染。
- **坑点**：watch Quill 内容同步 v-model 时误用 getContents()（返回 Delta），后续 DOMParser 解析 Delta JSON 串导致高亮/替换全部失配。
- **解决方案**：使用 quillRef.value?.getHTML?.() ?? '' 或 quill.root.innerHTML 取真实 HTML 字符串。
- **拓展**：封装 Quill 双向绑定时应统一提供 getHTML 路径，并在类型层禁止 getContents 直接写入 HTML 字段。

### 4. Vue watch 双向同步递归更新
- **技能点**：理解 Vue watch 互相唤醒导致 Maximum recursive updates 的机制，并能在同步链路中加值比较守卫。
- **坑点**：状态 A 写数组、数组 deep watch 写回状态 B、B 又触发 A 的 watch，即使值未变也因新引用反复触发。
- **解决方案**：每次回写前比较当前值与推导值是否相等，相等则直接 return；避免无意义的新引用赋值。
- **拓展**：可沉淀为 Vue 状态同步通用守则：任何 watch→写→另一个 watch 的闭环必须有终止条件。

### 5. toolbar-order 映射数组未展平
- **技能点**：识别 Quill toolbar 配置中「单按钮映射为数组」导致 addControls 把数组对象误当控件的根因。
- **坑点**：getDefaultButtonConfig 返回 [{list:'ordered'},{list:'bullet'}] 数组，buildToolbarContainer 用 map 未展平，Quill 把数组当对象生成了 ql-0 空按钮。
- **解决方案**：调用处改用 flatMap 将多按钮数组展平为一维 controls，每个按钮独立注册。
- **拓展**：所有「配置项展开为控件列表」的框架层都应约定 flatMap，并在单元测试中验证按钮数量。

### 6. antd 组件按需注册须显式引 CSS
- **技能点**：掌握按需组件注册与样式加载的关系，能在布局异常时优先怀疑样式未载入而非改业务布局。
- **坑点**：el-splitter 等较新组件通过 resolver 自动注册但 CSS 未全量引入，组件退化为普通 block 导致三栏竖向堆叠。
- **解决方案**：在组件文件中显式 import 'element-plus/theme-chalk/el-splitter.css' 和 el-splitter-panel.css，不依赖全量 index.css。
- **拓展**：可建立「按需组件必需显式引样式」清单，避免同类组件再次踩坑。

### 7. latex 空白差异导致严格匹配失配
- **技能点**：设计容空白正则匹配 latex 文本，同时保留严格匹配优先策略，保证公式与原文空白不一致时仍能命中。
- **坑点**：AI 生成的 typo original 无空格，但正文 latex 带空格，escapeRegExp(original) 严格字面匹配永远失败。
- **解决方案**：新增 buildWhitespaceTolerantRegex：字符间允许任意空白（\s*），纯空白字符串返回 null 防死循环；先严格后容错重试。
- **拓展**：可推广到其他「用户输入与正文存在格式化差异」的模糊匹配场景，如去标签、去定界符归一。

