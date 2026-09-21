---
title: "每日基础技术总结 · 2025-08-03 · API 网关：Spring Cloud Gateway 过滤器链"
date: 2025-08-03 20:00:00
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-08-03 · API 网关：Spring Cloud Gateway 过滤器链

## 📚 今日主题

> **API 网关：Spring Cloud Gateway 过滤器链**（Java 后端与 Spring 生态）

### 1. 核心概念速览
Spring Cloud Gateway 是基于 WebFlux（Reactor Netty）实现的非阻塞 API 网关，其核心机制是责任链模式与 Reactor 响应式数据流的结合。本质上是 HTTP 请求生命周期中的预处理拦截器链和后置处理断言链。它解决的是跨切面关注点分离问题：在路由分发前执行鉴权、限流、日志、参数修正等横切逻辑，在响应返回后执行头修改、性能监控等后处理逻辑。在后端生态中，它是微服务架构的服务入口边界控制器；对于前端工程师，理解它有助于打通从浏览器到后端服务的完整数据流转语义，消除对‘黑盒’网关的依赖焦虑，掌握高并发下零拷贝与背压控制的实现基础。

### 2. 底层原理剖析
1. 架构模型：基于 `GlobalFilter`（全局）和 `GatewayFilter`（局部）接口定义过滤器节点。
2. 执行时序：构建阶段（Bootstrap）将 FilterFactoryBean 实例化为具体的 Filter 对象，并注册到 KryoNetty/Reactor 的执行上下文中。运行阶段（Runtime）遵循 '先入后出' 的回溯原则：
   - Pre-阶段：请求到达 -> 过滤器1 Pre逻辑 -> 过滤器2 Pre逻辑 ... -> 路由匹配 -> Target Service
   - Post-阶段：Target Service 响应 -> 过滤器N-1 Post逻辑 ... -> 过滤器1 Post逻辑 -> 客户端
3. 核心接口差异对比：
   - Java 接口 vs TS 接口：TS 接口是编译时静态类型约束，用于契约描述；Java `GatewayFilter` 接口不仅是类型约束，更是可执行的行为抽象。`filter(ServerWebExchange exchange, GatewayFilterChain chain)` 方法接收一个函数式接口 `chain`，调用 `chain.filter(exchange)` 即代表 '透传当前请求至下一个过滤器节点或目标服务'。
4. 底层机制：利用 Reactor Publisher 特性，通过异步非阻塞 IO 实现过滤器的状态保持与恢复。每个过滤器持有上游连接池或目标地址引用，通过 `ServerWebExchange` 交换上下文传递属性（Attributes）。

### 3. 基础代码与实战验证
```text
// 核心接口定义解析
public interface GlobalFilter {
    /**
     * 
     * @param exchange 包含请求、响应及上下文属性的容器
     * @param chain    责任链本身，调用其 filter 方法以推进流程
     * @return Mono<Void> 返回完成信号，若报错则 emitError
     */
    Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}

// 极简实现示例：记录请求耗时过滤器
public class TimingFilter implements GlobalFilter {
    private static final Logger log = LoggerFactory.getLogger(TimingFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 1. 前置处理：标记开始时间存入 Exchange Attribute
        long startTime = System.currentTimeMillis();
        exchange.getAttributes().put(START_TIME_ATTR, startTime);
        
        // 2. 关键步骤：调用 chain.filter() 透传请求
        // 此处发生递归调用进入下一层过滤器或目标服务
        return chain.filter(exchange).doOnTerminate(() -> {
            // 3. 后置处理：请求结束（无论成功或失败）触发回调
            long duration = System.currentTimeMillis() - startTime;
            log.info(
```
