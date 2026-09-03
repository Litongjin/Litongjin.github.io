---
title: "每日基础技术总结 · 2026-09-04 · JSON 结构化输出"
date: 2026-09-04 07:01:36
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-04 · JSON 结构化输出

## 📚 今日主题

> **JSON 结构化输出**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
JSON 结构化输出（Structured Outputs / 约束解码）指在 LLM 自回归解码阶段，将 JSON 文法与 JSON Schema 编译为确定性的 token 级状态机（FSM/下推自动机），每一步生成时硬性屏蔽所有会导致最终序列违反 Schema 的 token，使输出必然落在合法语言 L(Schema) 内。其本质不是『生成后校验』，而是『生成前约束』：将开放词表上的概率分布截断到合法子空间并重新归一化，把格式正确性从概率问题转化为计算问题。它解决的核心矛盾是：语言模型的自由概率采样与机器可解析的形式化数据契约之间不可调和的冲突。该机制位于 LLM 推理与确定性程序（函数调用、Agent 循环、RAG 管线、API 编排）的边界，是模型输出可编程化的底层基础设施。专业工程师必须掌握它，因为任何以 JSON 作为进程间契约的系统，若仅依赖提示词或解析容错来保证格式，其可靠性上限是概率性的；结构化输出将这一上限提升为确定性的，是 Agent 系统能否稳定进入生产环境的决定性因素之一。

### 2. 底层原理剖析
底层机制分三层。
1. 文法编译层：JSON 由 RFC 8259 定义，是上下文无关文法；JSON Schema 是在该文法之上施加的类型与结构谓词（required、type、additionalProperties、enum 等）。将 Schema 编译为下推自动机或递归状态机——嵌套结构要求超越正则语言的表达能力，因此实际实现（如 Outlines、Pydantic、llama.cpp GBNF）采用类似 Earley 解析器的状态集合，每个状态记录『已生成 token 前缀』对应的解析进度与 Schema 约束上下文。
2. 解码约束层：第 t 步模型输出词表 V 上的 logits。约束解码器依据当前状态计算合法 token 集合 M_t，将 i ∉ M_t 的 logit 置为 -inf，再对 M_t 上的分布做 softmax 重归一化后采样。等价于在条件分布 P(y|x) 上施加硬约束：P'(y|x) ∝ P(y|x) · 1[y ∈ L(Schema)]。由于每一步都保证状态转移合法，最终序列必然可解析且满足 Schema——不是高概率，而是必然。
3. 工程实现层：OpenAI 的 strict structured outputs、vLLM 的 guided decoding、Outlines、llama.cpp 的 GBNF grammar 均为此原理，差异仅在状态机构造算法与批量掩码优化。注意区分两个强度层级：JSON mode（如早期 response_format={'type':'json_object'}）只保证语法合法，用一个识别 JSON 文法的自动机即可；JSON Schema mode 还需将 Schema 谓词编译进转移条件，保证语义约束。
与前端已有概念的对比：TS interface 是编译期擦除的静态类型，只存在于类型空间，运行时零存在感；JSON Schema 是运行时数据契约，而约束解码让该契约从『事后校验』升级为『生成期强制执行』——类比理解，TS 类型相当于编译期检查，结构化输出相当于把类型检查器搬进了 CPU 的指令执行路径。前端 zod/yup 校验是先取值再判错（后验，失败只能回滚）；约束解码是采样前剪枝（先验，非法状态不可达）。前者是测试，后者是编译。这与 Java interface（编译期与 JVM 边界的契约强制）在『契约由编译器/运行时强制执行而非靠约定』这一点上同构，区别是 Java 约束的是方法调用，结构化输出约束的是 token 序列的转移图。

### 3. 基础代码与实战验证
```text
# 极简验证：约束解码的本质 = 非法 token 的 logit 屏蔽 + 截断分布重归一化
# 目标语言：合法 JSON 数字（正则 -?[0-9]+(\.[0-9]+)?），编译为五状态 FSM
# 状态编码：0=起始, 1=负号后, 2=整数部分, 3=小数点后, 4=小数部分；-1=非法转移

import random

def step(state, ch):
    if state == 0 and ch == '-': return 1
    if state == 0 and ch.isdigit(): return 2
    if state == 1 and ch.isdigit(): return 2
    if state == 2 and ch.isdigit(): return 2
    if state == 2 and ch == '.': return 3
    if state == 3 and ch.isdigit(): return 4
    if state == 4 and ch.isdigit(): return 4
    return -1

def allowed(state):
    # 由当前状态反推合法 token 集合：凡 step 返回非 -1 的字符
    return {c for c in '0123456789-.' if step(state, c) >= 0}

def is_accept(state):
    return state in (2, 4)  # 合法数字不能以 '-' 或 '.' 结尾

def constrained_sample(probs, allowed_set):
    # 核心机制：非法 token 概率置 0，合法 token 重新归一化
    masked = {c: (p if c in allowed_set else 0.0) for c, p in probs.items()}
    total = sum(masked.values())
    norm = {c: p / total for c, p in masked.items()}  # 截断后重归一化
    r = random.random(); acc = 0.0
    for c, p in norm.items():
        acc += p
        if r < acc:
            return c

# 模拟一个语感很差的模型：50% 概率倾向输出非法字符 'x'
model_probs = {'1': 0.2, '2': 0.2, 'x': 0.5, '-': 0.1}

state, out = 0, ''
while not is_accept(state):
    c = constrained_sample(model_probs, allowed(state))  # 采样空间已被硬剪枝
    out += c
    state = step(state, c)

print(out)  # 无论模型分布多偏好 'x'，输出必为合法 JSON 数字
# 结论：合法性由 FSM 硬保证，模型分布只在合法集合内部决定『选哪个』

# 生产环境中的同一机制（OpenAI Structured Outputs）：
# strict=True 即开启服务端约束解码，而非依赖提示词约束
from openai import OpenAI
import json
client = OpenAI()
resp = client.chat.completions.create(
    model='gpt-4o',
    messages=[{'role': 'user', 'content': '返回北京市今天的气温'}],
    response_format={'type': 'json_schema', 'json_schema': {
        'name': 'weather', 'strict': True,
        'schema': {'type': 'object',
                   'properties': {'temperature': {'type': 'number'}},
                   'required': ['temperature'],
                   'additionalProperties': False}
    }}
)
data = json.loads(resp.choices[0].message.content)  # 必然可解析且满足 schema
```

### 4. 常见误区与进阶思考
误区一：认为『结构化输出 = 提示词约束 + JSON.parse 兜底』。提示词是软约束，模型以概率服从；JSON.parse 是后验检测，能发现非法却无法阻止非法 token 的产生。真实的结构化输出是解码期的硬约束：非法 token 在采样前 logit 已被置为 -inf，非法状态在概率空间不可达。工程上若只用提示词 + 解析重试，错误率随输出长度与 Schema 复杂度指数上升，且失败路径无法穷举。
误区二：混淆『JSON mode』与『JSON Schema mode』。response_format={'type':'json_object'} 只保证输出是合法 JSON 语法，不保证满足任何字段级约束——缺少 required 字段、出现额外字段、类型不符都是合法 JSON。只有将 Schema 编译进解码状态机的严格模式（strict:true / guided_json）才提供语义级保证。同理，『校验』与『约束』不是一回事：校验是事后判定，约束是事前剪枝，前者无法提升一次生成的合法性概率，后者将合法性概率直接置为 1。
思考题：设第 t 步合法 token 集合 M_t 的原始概率质量 Z_t = Σ_{i∈M_t} softmax(logits)_i 极低（如 0.001）。约束解码将分布截断并重归一化，等价于在分布上施加了一个巨大的 KL 惩罚。请分析：当采样温度 τ→0（贪心）时，这种强制截断会如何影响后续 token 的长期依赖？而当 τ 较大时，为何这种截断的破坏性反而较小？由此推导结构化输出的本质代价是什么——即『格式确定性』与『模型分布自然度』之间的兑换率由什么决定？
