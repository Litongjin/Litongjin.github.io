---
title: "每日基础技术总结 · 2026-08-26 · JSON 结构化输出"
date: 2026-08-26 06:55:38
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-26 · JSON 结构化输出

## 📚 今日主题

> **JSON 结构化输出**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
JSON 结构化输出是指：在以大语言模型（LLM）为推理核心的系统中，通过约束解码过程或对生成文本进行后处理，使模型输出严格符合 JSON 语法与既定 Schema 的文本。其本质是解决『自由文本生成』与『机器可解析数据』之间的鸿沟：模型原生输出是概率化的 token 序列，而业务系统要求确定性的、类型明确的、可程序化消费的数据结构。机制分为两层：一是解码期约束（structured decoding / grammar-constrained generation），在每次采样时屏蔽不符合 JSON 语法或 Schema 的 token；二是生成后期校验与修复（JSON parsing + retry/repair）。在整个计算机/AI 体系中，它位于 LLM 应用层与数据交换层的交界处，是 Agent 工具调用、RAG 结果回传、多步推理状态机等场景的契约基础。专业工程师必须掌握，因为这是将概率模型嵌入确定性工程系统的核心适配层，直接决定系统鲁棒性、可观测性与上下游集成的稳定性，也涉及安全边界（防止 prompt injection 伪造结构化指令）。

### 2. 底层原理剖析
底层运行机制可拆解为三条路径。路径一：Prompt 约束 + 后处理。模型按指令生成文本，系统用 JSON.parse 解析；若失败则重试或调用修复模型。这是最脆弱但兼容性最好的方案，本质是把语法责任外包给模型，工程上仅做容错。路径二：约束解码（Structured Decoding）。在自回归生成的每一步，维护一个有限状态机（FSM），该 FSM 由 JSON 语法（如 RFC 8259）和用户定义的 JSON Schema 联合编译而来。当前已生成的 token 序列决定 FSM 当前状态，下一合法 token 集合被限定为所有能使 FSM 转移成功的 token，采样时在合法集合内做概率归一化。其本质是将『语法约束』转化为『解码时的掩码（mask）』，保证输出 100% 语法合法且满足 Schema 约束（如类型、枚举、必填字段）。路径三：结构化生成框架（如 Outlines、Guidance、LlamaIndex structured output），它们只是路径二的工程封装，内部仍是 FSM 或基于正则的约束采样。与前端已有概念的对比：TypeScript 的 interface 是编译期类型约束，运行时被擦除，只保证开发时静态检查；JSON Schema 是运行时数据契约，既用于校验也用于生成约束。Java 的接口是 OOP 的多态契约，描述行为签名；JSON 结构化输出中的 Schema 描述的是数据形状，不绑定行为。此外，前端中 JSON.parse 是确定性解析器，输入错误就抛异常；LLM 结构化输出中的解析是概率性生成的产物解析，错误是常态而非异常，因此必须引入重试、修复、约束解码等机制，这是与前端『接口返回 JSON 必然可解析』这一假设的本质差异。

### 3. 基础代码与实战验证
```text
以下为极简约束解码核心逻辑的文字化伪代码，展示 FSM 掩码机制的本质：

// 输入：schema = {"type":"object","required":["name"],"properties":{"name":{"type":"string"}}}
// 输入：vocab = 模型词表（token -> id 映射）
// 输入：logits = 模型当前步输出的原始概率分数（长度 = vocab.size）

1. 将 schema 编译为状态机 states 与转移表 transitions。
   // 状态示例：
   // S0 = 期望 '{'，合法 token 集合 = {'{'}（忽略空白 token 则加空白）
   // S1 = 期望 key 的引号，合法 token 集合 = {'"'} 或 空白
   // S2 = 期望 ':'，合法 token 集合 = {':', 空白}
   // S3 = 期望 value（string 类型），合法 token 集合 = {'"', 空白}
   // S4 = 期望 '}' 或 ','，合法 token 集合 = {'}', ',', 空白}

2. 维护当前状态 current_state，初始为 S0。
   // 每次生成一个 token 后，根据该 token 更新状态。

3. 在每一步生成时：
   mask = []
   for token_id, token_str in vocab:
       if can_transition(current_state, token_str):  // 检查该 token 是否能让 FSM 合法转移
           mask[token_id] = 1
       else:
           mask[token_id] = 0

   masked_logits = logits + (mask == 0 ? -inf : 0)  // 非法 token 概率置为负无穷
   next_token = sample(masked_logits)               // 从合法 token 中按概率采样

   current_state = transitions[current_state][next_token]

4. 重复直到到达接受状态（生成完整 JSON 文本）。

// 效果：生成的文本必然满足 schema，不存在语法错误。
// 若采用后处理路径，则伪代码为：
//   text = llm.generate(prompt)
//   try: obj = json.loads(text)
//   except: obj = llm.repair(text, schema)  // 或重试
// 对比：约束解码从源头保证合法性，后处理是在出口救火。
```

### 4. 常见误区与进阶思考
误区一：认为『prompt 里写了 JSON 格式，模型就会稳定输出合法 JSON』。实际上，token 采样是概率性的，长文本中引号、逗号、括号的遗漏是必然分布事件，且模型的 tokenizer 可能把数字、字符串边界拆成不连贯片段，仅靠 prompt 无法保证语法正确。正确认知是：prompt 只是软约束，必须配合约束解码或严格的后处理校验+重试机制才能进入生产。误区二：混淆 JSON Schema 校验与结构化输出。校验是事后判断数据是否合法，结构化输出是事前限制生成空间；前者无法修复非法输出，只能失败重试，后者才是根治手段。若仅用校验，在高并发或长输出场景下，重试成本会指数上升。

深度思考题：当约束解码的 FSM 与模型 tokenizer 的『多字节字符边界』交互时（例如一个中文字符被拆成两个 token，且 Schema 要求该字符串必须以某个 Unicode 码点结尾），FSM 的合法 token 集合应该如何构建才能既保证 Schema 约束又不错杀能组成合法字符的 token？这要求你理解 token 级语法约束与字节级/字符级编码之间的层级关系，真正掌握后你就能解释为什么某些结构化生成框架在非英文场景下会性能下降或产生意外拒绝。
