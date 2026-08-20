---
title: "每日基础技术总结 · 2026-07-10 · GitLab CI 的 cache 与 artifact 的区别与作用域传递"
date: 2026-07-10 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-10 · GitLab CI 的 cache 与 artifact 的区别与作用域传递

## 📚 今日主题

> **GitLab CI 的 cache 与 artifact 的区别与作用域传递**（DevOps 与云原生）

### 1. 核心概念速览
GitLab CI 的 cache 与 artifact 是两种完全不同的文件复用机制。cache 是跨 pipeline 的输入依赖缓存，用于复用不会频繁变更的依赖（如 node_modules），以 key 为标识，在 job 执行前恢复、执行后保存，存储于 Runner 本地或远程缓存，存在被清理的可能，不保证每次命中，且仅在相同项目、相同 Runner（或共享缓存配置）间可见。artifact 是 pipeline 内的输出产物传递机制，用于将一个 job 产生的文件（如构建结果）上传至 GitLab 服务器，供同一 pipeline 中的后续 job 下载，或供人工下载，具有明确的过期时间，与 pipeline 强绑定。本质区别：cache 解决的是『重复获取相同依赖』的时间优化，artifact 解决的是『上游 job 输出如何传递给下游』的数据流问题。在整个 CI/CD 体系中，cache 位于执行环境层，artifact 位于流水线编排层。专业工程师必须区分二者，否则会设计出错误缓存策略、导致 job 间依赖失效或流水线不可复现。

### 2. 底层原理剖析
底层机制：Runner 执行 job 时，会先根据配置的 cache:key 在缓存存储中查找匹配的归档文件，若命中则解压到当前工作目录；执行 script；结束后将 cache:paths 指定的路径压缩并保存到缓存存储，key 更新为当前配置。缓存命中条件基于 key 的精确匹配，key 一般由分支名、环境名或文件哈希组成。cache 是尽力而为的，GitLab Runner 可能会清理过期或不常用的缓存；在分布式 Runner 环境下，若不配置 S3 等共享缓存，每个 Runner 只有本地缓存，不同 Runner 间不可见。artifact 流程：job 执行成功后，Runner 根据 artifacts:paths 收集文件，打包上传至 GitLab 服务器，GitLab 将其作为该 pipeline 的一个 job 的产物注册。后续 job 执行前，Runner 会向 GitLab 请求下载当前 pipeline 中满足 dependencies 条件的 job 的 artifact（默认所有之前成功的 artifact），解压到工作目录。dependencies 限定的作用域是同一 pipeline 内的 job，不可跨 pipeline。对比前端概念：cache 类似于 webpack 的持久化缓存（cache: filesystem），以 contenthash 为 key，跨构建复用，但可能因缓存失效或清理而重新编译；artifact 类似于 webpack 构建输出的 dist 目录，是构建的结果，可以被部署或传递给后续流程。更细粒度看，cache 是『中间态的可复用输入』，artifact 是『有界流的不可变输出』。

### 3. 基础代码与实战验证
```text
# 极简 .gitlab-ci.yml 验证 cache 与 artifact 的区别
stages: [build, test]

# 全局 cache 配置：作用于所有 job，按分支名隔离，跨 pipeline 复用
cache:
  key: "node-cache-${CI_COMMIT_REF_SLUG}"
  paths:
    - node_modules/   # 缓存 node_modules 目录

build-job:
  stage: build
  script:
    - npm install              # 若 cache 命中，此命令极快；否则重新下载
    - mkdir -p dist
    - echo "build ok" > dist/app.txt
  artifacts:
    paths:
      - dist/                  # 将 dist 作为 artifact 上传到 GitLab
    expire_in: 1 week          # artifact 保留 1 周

test-job:
  stage: test
  dependencies:
    - build-job               # 只下载 build-job 的 artifact，不下载其他 job 的
  script:
    - ls dist/app.txt         # 验证 artifact 从上游 job 传递过来
    - npm test                # 使用 node_modules，此目录由 cache 在 job 开始前恢复

# 执行流程说明：
# 1. build-job 开始前，Runner 尝试恢复 key 为 node-cache-xxx 的缓存到工作区；
# 2. build-job 结束前，Runner 将 dist/ 上传为 artifact；同时将 node_modules/ 保存为缓存；
# 3. test-job 开始前，Runner 先下载 build-job 的 artifact 并解压，再恢复同 key 的缓存（node_modules）；
# 4. test-job 中的 ls 能看到 dist/app.txt，证明 artifact 传递；npm test 能快速依赖安装后的 node_modules，证明 cache 复用。
# 注意：cache 在 test-job 结束后也会保存（因为全局配置），但 key 相同，属于冗余操作，实际中可用 cache: {} 覆盖禁用。
```

### 4. 常见误区与进阶思考
常见误区 1：把 cache 当作 job 间传递文件的手段。cache 虽然也能在 job 间共享，但它不是保证性的，可能被清理，且不同 Runner 间默认不共享；若 job 依赖上游产物，必须用 artifact。常见误区 2：混淆 dependencies 与 needs。dependencies 控制 artifact 下载列表，只影响文件传递；needs 控制 DAG 调度顺序，影响 job 何时开始，两者独立。进阶思考：如果同一项目有两个 pipeline 同时运行，且使用相同 cache key，底层会发生什么？极端情况下两个 Runner 同时写同一缓存会导致缓存归档损坏，后续 job 命中后解压失败。那么如何设计缓存策略使其在并发场景下安全？考虑将 key 中加入 CI_JOB_ID 或使用原子上传机制，或者依赖锁。这道题能检验你是否真正理解 cache 的生命周期和并发模型。
