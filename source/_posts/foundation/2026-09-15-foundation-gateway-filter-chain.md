---
title: "每日基础技术总结 · 2026-09-15 · API 网关：Spring Cloud Gateway 过滤器链"
date: 2026-09-15 08:00:00
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-15 · API 网关：Spring Cloud Gateway 过滤器链

## 📚 今日主题

> **API 网关：Spring Cloud Gateway 过滤器链**（Java 后端与 Spring 生态）

### 1. 核心概念速览
Spring Cloud Gateway 的核心本质是一个基于 WebFlux 与 Netty 的响应式网关，其请求处理模型可被抽象为一条由路由谓词（Predicate）与过滤器（Filter）构成的链式管道。过滤器链（Filter Chain）是网关对进入的请求和返回的响应进行横切处理的核心机制，它解决了跨路由的通用逻辑（如鉴权、限流、头改写、日志、熔断）复用与顺序编排问题。整个链的执行基于责任链模式：请求依次通过所有匹配的 GlobalFilter 与 GatewayFilter，每个过滤器在调用链上下文中完成自身逻辑后，再调用 `chain.filter(exchange)` 把控制权交给下一个过滤器，直至最终将请求转发到下游目标服务。它位于微服务架构的南北向流量入口，连接客户端与服务集群，是实现统一治理、安全隔离与可观测性的一道可编程屏障。专业工程师必须掌握它，因为网关过滤器链是分布式系统切面能力（Aspect Orientation）在网络层的具体落地，理解其生命周期、顺序控制与响应式语义，是驾驭生产级微服务基础设施而不是停留在配置调用的前提。

### 2. 底层原理剖析
过滤器链的底层运行机制是 Reactor 中的 Mono 链式装配与订阅执行。当请求到达 GatewayHandlerMapping 时，它会根据路由谓词找到匹配的 Route，然后构造一个 `FilteringWebHandler` 并执行。`FilteringWebHandler` 内部维护一个过滤器的有序列表：所有的 GlobalFilter 会与当前路由下的 GatewayFilter 合并在同一条链中，合并时通过与 `@Order` 注解或 `Ordered` 接口计算的顺序值（order）进行升序排序。执行阶段，处理器会递归地构建一个 `DefaultGatewayFilterChain`：它保存一个过滤器数组和一个指向当前处理位置的索引（`index`）。每次调用 `chain.filter(exchange)` 时，链式对象从当前位置取出下一个过滤器并执行，同时将 `index+1` 生成新的链对象传入过滤器，从而形成依次推进的责任链。由于所有逻辑都运行在 Reactor 的异步线程模型上，`filter(exchange, chain)` 返回的是一个 `Mono<Void>`，过滤器通过 `return chain.filter(exchange)` 或在其前后追加 `doOnNext`、`flatMap` 等操作来织入响应阶段逻辑，而没有显式调用 `chain.filter` 的过滤器则直接中断链，产生短路效果（比如鉴权失败返回 401）。最终，链的末端由 `NettyRoutingFilter` 将请求通过 HTTP 客户端代理到目标 URI，响应再沿调用栈逆向回溯。

与前端概念的对比：Java 中的过滤器链与前端中 Express/Koa 的中间件模型有本质亲缘，但最大差异在于响应式非阻塞。Koa 的洋葱模型是同步/异步回调栈，中间件通过 `await next()` 控制流转，作用域天然基于函数栈；而 Gateway 的过滤器链是 Mono 组装，没有承载请求上下文的调用栈，所有状态必须通过 `ServerWebExchange` 显式传递，过滤器的先后顺序不依赖代码嵌套，而是全局排序后线性展开。另外，Java 的接口与 TS 接口的区别：Java 接口更强调契约与多态，TS 接口本质是结构类型（structural typing）的编译期约束；在这个场景中，`GlobalFilter` 和 `GatewayFilter` 都是 Java 接口，它们约束了过滤器必须实现 `filter` 方法，并且借助 `Ordered` 接口描述顺序，而 TS 更倾向于通过函数签名或装饰器实现类似中间件约定。

### 3. 基础代码与实战验证
下面是一个极简但完整的 Spring Cloud Gateway 过滤器链验证代码，仅依赖 Spring WebFlux 与 Gateway 核心，展示如何编写自定义 GlobalFilter、控制顺序、挂载到链上并观察执行顺序。

```java
// 依赖：spring-cloud-starter-gateway（内含 WebFlux）

import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.core.Ordered;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;
import org.springframework.stereotype.Component;

/**
 * 第一个过滤器：order=1，先执行
 */
@Component
public class FirstFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        System.out.println("[FirstFilter] before chain");
        // 向 exchange 写入属性，供后续过滤器使用
        exchange.getAttributes().put("startTime", System.currentTimeMillis());
        return chain.filter(exchange)
            .doFinally(signal -> System.out.println("[FirstFilter] after chain"));
    }

    @Override
    public int getOrder() {
        return 1; // 值越小优先级越高
    }
}

/**
 * 第二个过滤器：order=2，在 First 之后执行
 */
@Component
public class SecondFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        System.out.println("[SecondFilter] before chain");
        return chain.filter(exchange)
            .doFinally(signal -> System.out.println("[SecondFilter] after chain"));
    }

    @Override
    public int getOrder() {
        return 2;
    }
}

// application.yml 配置一条路由，将 /api/** 转发到下游服务
// spring:
//   cloud:
//     gateway:
//       routes:
//         - id: example
//           uri: http://localhost:8081
//           predicates:
//             - Path=/api/**
//           filters:
//             - StripPrefix=1
```

关键行注释：
- `chain.filter(exchange)`：将控制权传递给链中下一个过滤器；返回值是 `Mono<Void>`，代表整个下游过滤链（包括最终转发）完成的异步信号。
- `doFinally(...)`：在响应式流终结（完成/错误/取消）时回调，用于模拟“链路返回后”的逆向处理，这正是洋葱模型在 Reactor 上的体现。
- `getOrder()`：网关通过该数值决定过滤器在链中的先后位置；默认全局过滤器同样参与排序，所以顺序是可控且全局统一的。
- `exchange.getAttributes()`：跨过滤器传递数据的唯一可靠方式，类似前端请求上下文对象 `ctx.state`，但底层由 Netty 的 Channel 属性扩展而来。

实际运行时，请求访问网关任意匹配路由的路径，控制台将依次输出 `[FirstFilter] before chain` → `[SecondFilter] before chain` → 下游响应 → `[SecondFilter] after chain` → `[FirstFilter] after chain`，验证了链式顺序与逆向回调。

### 4. 常见误区与进阶思考
1. 认知误区：误认为 `chain.filter(exchange)` 是同步阻塞调用，可以在其后直接操作响应内容。实际上，过滤器链是异步的，`chain.filter(exchange)` 返回 `Mono<Void>`，响应体可能在调用返回时尚未写入。正确做法是使用 `then`、`flatMap` 或 `doFinally` 等响应式操作符来在后续阶段处理响应，否则会导致读取到空或不完整的响应。
2. 认知误区：混淆 GlobalFilter 与 GatewayFilter 的顺序和装配方式。GlobalFilter 由 Spring 自动发现并应用所有路由，而 GatewayFilter 仅被关联到特定路由，二者合并后统一排序执行。若不显式实现 `Ordered`，默认顺序为 0，可能导致全局过滤器意外先于路由过滤器执行，产生难以排查的副作用。

思考题：假设你编写了一个 GlobalFilter 去读取响应体的内容并打印日志，同时又在路由级别配置了一个 `ModifyResponseBodyGatewayFilterFactory`，两者都会尝试消费响应体。在响应式流中，数据只能被消费一次，那么网关内部是如何通过 `ServerWebExchange` 对响应体进行缓存或重放，从而让两个过滤器都能读取到原始响应体的？请从 Reactor 的 `Mono` 复用机制与 `ServerWebExchange` 的 `getResponse()` 装饰器角度解释。
