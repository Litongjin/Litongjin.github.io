---
title: "每日基础技术总结 · 2025-11-05 · Spring MVC：DispatcherServlet 请求处理流程"
date: 2025-11-05 20:00:00
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-11-05 · Spring MVC：DispatcherServlet 请求处理流程

## 📚 今日主题

> **Spring MVC：DispatcherServlet 请求处理流程**（Java 后端与 Spring 生态）

### 1. 核心概念速览
DispatcherServlet 是 Spring MVC 架构中的中央控制器（Front Controller），本质是基于 Servlet API 的 HTTP 请求分发器。它解决的核心问题是解耦客户端请求与具体业务逻辑，通过策略模式将请求生命周期的各个阶段（解析、处理、视图渲染）标准化。在计算机体系中，它位于应用服务器（Tomcat/Jetty）之上、业务代码之下，充当协议适配层。专业工程师必须掌握其流程，因为这是理解 Spring Boot 自动装配机制、拦截器执行顺序、事务传播行为以及微服务网关路由逻辑的基础骨架。

### 2. 底层原理剖析
流程遵循严格的状态机模型：
1. 接收：Tomcat 线程池将 HttpServletRequest/Response 注入 DispatcherServlet。
2. 映射（HandlerMapping）：根据 URL、HTTP Method、Header 查找对应的 HandlerMethod（Controller 方法），获取 HandlerInterceptor 链。
3. 适配（HandlerAdapter）：将具体的 Handler 包装为统一的 Adapter 调用，解决不同 Controller 实现方式（如 @RestController, @Controller + ModelMap）的差异。
4. 执行：先执行 PreHandle -> 反射调用目标方法 -> 返回 ModelAndView（或直接写入 ResponseBody）。
5. 视图解析：若无视图需渲染（JSON 响应），则跳过；若有，通过 ViewResolver 定位模板并合并数据输出。
6. 清理：PostHandle 与 AfterCompletion 执行资源释放与日志记录。
对比前端：前端 Router (Vue/React) 是静态配置的路由映射，关注状态树更新；DispatcherServlet 是动态的请求路由与服务端逻辑执行引擎，涉及对象生命周期管理、并发控制和协议解析。

### 3. 基础代码与实战验证
```text
// 模拟 DispatcherServlet 核心 doDispatch 方法的简化源码
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;
    
    // 1. 前置拦截器执行前触发点
    try {
        // 2. 解析 Multipart (文件上传)
        if (isMultipart(request)) {
            // ... multipart resolution ...
        }
        
        // 3. 核心步骤：查找 Handler 和 Interceptor 链
        // HandlerMapping 内部维护了 URL Pattern 到 Controller 方法的映射表
        mappedHandler = getHandler(processedRequest);
        if (mappedHandler == null) {
            noHandlerFound(processedRequest, response);
            return;
        }
        
        // 4. 查找能处理该 Handler 的适配器
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
        
        // 5. 执行拦截器 PreHandle
        if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return; // 拦截器决定终止请求
        }
        
        // 6. 通过适配器反射调用目标 Controller 方法
        // ModelAndView 封装了模型数据和视图名，或直接序列化为 JSON
        mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
        
        // 7. 执行拦截器 PostHandle
        mappedHandler.applyPostHandle(processedRequest, response, mv);
        
    } catch (Exception ex) {
        // 异常处理...
    } finally {
        // 8. 视图渲染与后置清理
        if (mv != null && !mv.wasCleared()) {
            render(mv, processedRequest, response); // 整合数据并生成 HTML/JSON
        }
        mappedHandler.triggerAfterCompletion(processedRequest, response, null); // 资源释放
    }
}
```

### 4. 常见误区与进阶思考
误区一：认为 DispatcherServlet 等同于 Controller。实际上它只负责调度，不持有业务状态，真正的业务逻辑分散在各个被注册的 HandlerBean 中。
误区二：混淆 RequestScope 的作用域。DispatcherServlet 每次请求创建新的请求上下文，Handler 实例默认是单例的（Singleton），但方法参数绑定发生在每次请求栈帧中，需注意共享资源的线程安全问题。
进阶思考：当 @RequestBody 标注的方法接收到一个非标准 Content-Type 的请求时，Spring 是如何通过 HttpMessageConverter 链进行类型转换和异常捕获的？如果转换器抛出异常，DispatcherServlet 如何将其映射为标准的 HTTP Error Response？
