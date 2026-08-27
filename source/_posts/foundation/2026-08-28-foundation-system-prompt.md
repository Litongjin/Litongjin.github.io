---
title: "每日基础技术总结 · 2026-08-28 · System Prompt 的角色设定"
date: 2026-08-28 06:55:48
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-28 · System Prompt 的角色设定

## 📚 今日主题

> **System Prompt 的角色设定**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
System Prompt 是注入到对话模型输入序列最前端的预置文本，属于 inference-time 的上下文控制手段，不修改模型权重。其本质是给条件概率分布 P(next_token | system_tokens, history_tokens) 增加一个先验约束，使生成结果偏向特定角色、风格、任务目标或输出格式。它解决的问题是：LLM 作为无状态函数逼近器，每次生成只依赖当前输入上下文，缺少稳定任务标识，因此需要显式提供任务语义作为生成的条件。在计算机/AI 体系中，它处于应用层与模型层之间，与 RAG、工具调用、微调并列，是 LLM 应用可配置性最强的控制面之一。前端工程师视角下，它近似于一个运行时注入的全局配置上下文，但它不含类型校验和强制约束，属于概率性契约。专业工程师必须掌握它，因为 Agent 的稳定性、安全性和可调试性都依赖 system prompt 与模型能力的准确匹配。

### 2. 底层原理剖析
Transformer 对多轮对话的处理方式是：将所有消息（system、user、assistant）展平为单一 token 序列，并以特殊分隔符标记角色。模型的前向传播不会对 system token 做特殊处理，而是通过多头自注意力将 system prompt 中的语义信息编码进每个 token 的表示。生成阶段，最后一个位置的隐藏向量经过 LM Head 映射为词表大小的 logits，softmax 后得到下一个 token 的概率。因此，system prompt 对输出的影响完全来源于它对前后文 token 的注意力加权，而不是独立的执行引擎。形式化地：P(y_t | X_system, X_user, Y_<t) = softmax(W * h_t)，其中 h_t 是 t 时刻的隐藏状态，由 system 部分的 key/value 参与计算。这解释了为什么 system prompt 是概率性引导而非命令。与前端概念的对比：TypeScript interface 是编译期结构契约，不参与运行时行为；System Prompt 是运行时语义契约，通过文本语义影响行为。Java interface 要求实现方强制遵守，而 system prompt 没有强制机制。更贴近的类比是前端 React 中的 Context：它是一个贯穿组件树的运行时配置，嵌套时后续 Context 可以遮蔽先前的值——但 Context 是确定性的，而 system prompt 是随机上下文中的偏置项。另外，system prompt 在 API 中的位置并不赋予它硬优先级；若后续用户消息包含更强的指令（如“忽略 system”），模型的注意力权重可能被重新分配，导致 system 被抑制。

### 3. 基础代码与实战验证
```text
# 使用 OpenAI client 验证 system prompt 的作用（仅依赖 openai 包）
from openai import OpenAI

client = OpenAI()

def ask(system, user):
    # chat.completions.create 将 messages 序列化为 JSON，经 HTTP 发送给推理服务。
    # 服务端把 messages 拼接为 token 序列，system_prompt 位于序列最前，
    # 和其他 token 一起参与 transformer 自注意力计算，进而调整输出分布。
    resp = client.chat.completions.create(
        model='gpt-4o-mini',
        messages=[
            {'role': 'system', 'content': system},
            {'role': 'user', 'content': user}
        ],
        temperature=0.0
    )
    return resp.choices[0].message.content

# 基线：无 system prompt 时，模型按默认统计模式输出
print(ask('', '写一首描写秋天的短诗'))

# 实验：system prompt 限定角色和格式后，输出被约束在指定语义空间中
print(ask('你是严谨的语文教师，只输出七言绝句且必须包含“梧桐”。', '写一首描写秋天的短诗'))
```

### 4. 常见误区与进阶思考
1. 误区：误以为 system prompt 是硬性控制，模型必然服从。事实上它只改变条件概率分布，服从程度受模型预训练、上下文冲突、采样随机性影响；典型的“忽略 system 指令”注入攻击可以改写输出。工程上需要将它视作与用户输入同级的、可竞争的文本，而不是不可变的配置。
2. 误区：通过不断累加规则来增强角色稳定性。这会导致上下文变长，注意力被稀释，系统提示中较早的规则更容易被遗忘。工程上应将关键约束前置、冗余置后，或结合 logit 约束/输出校验来实现硬保证。
思考题：在相同 system prompt 下，将其从第一段移动到多轮对话的中间（与用户消息交错）时，模型遵循指令的概率显著下降。请从 Transformer 的自注意力模式与相对位置编码出发，解释为什么位置越靠前，对后续生成的全局影响越强？
