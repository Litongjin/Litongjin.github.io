---
title: "工作台日报 · 2026-08-19"
date: 2026-08-19 17:42:12
categories: [日报]
tags: [日报]
author: Litongjin
---

# 工作台日报 · 2026-08-19

> 自动生成于 2026-08-19 17:42 · 个人工作台 Agent

## 🔥 行业热点

- [AI usage patterns in software teams](https://linear.app/data) — *Hacker News*
  - 📌 **内容**：Explores how software teams are adopting AI across workflows.
  - 💡 **学习**：Identify effective patterns for AI-assisted development.
  - 🧭 **拓展**：Survey your team's current AI usage.
- [Memory prices climb 500% in 12 months](https://www.tomshardware.com/pc-components/ram/memory-prices-climb-500-percent-in-12-months-up-to-10x-the-lowest-ever-tracked-prices-128gb-of-ddr5-now-usd3-399) — *Hacker News*
  - 📌 **内容**：DRAM memory prices have surged 500% over the past year.
  - 💡 **学习**：Understand supply-demand impact on hardware costs.
  - 🧭 **拓展**：Monitor memory market for infrastructure planning.
- [Cursor launches Origin, GitHub alternative](https://cursor.com/changelog/origin-code-hosting) — *Hacker News*
  - 📌 **内容**：Cursor introduces Origin, a new GitHub alternative.
  - 💡 **学习**：Evaluate Origin's git hosting and collaboration features.
  - 🧭 **拓展**：Try migrating a test repo.
- [Claude Code May–August 2026 weekly limits promotion](https://support.claude.com/en/articles/15910845-claude-code-may-august-2026-weekly-limits-promotion) — *Hacker News*
  - 📌 **内容**：Anthropic announces weekly usage limits promotion for Claude Code from May to August 2026.
  - 💡 **学习**：Understand new limits to plan AI coding usage.
  - 🧭 **拓展**：Check Anthropic's official announcement.
- [A 3D fruit fly on macOS desktop powered by the real FlyWire connectome](https://github.com/DenisSergeevitch/desktop-fly) — *Hacker News*
  - 📌 **内容**：A macOS desktop app visualizes a 3D fruit fly brain using real FlyWire connectome data.
  - 💡 **学习**：Explore rendering large biological datasets in 3D.
  - 🧭 **拓展**：Inspect the FlyWire dataset or source code.
- [Cerebras CS-4](https://www.cerebras.ai/cs4) — *Hacker News*
  - 📌 **内容**：Cerebras unveils the CS-4, a new AI accelerator for large-scale training.
  - 💡 **学习**：Study wafer-scale engine architecture for AI.
  - 🧭 **拓展**：Compare CS-4 specs with GPU clusters.
- [Turbovec – Google's TurboQuant for vector search in Rust](https://github.com/RyanCodrai/turbovec) — *Hacker News*
  - 📌 **内容**：Google releases Turbovec, a Rust vector search library implementing TurboQuant.
  - 💡 **学习**：Learn quantization techniques for efficient vector search.
  - 🧭 **拓展**：Benchmark it against other vector indexes.
- [Teaching my kid to code with a modern MUD](https://tau.dev/2026/08/07/canon) — *Hacker News*
  - 📌 **内容**：A parent teaches coding using a modern MUD game environment.
  - 💡 **学习**：Use game-based scenarios to teach programming concepts.
  - 🧭 **拓展**：Try building a simple MUD with Python.
- [Claude writing a macOS driver for my obscure HP printer built only for Windows](https://twitter.com/kuberwastaken/status/2089377982536388964) — *Hacker News*
  - 📌 **内容**：Claude AI wrote a macOS driver for a Windows-only HP printer.
  - 💡 **学习**：Leverage AI for reverse engineering and systems programming.
  - 🧭 **拓展**：Attempt similar driver or firmware projects.
- [Apple announces changes for apps in the European Union](https://www.apple.com/newsroom/2026/08/apple-announces-changes-for-apps-in-the-european-union/) — *Hacker News*
  - 📌 **内容**：Apple updates app rules for EU developers.
  - 💡 **学习**：Understand new compliance requirements for EU distribution.
  - 🧭 **拓展**：Review the updated App Store guidelines.

## 🚀 技能提升点（工作总结汇总）

### 1. Quill Delta 与 HTML 同步
- **技能点**：掌握 vue-quill 内容读取 API 的语义差异（getContents 返回 Delta，getHTML 返回 HTML），能正确选择同步方法。
- **坑点**：误用 getContents() 得到 Delta 对象，把 Delta JSON 字符串当 HTML 传给 DOMParser，导致 v-model 被污染为 Delta JSON 文本（< 被转义为 &lt;）。
- **解决方案**：改调 quillRef.value?.getHTML?.() ?? ''，取真实 innerHTML；watch Quill 内容同步 v-model 一律用 getHTML / quill.root.innerHTML。
```text
// 错误
const delta = quillRef.value?.getContents().trim()
form.questionStem = processHtmlContent(delta)
// 正确
const html = quillRef.value?.getHTML?.() ?? ''
form.questionStem = html
```
- **拓展**：与 Delta 交互可显式 import Delta，但同步给 HTML 上下文必须走 getHTML。
- *来源：admin-workspace 2026-08-12*

### 2. Vue watch 双向同步防循环
- **技能点**：掌握 Vue watch 多状态互相写回时的循环触发机理，能用值比较守卫避免 Maximum recursive updates。
- **坑点**：状态A变化→watch写新数组引用→deep watch写回状态B→再触发，即使值未变也无限递归。
- **解决方案**：每一步写回前比较目标值，无变化则直接 return，不产生新引用、不触发下游 watch。
```text
watch([flagA, flagB], () => {
  const next = computeSizes()
  if (next.every((v, i) => v === localPanelSizes.value[i])) return
  localPanelSizes.value = next
})
watch(localPanelSizes, (sizes) => {
  const nextA = deriveA(sizes)
  if (flagA.value !== nextA) flagA.value = nextA
}, { deep: true })
```
- **拓展**：也可用 watch 的 flush 选项或单向数据流重构，但值守卫是最直接的兜底。
- *来源：admin-workspace 2026-08-11*

### 3. Quill 工具栏数组展平
- **技能点**：理解 Quill toolbar 对 control 的解析规则，能定位与修复数组未展平导致的空白按钮（ql-0）。
- **坑点**：getDefaultButtonConfig 的 list/indent 返回数组，order.map 混入数组项，Quill 把数组当对象、以索引 '0' 作 format 名生成空按钮。
- **解决方案**：将 order.map 改为 order.flatMap，每个子项独立成 control，Quill 正常识别。
```text
// 错误
const controls = order.map(name => getDefaultButtonConfig(name))
// 正确
const controls = order.flatMap(name => getDefaultButtonConfig(name))
```
- **拓展**：凡'一个配置项对应多个按钮'的场景，下游消费必须 flatMap，不能 map。
- *来源：admin-workspace-new 2026-08-12*

### 4. 富文本拍平与实体/标签解析
- **技能点**：掌握 HTML 属性值内含未转义 < > 时的标签解析边界处理，以及多字符 HTML 实体的整体解码与偏移映射。
- **坑点**：用 html.indexOf('>') 定位标签结束会被 data-formula 内的裸 > 截断；逐字符解码 &lt; 会把实体拆碎。
- **解决方案**：标签结束改为引号感知扫描（引号内跳过），文本节点累积后整体 decodeHtmlEntities，实体区间映射到起始偏移。
```text
function findTagEnd(html, i) {
  let inQuote = false
  for (let j = i + 1; j < html.length; j++) {
    const ch = html[j]
    if (ch === '"' || ch === "'") inQuote = !inQuote
    else if (ch === '>' && !inQuote) return j
  }
  return -1
}
```
- **拓展**：若上游生成 HTML 能将 latex 的 < > 实体化存储则更根本，解析侧兜底只是防御。
- *来源：admin-workspace 2026-08-12*

### 5. innerHTML 插入前转义裸符号
- **技能点**：认识到浏览器 HTML5 解析器会把裸 < 当作标签开始，能在写入含 latex 的文本前做针对性转义。
- **坑点**：v-katex render 直接 el.innerHTML = html，latex 中的裸 < 被误判为标签，破坏闭合，整段内容被吞并成一个公式节点。
- **解决方案**：用正则把 <question-latex> 内容中的 < > 替换为 &lt;/&gt;，浏览器解析安全，textContent 读取时自动解码回原 latex。
```text
html = html.replace(/(<question-latex\b[^>]*>)([\s\S]*?)(<\/question-latex>)/gi,
  (_, open, latex, close) => open + latex.replace(/</g, '&lt;').replace(/>/g, '&gt;') + close)
el.innerHTML = html
```
- **拓展**：任何把含数学/模板文本塞入 innerHTML 的场景都应先做 HTML 转义或使用结构化解析。
- *来源：admin-workspace MEMORY.md*

### 6. 按需注册组件的 CSS 显式导入
- **技能点**：掌握 unplugin-vue-components 按需注册组件但不会自动补齐全部组件 CSS 的坑，能显式 import 缺失样式解决布局失效。
- **坑点**：element-plus 的 el-splitter 等较新组件未全量引入 CSS，退化为普通 block 容器，三栏方向失效（上下堆叠）。
- **解决方案**：在组件文件显式 import 'element-plus/theme-chalk/el-splitter.css' 与 el-splitter-panel.css；不要依赖全局 index.css。
```text
import "element-plus/theme-chalk/el-splitter.css"
import "element-plus/theme-chalk/el-splitter-panel.css"
```
- **拓展**：排查样式失效时先确认对应组件 CSS 是否真的被加载，再谈布局属性。
- *来源：admin-workspace MEMORY.md*


## 📚 每日基础技术总结

> 今日主题：**Linux 常用命令与权限管理**（2. 后端基础（Node.js / Java / Python））

### 1. 核心概念速览
Linux 命令和权限管理，就像你拿到一台全新的电脑（服务器），你需要学会用键盘指挥它。权限管理则是这台电脑的门禁系统——决定谁能进哪个房间、能碰哪些东西。在计算机体系里，Linux 是所有后端服务（Node.js、Java、Python）的‘地基’，你写的代码最终都要跑在 Linux 上。前端工程师平时在浏览器里调试，但一旦涉及部署、容器、云服务器，就必须学会用命令‘开锁进门’和‘设置门禁’。不掌握它，你写的代码就永远只能在本地自嗨，无法真正上线。

### 2. 底层原理剖析
Linux 的底层机制其实很简单：一切皆文件，命令就是调用的程序，权限就是文件的属性。权限管理核心是‘用户’和‘文件权限位’。每个文件有三组权限：所有者（owner）、所属组（group）、其他人（others），每组有读（r=4）、写（w=2）、执行（x=1）三个位，用数字表示就是总和，比如 7=rwx。当你执行 `ls -l`，看到 `-rw-r--r--`，第一个字符是文件类型（- 普通文件，d 目录），后面 9 个字符就是三组权限。目录的权限和文件不同：读权限能列出文件名，写权限能创建/删除文件，执行权限能进入目录。这就像你家的门：你有钥匙（执行）才能进去，有清单（读）才能知道有什么，有笔（写）才能改动。对比前端：JS 的 `let` 和 `const` 是变量作用域控制，而 Linux 权限是系统级访问控制；JS 的 `Object.freeze` 只是表面不可变，Linux 权限是内核强制执行的，更底层。伪代码：if (用户身份 == 文件所有者) { 检查 owner 位 } else if (用户属于文件组) { 检查 group 位 } else { 检查 other 位 }。注意顺序，匹配到第一个就停止，不再往后检查。

### 3. 基础代码与实战验证
这里用一个纯命令序列来验证权限原理，不依赖任何框架。

```bash
# 1. 创建一个测试文件
$ touch secret.txt

# 2. 查看默认权限
$ ls -l secret.txt
# 输出：-rw-r--r-- 1 user group 0 date secret.txt
# 解析：所有者可读写，组和其他人只读

# 3. 用数字修改权限为 700（仅所有者可读写执行）
$ chmod 700 secret.txt
$ ls -l secret.txt
# 输出：-rwx------ 1 user group ... secret.txt

# 4. 切换到普通用户（假设存在）尝试读文件
$ su - otheruser
$ cat /home/user/secret.txt
# 报错：Permission denied，因为 other 位为 0，没有读权限

# 5. 切回原用户，给 other 加读权限
$ exit
$ chmod o+r secret.txt
$ su - otheruser
$ cat /home/user/secret.txt
# 成功显示内容，说明权限位生效
```

每一行注释都解释了底层如何运作：`chmod 700` 把权限位设为二进制 111 000 000，内核在每次打开文件时检查调用者身份和文件属性，不符合就返回 `EACCES` 错误，`cat` 程序收到后打印 Permission denied。这就是权限管理最直观的验证。

### 4. 常见误区与进阶思考
误区一：以为 `chmod 777` 是万能解决方案。很多新手为了省事直接放开所有权限，导致安全隐患。权限管理是安全底线，就像把家门钥匙复制给所有人，数据泄露风险极高。正确做法是最小权限原则，只给需要的用户和组分配必要的权限。

误区二：混淆目录的写权限和文件的写权限。对目录有写权限可以在目录里创建/删除文件，即使文件本身只读。比如你有个目录权限是 `drwxrwxrwx`，里面有个文件权限是 `-r--r--r--`，你依然能删除这个文件，因为删除操作依赖目录的写权限，而不是文件的写权限。这就像你有房间的钥匙（目录写），就能把桌上的书（文件）拿走，书的封面写不写'不可动'没用。

思考题：如果文件所有者是 root，普通用户对文件有 `r` 权限，但文件所在目录对普通用户没有任何权限，普通用户能直接通过绝对路径读取这个文件吗？答案是不能，因为内核解析路径时需要逐级获取目录的执行权限，没有目录执行权限就无法访问到文件。你能说清楚这个底层逻辑吗？
