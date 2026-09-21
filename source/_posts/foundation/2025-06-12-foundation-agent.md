---
title: "每日基础技术总结 · 2025-06-12 · Agent 架构：规划/记忆/工具循环"
date: 2025-06-12 20:00:00
categories: [技术分享]
tags: ["技术分享", "AI / LLM 工程实战"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-06-12 · Agent 架构：规划/记忆/工具循环

## 📚 今日主题

> **Agent 架构：规划/记忆/工具循环**（AI / LLM 工程实战）

### 1. 核心概念速览
Agent 架构中的规划(Planning)/记忆(Memory)/工具使用(Tool Use)循环是构建自主智能体的核心控制流机制。其本质是将大型语言模型（LLM）从一个静态的概率分布生成器转化为具备状态机特性的动态执行引擎。

1. 规划：解决‘做什么’的问题，通过分解目标、生成子任务序列或调整策略，处理非结构化问题空间。机制上通常依赖链式思维（CoT）或树搜索（ToT）在隐式空间中遍历解路径。
2. 记忆：解决‘状态持久化与上下文约束’的问题。分为短期记忆（工作上下文窗口，用于当前推理的支撑）和长期记忆（向量数据库/知识图谱，用于跨会话的知识检索与关联）。机制上涉及状态压缩、检索增强生成（RAG）及上下文窗口管理。
3. 工具使用：解决‘能力边界扩展’的问题。LLM 本身仅擅长文本处理，需通过 API 调用外部函数（Calculator, Database, Web Search 等）获取实时信息或执行动作。机制上是基于模式匹配的函数调用协议（如 ReAct 协议）。

该循环打破了 LLM 的孤立性，使其具备感知-决策-行动的闭环能力。对于全栈工程师而言，掌握此机制意味着从‘编写确定性代码逻辑’转向‘设计概率性执行流程与状态管理’，这是构建复杂后端 AI 应用的基础设施级能力。

### 2. 底层原理剖析
该循环遵循典型的 Agent 循环（Agent Loop）算法结构，可类比为前端 React 的 Effect Hook + State Update 机制，但运行在异步 I/O 和分布式环境中。

底层运行机制伪代码如下：
while not terminal_condition:
    # 1. Memory Retrieval (感知阶段)
    context = memory.query(current_state, history)

    # 2. Planning & Reasoning (决策阶段)
    # LLM 根据上下文生成下一步行动指令，可能包含自然语言思考过程
    llm_output = model.generate(context=context, prompt_template=planning_prompt)

    if llm_output.action == 'finish':
        break
    elif llm_output.action == 'tool_use':
        # 3. Tool Execution (执行阶段)
        tool_result = execute_tool(llm_output.tool_name, llm_output.arguments)
        # 将结果写回记忆，形成闭环反馈
        memory.append(action=llm_output, result=tool_result)
        # 更新上下文窗口，注意 Token 限制导致的上下文滑动或摘要压缩
        context = context.merge(tool_result).trim_max_tokens()
    else:
        # 纯文本推理步骤，无外部副作用
        memory.append(step=llm_output)
        context = update_context(context, llm_output)

与前端概念对比：
- 前端 Promise/Async Await：处理的是已知接口的异步等待；Agent 的工具调用是‘动态路由’，函数名和参数由 LLM 运行时决定，类似于 JS 中的 eval 或 dynamic function invocation，但受限于安全沙箱。
- 前端 State Management (Redux/Zustand)：Agent 的记忆模块不仅存储数据，还存储‘推理轨迹’。不同之处在于，前端的 State 变化由事件触发且确定性强；Agent 的 Context 变化由概率性生成且存在‘漂移风险’（Garbling），需要显式的状态压缩策略（如 Summary Buffer）。

### 3. 基础代码与实战验证
```text
# 极简示例：模拟一个具备简单规划和工具调用的 Agent 核心循环
import json
from typing import Dict, List

# 模拟外部工具集
tool_registry = {
    "get_weather": lambda city: {"temp": 25, "condition": "Sunny"},
    "search_code": lambda query: {"repo": "example", "file": "index.ts"}
}

def run_agent_loop(initial_goal: str, max_steps: int = 5):
    # Memory: 初始化为空的历史记录，包含系统提示和目标
    memory: List[Dict] = [{
        "role": "user",
        "content": f"Goal: {initial_goal}"
    }]
    
    for step in range(max_steps):
        print(f"--- Step {step + 1}: Planning ---")
        
        # 1. 调用 LLM 进行规划（此处简化为规则匹配，实际应为 LLM call）
        # 真实场景中：response = openai.ChatCompletion.create(messages=memory)
        simulated_llm_response = determine_next_action(memory)
        action_type = simulated_llm_response.get("type")
        
        if action_type == "finish":
            print(f"Agent finished with conclusion: {simulated_llm_response.get('output')}")
            return simulated_llm_response.get('output')
            
        elif action_type == "call_tool":
            tool_name = simulated_llm_response.get("tool_name")
            params = simulated_llm_response.get("params")
            
            print(f"Executing tool: {tool_name}({params})")
            # 2. 工具执行
            result = tool_registry.get(tool_name)(**params)
            
            # 3. 记忆更新：将动作和结果追加到历史上下文中
            memory.append({
                "role": "assistant",
                "content": f"Executed {tool_name}",
                "tool_call_id": f"call_{step}"
            })
            memory.append({
                "role": "tool",
                "content": json.dumps(result),
                "tool_call_id": f"call_{step}"
            })
            # 关键机制：上下文窗口增长。若超过 Limit，需在此处实施 RAG 或 Summarization
        else:
            # 内部推理步骤，直接写入内存供后续参考
            memory.append(simulated_llm_response)

def determine_next_action(history: List[Dict]):
    # 这里是占位符，代表 LLM 的推理过程
    # 返回格式必须严格符合 JSON Schema 以便程序解析
    # 例如: {"type": "call_tool", "tool_name": "get_weather", "params": {"city": "Beijing"}}
    return {
        "type": "call_tool",
        "tool_name": "get_weather",
        "params": {"city": "Beijing"}
    }

# run_agent_loop("Check weather and code a repo")
```

### 4. 常见误区与进阶思考
1. **上下文爆炸与幻觉累积（Context Bleeding & Hallucination Accumulation）**：
   专业误区在于认为无限增加 History 总能提升效果。实际上，超出 LLM 有效注意力范围（Effective Attention Span）后，旧信息会被噪声掩盖，导致中间推理步骤的错误被下一轮继承放大。解决之道不是单纯堆砌 Token，而是引入‘记忆压缩’（Summarization）或‘选择性检索’（Selective Retriever）机制，仅保留高信噪比的状态片段。

2. **同步阻塞误用（Synchronous Blocking）**：
   在前端思维惯性下，容易将 Agent 循环写为串行阻塞调用。然而，工业级 Agent 往往需要并行调用多个工具（Parallel Tool Calling）以提升延迟，或在规划阶段使用多播机制（Fan-out/Fan-in）。必须理解 Agent 本质是一个基于事件的异步状态机，而非线性脚本。

思考题：
在长周期任务中，当‘记忆’模块因 Token 限制被迫丢弃早期关键指令时，如何在架构层面保证 Agent 不偏离初始‘规划’目标？请结合向量数据库的分层索引或元数据过滤机制进行阐述。
