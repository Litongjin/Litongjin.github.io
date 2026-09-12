---
title: "每日基础技术总结 · 2026-09-13 · JSON 结构化输出"
date: 2026-09-13 07:01:32
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-13 · JSON 结构化输出

## 📚 今日主题

> **JSON 结构化输出**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
JSON 结构化输出（Structured Output）是指在 LLM 推理过程中，通过约束解码空间或后处理校验，使模型输出严格符合预定义 JSON Schema 的过程。其本质是将概率性的自然语言生成问题转化为受语法约束的序列生成问题——在词表上计算概率时，仅允许产生能构成合法 JSON 且满足 Schema 约束的 token。它解决的核心问题是 LLM 输出的不可控性，使下游程序无需模糊解析即可安全、确定地消费模型产物。在整个 AI 体系中，它位于模型推理层与应用契约层之间，是 Agent 工具调用、RAG 结果格式化、自动化流水线等场景的基础设施。专业工程师必须掌握它，因为结构化输出本质上定义了系统边界上的数据协议，与前端对接后端 API 时的强类型契约同构，是构建可靠 AI 应用的底线能力。

### 2. 底层原理剖析
底层机制分为两种主流路线。其一为约束解码（Constrained Decoding）：在模型自回归生成每个 token 时，维护一个有限状态机（FSM），该 FSM 由目标 JSON Schema 编译而来，状态转移边上有允许的 token 集合（以字节或 BPE 子词为单位）。每一步解码时，根据当前已生成的 token 序列计算出 FSM 状态，从而获得当前允许的 token 掩码（mask），对模型输出的 logits 应用掩码后执行 softmax 和采样，保证生成的任何序列都必然满足 Schema。该过程不依赖模型推理能力，只依赖词法和语法约束，时间复杂度为 O(1) 的掩码查询。其二为后处理校验与重试：模型先自由生成，再用 JSON parser 和 Schema validator 校验，失败则重新生成或进行修复。该路线的确定性弱，存在概率性失败，但实现简单。本质区别在于——约束解码是'从源头保证'，后处理是'事后检查'。与前端概念的对比：TS 的接口是编译期类型约束，擦除后运行时不存在，用于静态检查；JSON Schema 是运行期数据约束，需在运行时校验器（如 ajv）中执行；而约束解码中的 FSM 则更进一步，将 Schema 编译为生成器本身的语言约束，相当于把 TS 类型检查嵌入到 AST 生成的每一步中。接口描述的是'数据长什么样'，JSON Schema 描述的是'数据必须符合什么规则'，而结构化输出的 FSM 则决定'生成器每一步只能说哪些话'。前端中的 Zod 或 io-ts 在运行时结合了类型与解析，但依然是在值产生之后进行校验；约束解码则在值产生之前就锁定了合法性。

### 3. 基础代码与实战验证
为了验证原理，使用 Python 伪代码实现一个基于 FSM 的极简约束解码流程。假设模型 logits 为形状 [vocab_size] 的张量，tokenizer 将 token 映射为字符串片段，我们要强制模型生成一个形如 {"age": 整数} 的 JSON。关键代码：

```python
import torch
from typing import Set

# 1. 定义词法状态机：状态 0 为初始，状态 1 为 '{"' 后，状态 2 为 age 后，状态 3 为冒号后，状态 4 为整数，状态 5 为结束
# 每个状态允许的 token 前缀集合（实际中按字节/子词操作）
allowed_by_state = {
    0: {'{'},                     # 只能以 { 开始
    1: {'"age"'},                 # key 固定为 "age"
    2: {':'},                     # 冒号
    3: {'0','1','2','3','4','5','6','7','8','9','-'},  # 整数开头
    4: {'0','1','2','3','4','5','6','7','8','9','}', ' '},  # 继续整数或结束
    5: {'EOF'}
}

def compute_mask(state: int, logits: torch.Tensor) -> torch.Tensor:
    # 获取当前状态允许的 token 索引集合（假定 tokenizer 能映射片段到 id）
    allowed_token_ids: Set[int] = get_allowed_ids(allowed_by_state[state])
    mask = torch.full_like(logits, float('-inf'))
    for idx in allowed_token_ids:
        mask[idx] = 0  # 允许的 token 不改变 logits
    return mask + logits  # 实际上应用方式为 logits + mask，因为掩码中 -inf 会禁止采样

# 2. 模拟生成循环
state = 0
generated = []
for _ in range(10):
    logits = model(next_token_embeddings)  # 真实推理中每次输入拼接新 token
    masked_logits = compute_mask(state, logits)
    next_token = torch.multinomial(torch.softmax(masked_logits, dim=-1), 1)
    token_str = tokenizer.decode(next_token)
    generated.append(token_str)
    state = transition(state, token_str)  # 根据 token 推进状态机
    if state == 5:
        break
```

上述代码的核心逻辑：`compute_mask` 依据状态机的当前状态，将不允许的 token 的 logits 设为 -inf，这样 softmax 后这些 token 概率为 0，模型无法输出。`transition` 根据实际输出的 token 字符串更新状态，保证状态与已生成序列一致。真实实现（如 outlines、jsonformer）中状态机是编译成确定性有限自动机（DFA），支持任意 JSON Schema，且 token 允许集合通过提前计算在 BPE 词表上的映射获得，避免运行时逐 token 检查。

### 4. 常见误区与进阶思考
误区一：认为 JSON 结构化输出只需要在 prompt 里写上'请输出 JSON'即可。实践中模型可能因 tokenizer 边界、上下文遗忘或数学推理压力输出非法 JSON，导致下游 JSON.parse 崩溃。本质上这是把概率事件当确定事件处理，违背了工程中的契约优先原则。应该使用约束解码或在应用层强制用校验+重试机制。误区二：把 JSON Schema 当作 TS 接口来用，忽略 JSON Schema 的复杂性——诸如 additionalProperties、anyOf、$ref、oneOf 等关键字会显著影响约束解码状态机的规模和性能。直接对完整 JSON Schema 做 FSM 编译可能导致状态爆炸，而前端接口不涉及运行时开销。思考题：给定一个 JSON Schema 允许任意嵌套的数组和对象，但要求所有叶子节点必须是整数。在设计约束解码 FSM 时，如何处理"嵌套深度未知"这一特点？能否在 O(1) 常量空间内表示状态，还是必须依赖递归或堆栈？这揭示了结构化输出的本质是语言识别问题，与上下文无关语法（CFG）的可判定性有直接关联。
