---
title: "每日基础技术总结 · 2024-05-28 · MyBatis：#{} 与 ${} 区别及 SQL 注入防护"
date: 2024-05-28 20:00:00
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-05-28 · MyBatis：#{} 与 ${} 区别及 SQL 注入防护

## 📚 今日主题

> **MyBatis：#{} 与 ${} 区别及 SQL 注入防护**（Java 后端与 Spring 生态）

### 1. 核心概念速览
MyBatis 中的 #{ } 与 ${ } 是两种截然不同的参数预处理机制，核心区别在于是否经过 JdbcPreparedstatement 的参数绑定流程。#{ } 本质是 PreparedStatement 占位符（?）的语法糖，通过 JDBC 驱动层的类型转换与二进制传输数据，从底层阻断 SQL 注入；${ } 本质是字符串模板替换，在 SQL 解析阶段直接将变量值拼接到 SQL 语句文本中，无任何转义保护，极易引发 SQL 注入。掌握此区别是构建安全后端基础设施的前提，因为数据库交互是数据持久化的最后一道防线，任何逻辑层的安全校验都必须建立在传输层或存储层的安全协议之上。

### 2. 底层原理剖析
1. #{ } 机制：MyBatis 将 #{} 解析为 JDBC 的 '?' 占位符，并调用 PreparedStatement.setXXX() 方法。该过程由数据库驱动（如 MySQL Connector/J）处理，利用客户端-服务端通信协议的 binary 格式传输参数值。数据库引擎在准备执行计划（Prepare Plan）时，参数部分被视为纯数据而非可执行代码，因此无论输入内容如何，均无法改变原有 SQL 的逻辑结构。
2. ${ } 机制：MyBatis 在解析 XML/注解配置时，直接对 String 进行正则匹配与替换。生成的最终 SQL 字符串随后被传递给 Connection.createStatement().executeQuery(sql)。此时，数据库接收到的是一段完整的、未经预编译的代码文本，若变量包含恶意 SQL 片段（如 ' OR 1=1; --），则会被解析为合法逻辑。
3. 前端类比：这与 TypeScript 编译期类型检查 vs JavaScript 运行期 eval() 的区别类似。#{ } 如同严格的静态类型约束，确保数据结构合规后才进入执行流；${ } 则如同动态拼接 HTML 字符串，缺乏上下文感知，直接将不可信数据混入指令集。

### 3. 基础代码与实战验证
```text
// MyBatis Mapper 接口定义
public interface UserMapper {
    // 使用 #{ }：生成 PreparedStatement，参数设为 ?
    // SQL 实际发送给 DB: SELECT * FROM users WHERE name = ?
    User findByNameSafe(@Param("name") String name);

    // 使用 ${ }：直接字符串替换
    // 若 name="admin' OR '1'='1"，最终 SQL 变为:
    // SELECT * FROM users WHERE name = admin' OR '1'='1'
    // 导致全表泄漏，存在高危 SQL 注入风险
    List<User> findDynamicTable(@Param("tableName") String tableName);
}

// 对应的 XML 映射文件
<mapper namespace="com.example.UserMapper">
    <select id="findByNameSafe" resultType="User">
        <!-- 底层机制：JDBC PreparedStatement 预编译 -->
        SELECT * FROM users WHERE username = #{name}
    </select>

    <select id="findDynamicTable" resultType="User">
        <!-- 底层机制：String Concatenation -->
        SELECT * FROM ${tableName}
    </select>
</mapper>
```

### 4. 常见误区与进阶思考
误区一：认为在业务层做了严格的白名单校验或 XSS 过滤后，使用 ${ } 就是安全的。事实是：SQL 注入发生在解析层，一旦 ${ } 参与 SQL 构成，即失去防御能力；且业务层校验极易遗漏特殊字符组合，防御纵深应前置到持久层框架的标准用法上。
误区二：混淆 ${ } 用于排序字段（ORDER BY）与注入攻击。虽然 ${ } 常用于动态表名或列名（因 PreparedStatement 不支持标识符占位），但这属于‘受控的动态性’，必须配合应用端的严格枚举校验（Enum White-list），绝不能直接信任用户输入。
思考题：在分布式架构下，如果我们将 SQL 构建委托给一个独立的 SQL 模板引擎服务，而该服务仅支持 ${ } 风格的字符串替换，如何从系统架构层面设计机制，以确保下游数据库实例仍能有效抵抗通过该中间件发起的 SQL 注入？
