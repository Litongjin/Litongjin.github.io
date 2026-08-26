---
title: "每日基础技术总结 · 2026-08-27 · SQL 基础查询与 JOIN"
date: 2026-08-27 06:55:58
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-27 · SQL 基础查询与 JOIN

## 📚 今日主题

> **SQL 基础查询与 JOIN**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
SQL（Structured Query Language）是关系数据库的标准数据操作语言，其本质是关系代数（Relational Algebra）的文本化表达。基础查询对应 SELECT 语句，由三类核心运算构成：投影（Projection，π，选取列）、选择（Selection，σ，过滤行）、分组聚合（Grouping/Aggregation，γ）。JOIN 是二目关系运算，依据关联谓词（通常为外键等值条件）将两个关系中的行横向组合为新的关系，其形式化定义为 R ⋈_θ S = σ_θ(R × S)：先对两表做笛卡尔积，再施加选择谓词。它解决的核心问题是：在数据引擎内部完成跨表信息重组，避免应用层逐条发起的 N+1 次查询与内存手工关联，将网络往返次数从 O(N) 降为 O(1)。在整个计算机体系中，SQL 位于存储引擎之上、ORM 与应用框架之下，是所有后端数据访问的公共语义层；Node.js 的 Sequelize/Prisma、Java 的 MyBatis/Hibernate、Python 的 SQLAlchemy 最终均编译为 SQL 文本。在 AI 体系中，SQL 是特征工程与训练样本抽取的入口，离线数仓与实时管道的 ETL/ELT 均以 SQL 为核心。专业工程师必须掌握其底层语义，因为 ORM 隐藏了 JOIN 的类型与谓词求值时机，而生产环境的性能劣化（索引失效、笛卡尔积爆炸、LEFT JOIN 退化）全部暴露在 SQL 层。与前端知识体系的本质差异：JS 数组的 filter/map/reduce 是命令式、按书写顺序逐元素执行的；SQL 是声明式、由优化器基于统计信息选择物理执行路径的集合级操作。二者表达力等价，但求值时机与性能模型完全不同。

### 2. 底层原理剖析
一、逻辑执行顺序（标准 SQL 语义，与书写顺序无关）：
1. FROM：确定源关系，多表时构造笛卡尔积
2. ON：按连接谓词过滤笛卡尔积（JOIN 专属阶段）
3. WHERE：对连接后的中间关系做行选择（σ）
4. GROUP BY：按指定列分组
5. HAVING：对分组结果做组级过滤
6. SELECT：投影（π）并计算表达式与别名
7. DISTINCT：消除重复行
8. ORDER BY：排序
9. LIMIT/OFFSET：结果集截断
由此推出两条硬性约束：WHERE 无法引用 SELECT 别名；ON 与 WHERE 的谓词求值阶段不同，直接决定 OUTER JOIN 的语义边界。

二、JOIN 的物理实现（优化器三选一）：
- Nested Loop Join：外层表逐行扫描内层表，复杂度 O(N×M)；内层连接列有索引时为 Index Nested Loop，降为 O(N×log M)。算子级伪代码：for r in R: for s in S: if θ(r,s): emit(r⊕s)
- Hash Join：对较小表构建哈希表，再扫描较大表逐行探测，O(N+M)，仅适用于等值连接
- Merge Join：两表先按连接键排序，再双指针归并，O(NlogN+MlogM)，适用于连接列已有序或有索引的场景
优化器依据表行数、列基数、索引选择性等统计信息选择执行计划——这是声明式语言的本质：调用方描述结果集形态（what），引擎决定物理路径（how）。

三、与前端知识体系的映射：
- SELECT ... FROM a JOIN b ON a.k = b.k 等价于 JS 的 a.flatMap(x => b.filter(y => x.k === y.k).map(y => ({...x, ...y})))；但 JS 固定 O(N×M)，SQL 经 Hash Join 可降为 O(N+M)
- 表是多重集（Bag，允许重复行且无序），数组是有序序列；SQL 中 ORDER BY 是唯一保证输出顺序的手段
- SQL 的 NULL 三值逻辑（TRUE/FALSE/UNKNOWN）比 JS 的 null/undefined 更严格：任何与 NULL 的比较或算术运算返回 UNKNOWN，WHERE 仅保留 TRUE 行；而 JS 中 undefined === undefined 为 true
- 表结构是强制的运行时物理约束（类型/非空/外键），近似 TS 接口的结构匹配，但 TS 接口在编译期即被擦除，表约束在每次写入时强制生效

### 3. 基础代码与实战验证
```text
-- 验证环境：PostgreSQL 15+，标准 SQL 语法在 MySQL/SQLite 中同样成立
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  age INT
);

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id),  -- 外键：建立 users.id 与 orders.user_id 的关联
  amount NUMERIC
);

INSERT INTO users (name, age) VALUES ('Alice', 30), ('Bob', 25), ('Carol', 17);
INSERT INTO orders (user_id, amount) VALUES (1, 100), (1, 200), (2, 50);

-- 1. 基础查询：投影 + 选择 + 排序 + 分页
-- 逻辑执行顺序：FROM users → WHERE age > 18 → SELECT name, age → ORDER BY id DESC → LIMIT 1
-- WHERE 先于 SELECT 求值，故 SELECT 中定义的别名在 WHERE 内不可见
SELECT name, age
FROM users
WHERE age > 18
ORDER BY id DESC
LIMIT 1;

-- 2. INNER JOIN：仅保留两表匹配的行
-- 逻辑等价于 σ_users.id=orders.user_id (users × orders)：先构造笛卡尔积再过滤
-- 物理上：若 orders.user_id 存在索引，走 Index Nested Loop；否则走 Hash Join
SELECT u.name, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- 3. LEFT JOIN：左表全保留，右表无匹配时补 NULL
-- ON 的求值发生在 JOIN 阶段：Carol 无任何订单，其 o.id/o.amount 被置为 NULL，但行仍保留
-- 若把金额条件放入 WHERE，该行会被删除，LEFT JOIN 退化为 INNER JOIN（见误区一）
SELECT u.name, o.id AS order_id, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- 4. LEFT JOIN + 分组聚合：统计每个用户的订单数与总额
-- 执行顺序：FROM → LEFT JOIN（Alice 2 行、Bob 1 行、Carol 1 行，共 4 行）→ GROUP BY u.id → SELECT 聚合
-- COUNT(o.id) 仅统计非 NULL 值，Carol 的订单数为 0；若误用 COUNT(*) 会把 NULL 行计入，结果为 1
SELECT u.name, COUNT(o.id) AS order_count, COALESCE(SUM(o.amount), 0) AS total_amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name
ORDER BY order_count DESC;
```

### 4. 常见误区与进阶思考
误区一：混淆 ON 与 WHERE 的过滤时机，导致 LEFT JOIN 意外退化为 INNER JOIN。
对比两条语句：
A. LEFT JOIN orders o ON u.id = o.user_id AND o.amount > 100
B. LEFT JOIN orders o ON u.id = o.user_id WHERE o.amount > 100
语句 A 中，AND o.amount > 100 属于 ON 谓词，在 JOIN 阶段参与右表匹配过滤；无匹配的左表行仍保留（右列置 NULL），因此无订单的 Carol 仍在结果集中。语句 B 中，o.amount > 100 在 JOIN 完成后于 WHERE 阶段求值；Carol 行 o.amount 为 NULL，NULL > 100 返回 UNKNOWN，而 WHERE 仅保留 TRUE 行，故 Carol 被删除——LEFT JOIN 在语义上退化为 INNER JOIN。根因是谓词求值阶段不同叠加 NULL 三值逻辑，这是生产环境数据缺失类 bug 的高频来源。

误区二：在 WHERE 或 GROUP BY 中引用 SELECT 别名。
SQL 求值顺序为 FROM → JOIN → WHERE → GROUP BY → SELECT，别名在 SELECT 阶段才绑定，故 WHERE 与 GROUP BY 中不可见。这与 JS 代码自上而下顺序执行的直觉相悖；正确做法是：过滤分组结果用 HAVING，或使用子查询/CTE 在外层引用别名。此约束源于 SQL 的声明式求值模型——SELECT 是投影算子，其输出关系在流水线末尾才物化。

思考题：PostgreSQL 优化器在执行 SELECT * FROM users u LEFT JOIN orders o ON u.id = o.user_id WHERE o.amount IS NOT NULL 时，会自动将其改写为 INNER JOIN 执行。请解释该等价变换成立的两个前提：(1) LEFT JOIN 中右表列 NULL 的唯一来源是左表行无匹配；(2) WHERE 谓词是 null-rejecting 的，对右表列为 NULL 的行一律返回 UNKNOWN，过滤后剩余行的右表列必非空，等价于断言每行都存在匹配，LEFT 与 INNER 结果集相同。进一步思考：若将条件改为 WHERE o.amount > 100，该改写是否仍成立？为什么？
