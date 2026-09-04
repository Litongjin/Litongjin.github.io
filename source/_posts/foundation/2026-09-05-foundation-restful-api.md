---
title: "每日基础技术总结 · 2026-09-05 · RESTful API 设计规范"
date: 2026-09-05 07:14:02
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-05 · RESTful API 设计规范

## 📚 今日主题

> **RESTful API 设计规范**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
REST（Representational State Transfer）是一种面向资源的分布式超媒体系统架构风格，由 Roy Fielding 在博士论文中定义。它不是协议，而是一组约束条件：客户端-服务端、无状态、可缓存、统一接口、分层系统、按需代码（可选）。其本质是对资源的抽象建模，通过统一接口（URI 标识资源、HTTP 方法表达操作、表示（representation）传递状态）驱动状态转移。它解决的问题是：在互联网规模下，如何让不同系统之间以松散耦合、可伸缩、可缓存的方式进行交互。RESTful API 是遵循这些约束的 HTTP API 设计规范，但并非所有 HTTP API 都是 RESTful。在计算机体系中的位置：位于表示层（表示状态转移层），是 Web 架构的核心范式之一，也是 AI/LLM Agent 系统中工具调用、服务间通信（如 MCP 协议底层的 HTTP+JSON）的基础。专业工程师必须掌握它，因为它是系统间互操作的事实标准，理解其约束才能真正设计出可进化、可缓存、可测试的分布式接口。
