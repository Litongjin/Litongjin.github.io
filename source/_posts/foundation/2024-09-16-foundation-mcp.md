---
title: "每日基础技术总结 · 2024-09-16 · MCP 协议：模型上下文与工具标准化"
date: 2024-09-16 20:00:00
categories: [技术分享]
tags: ["技术分享", "AI / LLM 工程实战"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-09-16 · MCP 协议：模型上下文与工具标准化

## 📚 今日主题

> **MCP 协议：模型上下文与工具标准化**（AI / LLM 工程实战）

### 1. 核心概念速览
MCP (Model Context Protocol) 是 Anthropic 发起并开源的开放标准，旨在解决 LLM 应用中上下文管理与工具调用的碎片化问题。其本质是一个标准化的通信协议层，定义了模型（Host）、客户端（Client）与数据源/工具提供端（Server）之间的交互规范。它解决了两大核心痛点：1. 标准化能力接入：将不同后端服务、API 或文件系统封装为统一的可发现、可调用的 'Tool' 和可加载的 'Resource'；2. 上下文标准化：通过 URI 和 JSON-RPC 机制，让模型能够以统一方式读取结构化与非结构化数据。在 AI 工程体系中，MCP 位于应用逻辑与大模型推理引擎之间，充当‘适配器’角色。专业工程师必须掌握，因为它消除了自定义集成代码的维护成本，实现了模型能力的插件化与热插拔，是构建模块化、可扩展 Agent 架构的基础设施。

### 2. 底层原理剖析
MCP 基于 JSON-RPC 2.0 实现通信，传输层支持 Stdio（本地进程间通信）和 HTTP + SSE（远程流式通信）。

底层运行机制：
1. 握手阶段 (Initialize): Client 向 Server 发送 Initialize 请求，携带协议版本及 Capability（支持的工具列表、资源模式等）。Server 返回自身支持的 Capabilities。
2. 发现阶段 (Discovery): Client 枚举 Server 提供的 Resources (资源) 和 Tools (工具)，获取元数据 (Metadata)，如名称、描述、参数 Schema (JSON Schema)。
3. 执行阶段 (Invocation):
   - 工具调用: Client 构造 Tool Call Request，传入符合 JSON Schema 的参数。Server 执行逻辑后返回 Result。
   - 资源读取: Client 发起 Read Resource Request (URI)。Server 返回 content (text/binary/base64) 及 mimeType。

对比前端接口概念：
- Java Interface / TS Interface: 仅定义类型契约 (Static Contract)，不规定序列化格式与传输语义。MCP 不仅定义了数据结构 (JSON Schema)，还定义了生命周期管理 (Initialize/Shutdown)、错误处理 (Error Codes) 以及双向 RPC 语义。MCP 的 'Tool' 类似于 Web API Endpoint，但其参数校验由 Client 侧根据 Model 生成的意图动态执行，而非静态编译期检查。
- 核心差异: MCP 强调 'Context Injection' (上下文注入)，即资源读取直接作为模型的输入上下文的一部分，而传统 API 调用通常被视为外部动作。

### 3. 基础代码与实战验证
```text
// 简化版 Python 伪代码演示 MCP Client 调用逻辑
# 假设使用 mcp SDK
import json
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
    # 1. 建立连接: 类似 subprocess.Popen，但封装了 JSON-RPC 收发
    server_params = StdioServerParameters(
        command="python", args=["mcp_server_tool.py"] 
    )
    async with stdio_client(server_params) as (read_stream, write_stream):
        async with ClientSession(read_stream, write_stream) as session:
            # 2. 初始化握手: 协商 Capabilities
            await session.initialize()
            
            # 3. 资源发现: 获取可用工具列表
            tools_response = await session.list_tools()
            tool_names = [t.name for t in tools_response.tools]
            
            # 4. 工具调用: 模拟 LLM 生成参数并执行
            # 注意: 实际中参数由 LLM 生成，此处硬编码验证
            result = await session.call_tool(
                name="get_weather",
                arguments={"location": "Beijing", "days": 3}
            )
            
            # 5. 结果解析: 返回内容为结构化数据 (TextContent)
            print(json.dumps([c.text for c in result.content], indent=2))
```

### 4. 常见误区与进阶思考
['误区一：将 MCP 视为框架而非协议。MCP 本身不包含状态机管理、重试逻辑或复杂的业务编排，它是一个纯粹的 I/O 抽象层。开发者仍需自行构建调用链、权限控制和数据清洗管道。过度依赖 MCP 库会导致对底层 JSON-RPC 消息流的失控。', '误区二：混淆 Tool 与 Resource 的使用场景。Resource 适用于读取静态或半静态数据作为上下文（如文档、配置文件），具有只读性语义；Tool 适用于产生副作用或即时计算的 action。将复杂查询封装为 Resource 可能导致上下文窗口浪费，反之将简单配置作为 Tool 调用会增加不必要的 RPC 开销。', '思考题：在一个高并发 Agent 系统中，如果多个并行子 Agent 同时访问同一 MCP Server 的同一个受限资源（如数据库连接池较小的文件服务），MCP 协议层的无状态特性（Stateless JSON-RPC）会如何放大后端服务瓶颈？应如何在 Client 层或 Server 层设计资源锁定或缓存策略以违背协议的纯粹性从而换取性能？']
