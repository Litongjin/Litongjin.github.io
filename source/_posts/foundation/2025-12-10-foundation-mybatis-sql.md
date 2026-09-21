---
title: "每日基础技术总结 · 2025-12-10 · MyBatis：SQL 执行流程与一级/二级缓存"
date: 2025-12-10 20:00:00
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-12-10 · MyBatis：SQL 执行流程与一级/二级缓存

## 📚 今日主题

> **MyBatis：SQL 执行流程与一级/二级缓存**（Java 后端与 Spring 生态）

### 1. 核心概念速览
MyBatis 是一个半自动化的 ORM（对象关系映射）框架，核心本质是 JDBC 的封装与 SQL 语句的动态生成、执行及结果集映射。SQL 执行流程基于责任链模式，通过 Configuration、Executor、StatementHandler、ParameterHandler 和 ResultSetHandler 五个核心组件协作完成从 SQL 解析到 Java 对象转换的全过程。缓存机制旨在减少数据库 I/O 交互：一级缓存（SqlSession 级别）基于本地 HashMap 实现会话内重复查询零网络开销；二级缓存（Mapper Namespace 级别）跨 SqlSession 共享，需序列化支持，解决多会话数据一致性与性能权衡问题。掌握该知识点是理解 Java Web 数据访问层性能瓶颈、事务边界及分布式系统数据一致性的基础，对于后端工程师优化高并发场景下的 DB 连接压力至关重要。

### 2. 底层原理剖析
SQL 执行底层逻辑可抽象为静态资源加载与动态执行流程的解耦。1. 配置阶段：mybatis-config.xml 和 Mapper.xml 被 XmlConfigBuilder 解析为 Configuration 对象，其中映射了 Statement Id 与 BoundSql（包含最终拼接的 SQL 字符串及参数元数据）。2. 执行阶段：SqlSession.getMapper() 获取代理对象，调用方法时触发 JdkDynamicAopProxy 或类似机制，最终指向 Executor.query/update。3. 路由策略：Executor 根据配置决定使用 CachingExecutor（二级缓存拦截器）还是 BaseExecutor。CachingExecutor 先查二级缓存，未命中则委托给具体的 Delegate Executor（如 SimpleExecutor/ReuseExecutor/BatchExecutor）。4. JDBC 交互：BaseExecutor 通过 newStatementHandler() 创建处理器，经 ParameterHandler 填充 PreparedStatement 参数，由 StatementHandler 执行并返回 ResultSet，最后由 ResultSetHandler 依据 TypeHandler 将字节流转换为 Java 对象。前端类比：JS 中模板引擎预编译 Template 类似于 MyBatis 的 XML 解析；Promise/AJAX 请求对应 Network IO；而 TS 接口定义类型契约，Java 接口在 MyBatis 中不仅定义方法签名，还通过 @Select 等注解或 XML 绑定物理 SQL 实现，具有运行时反射绑定的特性。二级缓存在前端无直接对应物，最接近于 Redis/Cookie 等持久化存储与内存 Cache 的分层架构思想。

### 3. 基础代码与实战验证
```text
public void testCacheFlow(SqlSession sqlSession) {
    // 获取 Mapper 代理，此时尚未发送 SQL
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);
    
    // 第一次查询：触发完整 JDBC 流程
    // 1. CachingExecutor 检查二级缓存 (Namespace: UserMapper, Key: selectUser|param)
    // 2. 若未命中，委托 SimpleExecutor
    // 3. ParameterHandler 设置参数
    // 4. StatementHandler 创建 PreparedStatement 并 executeQuery
    // 5. ResultSetHandler 映射结果集
    // 6. 结果写入一级缓存 (SqlSession.localCache)
    User user1 = mapper.selectById(1001);
    
    // 模拟业务逻辑或手动提交以关闭当前事务上下文对一级缓存的影响（视隔离级别而定）
    // 注意：一级缓存随 SqlSession 生命周期存在
    System.out.println("First DB Hit: true");
    
    // 第二次查询同一 ID：直接读取 SqlSession 内部 HashMap
    // 不涉及任何网络 IO 或 JDBC 调用
    User user2 = mapper.selectById(1001);
    System.out.println("Second DB Hit: false");
    
    // 验证一级缓存独立性
    SqlSession session2 = sqlSessionFactory.openSession();
    UserMapper mapper2 = session2.getMapper(UserMapper.class);
    // 二级缓存若开启且配置了 serialize，此处可能命中二级缓存
    // 否则必然再次发起 DB 请求
    User user3 = mapper2.selectById(1001);
    System.out.println("Third DB Hit (New Session): " + (user1.equals(user3) ? "Possible (if L2 enabled)" : "true"));
    
    session2.close(); // 清理资源
}
```

### 4. 常见误区与进阶思考
误区一：混淆一级缓存的范围。一级缓存仅在同一个 SqlSession 实例内有效，不同的 SqlSession 对象即使连接同一数据库事务，彼此的一级缓存也是隔离的。误区二：忽视二级缓存的生命周期管理。二级缓存默认在事务提交（commit）后才生效，若在事务中更新数据但未提交，后续查询仍会读到旧缓存数据，导致脏读。此外，非 POJO 对象或复杂关联映射对象若无正确实现 Serializable 接口且缓存类型为 SERIALIZABLE，将在跨会话取用时报错。
深度思考题：在分布式微服务架构下，MyBatis 的原生二级缓存为何失效？如果要实现集群环境下的数据一致性缓存，你是选择引入外部中间件（如 Redis），还是在应用层实现自定义的 PerpetualCache 委托机制？请从 CAP 理论和缓存穿透/击穿角度分析其优劣。
