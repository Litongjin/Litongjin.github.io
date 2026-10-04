---
title: "每日基础技术总结 · 2026-10-05 · 密码存储：加盐哈希与 bcrypt/argon2"
date: 2026-10-05 07:05:22
categories: [技术分享]
tags: ["技术分享", "安全进阶（Web / 认证授权 / 密码学）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-05 · 密码存储：加盐哈希与 bcrypt/argon2

## 📚 今日主题

> **密码存储：加盐哈希与 bcrypt/argon2**（安全进阶（Web / 认证授权 / 密码学））

### 1. 核心概念速览
密码存储的本质是在不可逆性、抗暴力破解性和计算成本可控性之间建立工程权衡。正确做法绝不保存明文密码，也不保存简单哈希，而是存储『加盐慢哈希』结果。其机制为：用户注册时生成高熵随机盐值，将密码与盐输入密码哈希函数，输出哈希值和参数；登录时取出盐值和参数，重新计算哈希并做恒定时间比较。盐解决相同密码产生相同哈希的问题，防止彩虹表与批量碰撞；慢哈希通过内存硬化或高计算成本提高离线爆破代价。bcrypt 基于 Blowfish 派生密钥，强调 CPU 成本；Argon2 是 Password Hashing Competition 胜出算法，强调内存硬化，可抵抗 GPU/ASIC 并行攻击。该知识点位于认证授权、密码学应用与后端安全边界处，是工程师理解身份系统、会话安全、数据泄露影响面和攻击成本模型的底层基础。

### 2. 底层原理剖析
密码存储要解决三类攻击：数据库泄露后的离线爆破、彩虹表预计算攻击、撞库与批量破解。
1. 哈希函数本身只保证单向性与确定性，不保证安全存储。MD5/SHA-256 速度过快，攻击者可在泄露库上高速穷举。
2. 加盐的本质是打破『同一密码 → 同一哈希』的映射关系。盐必须满足：每用户唯一、密码学安全随机、足够长、可与哈希一起明文存储。
3. 慢哈希的本质是引入可配置工作因子，使攻击成本从 O(1) 提升到 O(cost) 甚至 O(memory-hard)。
4. bcrypt 的核心是 EksBlowfishKeySetup：将密码和盐反复混入 Blowfish 密钥调度，cost 控制指数级迭代次数。它不显式消耗大内存，因此对现代硬件并行攻击的抵抗弱于 Argon2。
5. Argon2 的核心是 memory-hard function：通过填充大块内存并按数据依赖方式访问，使攻击者无法仅靠算力优势加速，因为内存带宽与容量成为瓶颈。Argon2id 融合 Argon2i 的侧信道抵抗与 Argon2d 的 GPU 抵抗，是当前推荐形态。
6. 验证流程必须是：读取存储记录中的算法、参数、盐 → 用相同参数重新计算 → 恒定时间比较。恒定时间比较避免时序侧信道泄露哈希前缀差异。
与前端已有概念对比：TS interface 是编译期类型约束，运行时无实体；而密码哈希算法不是『接口声明』，而是带有具体成本模型的密码学原语。更准确的类比是：bcrypt/Argon2 相当于带运行时资源契约的纯函数，输入相同参数、盐和口令必然输出相同摘要，但计算资源消耗由参数显式决定，且该资源消耗本身就是安全语义的一部分。

### 3. 基础代码与实战验证
```text
// Node.js 极简加盐哈希对比示例：说明为什么不使用快速哈希，以及验证流程的核心结构。
// 实际生产应使用 bcrypt/argon2 库；这里用 scrypt 演示内置 memory-hard 原语，便于理解机制。
import { randomBytes, scryptSync, timingSafeEqual } from "node:crypto";

// 1. 注册：生成密码学安全随机盐，避免相同密码产生相同哈希。
const salt = randomBytes(16); // 128 bit，足以消除预计算彩虹表复用价值。

// 2. 参数化成本：N 是 CPU/内存成本，r/p 控制块大小与并行度。
// 生产环境使用 Argon2id 时应配置 m=19MiB+, t=2+, p=1 起步，并按硬件压测调整。
const derivedKey = scryptSync(password, salt, 64, { N: 16384, r: 8, p: 1 });

// 3. 存储格式：算法参数 + 盐 + 哈希必须一起持久化，否则无法验证。
const record = `scrypt$N=16384,r=8,p=1$${salt.toString("base64")}$${derivedKey.toString("base64")}`;

// 4. 登录验证：从记录还原参数与盐，重新派生密钥，而不是『解密』。
function verify(password, record) {
  const [, params, saltB64, hashB64] = record.split("$");
  const salt = Buffer.from(saltB64, "base64");
  const expected = Buffer.from(hashB64, "base64");
  const actual = scryptSync(password, salt, expected.length, { N: 16384, r: 8, p: 1 });
  // 5. 恒定时间比较：防止因逐字节短路比较泄露时序差异。
  return timingSafeEqual(actual, expected);
}
```

### 4. 常见误区与进阶思考
误区一：把『加盐』当成万能安全层，仍使用 SHA-256(password + salt) 或 MD5。盐只能阻止彩虹表和跨用户复用，不能抵抗高速哈希的离线穷举；必须使用 bcrypt/Argon2/scrypt/PBKDF2 这类慢哈希或内存硬化函数。
误区二：自己实现盐拼接、迭代次数或比较逻辑，例如把盐全局固定、用字符串 === 比较哈希、把参数硬编码且不升级。正确做法是选择成熟库，使用随机每用户盐，恒定时间比较，并将算法、版本、cost、salt、hash 统一序列化存储，以便未来平滑迁移到更强参数或新算法。
思考题：如果数据库泄露，攻击者拥有所有盐和哈希，盐为什么仍然有价值？进一步，为什么在 GPU 攻击模型下，Argon2id 的 memory-hard 特性比单纯提高 bcrypt cost 更具长期防御意义？
