---
title: "每日基础技术总结 · 2026-07-08 · Admission Controller 的 Mutating 与 Validating 顺序及动态准入"
date: 2026-07-08 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-08 · Admission Controller 的 Mutating 与 Validating 顺序及动态准入

## 📚 今日主题

> **Admission Controller 的 Mutating 与 Validating 顺序及动态准入**（DevOps 与云原生）

### 1. 核心概念速览
Admission Controller 是 Kubernetes API Server 请求处理链中的一个可插拔阶段，位于认证（Authentication）与授权（Authorization）之后、对象持久化（etcd 写入）之前。其本质是一段在对象被接受或拒绝前执行策略校验或请求修改的代码。它解决的核心问题是：在资源声明式定义的边界上，强制执行集群级策略（如安全上下文、资源配额、网络策略）和进行默认值注入或字段规范化。机制上分为两类：MutatingAdmission（变更准入）用于修改请求对象，ValidatingAdmission（验证准入）用于校验对象合法性。整个机制在 Kubernetes 控制平面中属于 API 请求的‘最后一道闸门’，是集群安全性与一致性的关键保障。专业工程师必须掌握它，因为它是扩展 Kubernetes 原生能力、实施组织级治理、构建平台工程体系的底层基础，尤其在多租户、合规性、成本治理等场景中，Admission Controller 是唯一的标准化拦截点。

### 2. 底层原理剖析
底层运行机制如下：API Server 收到请求后，依次执行：1) 认证（Authentication）确定身份；2) 授权（Authorization）确定权限；3) 进入准入阶段。准入阶段内，先执行所有 MutatingAdmissionController，再执行所有 ValidatingAdmissionController。该顺序是强制性的，原因在于 Validating 必须基于 Mutating 完成后的最终对象进行校验——否则可能出现先验证通过、后变更导致违反策略的情况。内置控制器（如 NamespaceLifecycle、ResourceQuota）与动态 Webhook 均遵循此顺序。动态准入通过 MutatingWebhookConfiguration 和 ValidatingWebhookConfiguration 资源声明，API Server 会将 AdmissionReview 对象（包含请求的原始对象和用户信息）通过 HTTPS 发送到外部 Webhook 服务，Webhook 返回包含允许/拒绝决策以及（对 mutating 而言）可能修改后的对象。若 mutating webhook 修改了对象，API Server 会基于修改后的对象重新进行 schema 校验（openapi validation），然后再执行后续 mutating webhook（若有）及 validating webhook。多个同类型的 webhook 按配置中的顺序（webhooks 数组顺序）依次调用，并非并行。注意：API Server 在调用 webhook 时会根据对象的操作（CREATE/UPDATE/DELETE）以及资源类型匹配规则进行过滤，只有匹配的请求才会触发。与前端已有概念的对比：Admission 阶段类似 Express/Redux 中间件链，但存在关键差异——前端中间件可以自由选择‘先修改后校验’或‘先校验后修改’，而 Kubernetes 强制将修改与校验拆分为两个连续的阶段，且所有修改先于所有校验。这类似于 TypeScript 中‘类型校验’发生在编译期，而 JavaScript 的‘运行时修改’发生在执行期，但这里两者都发生在对象持久化前，且是两遍独立过滤。本质上是将‘规范化’与‘约束’在请求路径上解耦，保证约束逻辑看到的是规范化后的数据。

### 3. 基础代码与实战验证
```text
以下为极简 Node.js HTTP 服务器，模拟一个 MutatingAdmissionWebhook（仅演示机制，真实环境需 HTTPS 与认证）。关键行为：读取 AdmissionReview，修改 Pod 的 labels，返回响应。

const http = require('http');

// 处理 AdmissionReview 请求
function handleAdmissionReview(review) {
  // 仅处理 Pod 资源
  if (review.request.resource.resource !== 'pods') {
    return { allowed: true };
  }

  const obj = review.request.object;
  // 进行 Mutation：向 Pod labels 注入一个字段
  obj.metadata.labels = obj.metadata.labels || {};
  obj.metadata.labels['admitted-by'] = 'my-mutating-webhook';

  // 构造响应：allowed 为 true，且返回修改后的对象
  return {
    allowed: true,
    uid: review.request.uid,
    patchType: 'JSONPatch', // 使用 JSONPatch 或直接返回完整对象（此处演示完整对象返回）
    // 实际中通常返回 patch，这里直接返回完整对象以简化
    // 注意：如果返回 patch，则需设置 patchType 与 patch 字段
    // 此处改为返回完整对象需要 API Server 支持（实际 Kubernetes 要求要么返回 patch，要么返回对象? 根据规范，mutating webhook 可返回 JSONPatch 或完整对象）
    // 为精确演示，我们采用 JSONPatch 方式：
    // 实际实现应返回 patch，但此处用对象返回做简化。
    // 我们稍后修正为 JSONPatch。
    // 但为了保持代码可运行，我们假设 API Server 接受完整对象。实际上，MutatingWebhook 返回的 AdmissionReview 中允许包含 patch 字段，或者直接修改 object 并返回。官方要求：如果返回 patch，则必须设置 patchType。这里我们展示基于 patch 的做法。
  };
}

// 完整实现 JSONPatch 方式：
function buildPatch(original, modified) {
  // 极简 patch 生成：比较两个对象，生成 add 操作
  const patch = [];
  const origLabels = original.metadata.labels || {};
  const modLabels = modified.metadata.labels;
  for (const key of Object.keys(modLabels)) {
    if (origLabels[key] !== modLabels[key]) {
      patch.push({
        op: 'add',
        path: `/metadata/labels/${key}`,
        value: modLabels[key]
      });
    }
  }
  return patch;
}

const server = http.createServer((req, res) => {
  if (req.method === 'POST' && req.url === '/mutate') {
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      const review = JSON.parse(body);
      const original = review.request.object;
      // 深拷贝原始对象（简化用 JSON 方法）
      const modified = JSON.parse(JSON.stringify(original));
      modified.metadata.labels = modified.metadata.labels || {};
      modified.metadata.labels['admitted-by'] = 'my-mutating-webhook';

      const patch = buildPatch(original, modified);
      const admissionResponse = {
        uid: review.request.uid,
        allowed: true,
        patchType: 'JSONPatch',
        patch: Buffer.from(JSON.stringify(patch)).toString('base64') // patch 需要 base64 编码
      };

      const response = {
        apiVersion: review.apiVersion,
        kind: review.kind,
        response: admissionResponse
      };
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify(response));
    });
  } else {
    res.writeHead(404);
    res.end();
  }
});

server.listen(8443, () => console.log('Webhook listening on 8443'));

// 说明：
// 1. API Server 发送 AdmissionReview，其中 request.object 是待处理的原始对象。
// 2. Webhook 对对象进行修改，计算 JSONPatch（比较原始与修改后的差异），并返回给 API Server。
// 3. API Server 应用 patch，得到最终对象，然后继续执行后续 admission controller（包括 validating webhook）。
// 4. 此代码省略 TLS 与配置（ValidatingWebhookConfiguration / MutatingWebhookConfiguration），仅展示核心机制。
```

### 4. 常见误区与进阶思考
误区 1：认为 ValidatingAdmissionWebhook 会在所有 MutatingAdmissionWebhook 之前或之间运行。实际上，Kubernetes 强制将所有 mutating 阶段执行完毕后才进入 validating 阶段。这意味着如果 validating webhook 需要基于某个 mutating webhook 修改后的字段做校验，则只能依赖全局顺序；无法在同一个链中穿插执行。

误区 2：认为 mutating webhook 的修改结果不会被再次验证。实际上，API Server 在应用所有 mutating patch 后，会重新执行 OpenAPI schema 校验（即结构校验），然后才调用 validating webhook。如果 mutating webhook 产生了不符合资源 schema 的修改，请求会失败。此外，validating webhook 看到的对象是最终修改后的对象，所以必须针对该对象设计校验逻辑。

思考题：如果存在两个 mutating webhook，A 和 B，配置顺序为 A、B。A 修改了对象的字段 X，B 基于 X 的值来决定是否修改 Y。那么，如果我们将 A 和 B 的顺序交换，会对最终持久化对象产生什么影响？请从请求处理流程的角度，详细说明顺序对对象最终状态及后续 validating 阶段的影响。该题检验你是否真正理解 mutating 链是串行且后置修改覆盖前置修改，以及 validating 看到的是最终合并后的对象。
