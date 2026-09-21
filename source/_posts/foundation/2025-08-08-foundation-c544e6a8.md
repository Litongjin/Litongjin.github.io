---
title: "每日基础技术总结 · 2025-08-08 · 缓存穿透的布隆过滤器与空值缓存"
date: 2025-08-08 20:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-08-08 · 缓存穿透的布隆过滤器与空值缓存

## 📚 今日主题

> **缓存穿透的布隆过滤器与空值缓存**（后端基础）

### 1. 核心概念速览
缓存穿透（Cache Penetration）指查询不存在的数据，导致请求直接击穿缓存层抵达数据库。核心解决方案包含两种机制：1. 布隆过滤器（Bloom Filter）：基于位数组与多哈希函数的概率型数据结构，用于以极小误判率判定键是否“绝对不在”集合中，若存在则可能误报；2. 空值缓存（Null Value Caching）：将查询结果为空的键及其过期时间写入缓存，阻断后续重复请求直达DB。本质是利用空间换时间与概率剪枝，防止DB因无效全表扫描或主键查询而过载。这是高并发系统防御性编程的基石，专业工程师需掌握其内存布局、哈希冲突处理及一致性权衡。

### 2. 底层原理剖析
布隆过滤器运行机制：初始化一个m位的二进制数组（初始全0）和k个独立哈希函数H1...Hk。插入键K时，计算h=Hi(K)得到k个索引，并将数组对应位置置为1。查询键K时，若任一对应位为0，则K必然不存在；若全为1，则K可能存在（存在假阳性False Positive，无假阴性False Negative）。
空值缓存机制：后端逻辑检查缓存，若未命中且DB返回空，则将特定标记（如null或特殊标识）存入Redis，设置较短TTL。后续相同请求仅查缓存即返回。
前端对比：类比TS中的`Set.has(key)`（精确匹配，内存占用大）与JS稀疏数组+哈希（近似匹配，省内存但有碰撞）。布隆过滤器类似分布式环境下的`Set`，但牺牲了准确性换取O(1)时间与固定空间；空值缓存类似将API返回的`undefined`序列化后存入LocalStorage/SessionStorage，但其核心目的是保护下游服务而非提升用户读取速度。

### 3. 基础代码与实战验证
```text
// Java伪代码展示核心逻辑：布隆过滤器接口定义与空值判断
// 假设 BloomFilter<Integer> bf = new BloomFilter<>(expectedInsertions, fpp);

public Object getFromCacheOrDB(String key) {
    // 1. 第一道防线：布隆过滤器快速过滤绝对不存在的键
    if (!bf.mightContain(key)) { 
        // bf.mightContain()内部执行多个哈希运算并检查bit数组
        // 若某一位为0，直接认定不存在，无需任何IO操作
        return null; 
    }

    // 2. 第二道防线：尝试获取业务缓存
    String value = redis.get(key);
    if (value != null) {
        return deserialize(value);
    }

    // 3. 第三道防线：穿透至DB
    Object result = db.query(key);
    
    // 4. 关键逻辑：区分“真不存在”与“数据为空”
    if (result == null) {
        // 空值缓存：存入标记，防止频繁穿透DB
        // 注意：key-value结构通常不支持直接存null，需转为特定字符串或对象
        redis.set(key, NULL_PLACEHOLDER, TTL_SHORT); 
        return null;
    } else {
        // 正常数据：存入业务缓存
        redis.set(key, serialize(result), TTL_LONG);
        return result;
    }
}
```

### 4. 常见误区与进阶思考
误区一：认为布隆过滤器是完美屏障。实际上它存在误判率（False Positive），可能导致合法请求被错误拦截或放行至缓存层仍命中等情况，需结合具体业务场景调整位数组大小与哈希函数数量以平衡准确率与资源消耗。
误区二：混淆空值缓存与永久失效。空值缓存必须设置短期TTL，否则当数据从DB补充或脏数据清除后，应用侧无法感知更新，导致长时间“假死”。
深度思考题：在分布式环境下，如果布隆过滤器本身发生节点故障或数据同步延迟（如Cassandra/Baidu/Caffeine实现差异），如何设计重试或降级策略以避免服务雪崩？请从CAP理论角度分析一致性与时延的取舍。
