---
title: "每日基础技术总结 · 2024-02-25 · 哈希函数：SHA 族与抗碰撞性"
date: 2024-02-25 20:00:00
categories: [技术分享]
tags: ["技术分享", "安全进阶（Web / 认证授权 / 密码学）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-02-25 · 哈希函数：SHA 族与抗碰撞性

## 📚 今日主题

> **哈希函数：SHA 族与抗碰撞性**（安全进阶（Web / 认证授权 / 密码学））

### 1. 核心概念速览
SHA族（Secure Hash Algorithm）是一类单向哈希函数，本质是将任意长度输入映射为固定长度摘要的确定性算法。核心机制基于Merkle-Damgård结构或Sponge结构，通过非线性压缩函数实现信息混洗。解决的核心问题是数据完整性校验与身份认证链构建，确保在不可逆前提下实现唯一标识。位置：处于密码学底层 primitives 层，是数字签名、区块链共识、证书信任链的基础设施。专业工程师必须掌握，因为它是现代安全协议（TLS 1.3、JWT、Git）的信任锚点，理解其抗碰撞性决定系统能否抵御预映像攻击和选择前文攻击。

### 2. 底层原理剖析
SHA家族演进逻辑：
1. SHA-0/1：基于Merkle-Damgård结构，存在长度扩展攻击缺陷，已被淘汰。
2. SHA-256/512：加固轮函数，增加常数与非线性变换，计算强度高，目前广泛使用但受限于MD结构潜在的理论弱点。
3. SHA-3 (Keccak)：采用Sponge结构，吸收阶段混合输入，挤压阶段提取状态，彻底摒弃MD结构，提供不同的安全性假设。
抗碰撞性（Collision Resistance）：指计算复杂度接近穷举2^(n/2)次运算（生日悖论）。强抗碰撞要求找不到任意两个不同消息产生相同哈希；弱抗碰撞要求给定消息m1，找不到另一个m2使得Hash(m1)=Hash(m2)。
对比前端TS接口：TS接口是编译时静态契约，用于类型检查；SHA哈希是运行时数学约束，用于状态一致性验证。TS防止代码逻辑错误，SHA防止数据篡改和伪造。

### 3. 基础代码与实战验证
```text
#include <openssl/sha.h>
#include <stdio.h>
#include <string.h>

int main() {
    // 待计算字符串
    const char* input = "hello world";
    unsigned char hash[SHA256_DIGEST_LENGTH]; // SHA-256 输出固定 32 字节

    // EVP_MD_CTX: OpenSSL 推荐的抽象上下文，支持灵活算法切换
    EVP_MD_CTX* ctx = EVP_MD_CTX_new();
    EVP_DigestInit_ex(ctx, EVP_sha256(), NULL); // 初始化 SHA-256 上下文
    EVP_DigestUpdate(ctx, input, strlen(input)); // 更新数据，处理块大小不足需填充
    EVP_DigestFinal_ex(ctx, hash, NULL);         // 最终化，执行最后填充并输出摘要
    EVP_MD_CTX_free(ctx);

    // 十六进制打印结果，验证确定性
    for (int i = 0; i < SHA256_DIGEST_LENGTH; i++) {
        printf("%02x", hash[i]);
    }
    printf("\n");
    return 0;
}
```

### 4. 常见误区与进阶思考
误区1：认为 SHA-256 可加密解密。哈希是单向散列，无密钥概念，不存在‘解密’操作，只能暴力破解（字典/彩虹表）而非逆向计算。
误区2：混淆抗碰撞性与预映像攻击。抗碰撞仅保证两不同消息不产生相同哈希，不保护输入隐私（易遭受离线字典攻击），敏感数据必须加盐（Salt）或使用 Argon2/Bcrypt 等慢哈希。
思考题：在 Merkle 树结构中，如果中间某个叶子节点的哈希值被篡改，但父节点哈希保持不变，能否检测到根哈希的不一致？请结合哈希函数的雪崩效应解释为什么这不可能发生。
