---
title: "每日基础技术总结 · 2026-07-24 · Git 的 merge-base 与三路合并及冲突标记"
date: 2026-07-24 08:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-24 · Git 的 merge-base 与三路合并及冲突标记

## 📚 今日主题

> **Git 的 merge-base 与三路合并及冲突标记**（架构与设计）

### 1. 核心概念速览
merge-base 是给定两个提交（通常是两个分支的末端）在提交图中计算出的最近公共祖先（LCA）。三路合并以该 merge-base 作为基准（base），分别计算两个目标版本（ours 与 theirs）相对 base 的增量，再尝试自动合并这些增量；若两边增量在相同位置发生冲突，Git 将冲突区域用 <<<<<<<、=======、>>>>>>> 标记写入工作区文件。该机制是 Git 合并、rebase、cherry-pick 的核心算法，属于版本控制系统的底层基础。专业工程师必须掌握它，因为冲突解决不是随机修改，而是理解“谁基于什么版本做了什么改动”的推理过程。这与前端中 Object.assign 之类的二路合并有本质区别：二路合并没有 base，无法区分“保留”与“修改”；三路合并通过 base 将两侧改动解析为独立增量，从而能自动合并绝大多数不冲突的改动。

### 2. 底层原理剖析
三路合并的底层过程可形式化为：
1. 在提交图中定位 merge-base：从两个目标提交沿父指针向上做双向广度优先遍历，第一个共同到达的提交即 LCA；若存在多个 LCA（合并历史复杂时），Git 的 recursive 策略会先对它们做虚拟合并生成一个虚拟提交作为 base。
2. 对仓库中的每个文件，读取 base、ours、theirs 三个版本。
3. 分别计算 base→ours 与 base→theirs 的差异（diff），通常以行为单位生成 hunk。
4. 将两个 hunk 集合应用到 base 上，规则为：某区域只有一侧有改动，则采用该侧改动；两侧均无改动，则保留 base；两侧都有改动且改动内容一致，则保留；两侧都有改动且内容不同，则判定冲突。
5. 冲突时，工作区文件写入冲突标记，索引记录该文件为未合并状态（stage 1/2/3）；非冲突部分自动合并并暂存。
该过程与前端中 React 的 reconciler 的“三个孩子节点比较”有异曲同工之妙，但 React 是树级别的双向比较，Git 是文件行级别的三路增量合并。更精确的对比是 TypeScript 的接口合并：多个声明合并为同一接口，但没有 base 冲突检测，因此后者不能用于版本语义合并。

### 3. 基础代码与实战验证
```text
以下为验证三路合并的极简脚本（假设 Bash 环境）：
# 初始化仓库
git init demo
cd demo
git config user.email 'test@test.com'
git config user.name Test
# 创建 base 提交：文件包含三行
echo 'line1' > file.txt
echo 'line2' >> file.txt
echo 'line3' >> file.txt
git add file.txt
git commit -m 'base commit'
# 创建 feature 分支，修改 line2
git checkout -b feature
sed -i 's/line2/feature-line2/' file.txt
git commit -am 'modify line2 in feature'
# 回到 master，对同一行做不同修改
git checkout master
sed -i 's/line2/master-line2/' file.txt
git commit -am 'modify line2 in master'
# 合并：两边均修改同一行，必然冲突
git merge feature
# 查看冲突标记
cat file.txt
输出：
line1
<<<<<<< HEAD
master-line2
=======
feature-line2
>>>>>>> feature
line3
其中 <<<<<<< HEAD 之后是当前分支（ours）的版本，======= 之后是待合并分支（theirs）的版本，>>>>>>> feature 表示冲突块结束。由于只有 line2 被双方修改，line1 和 line3 保留 base 原样。索引中该文件处于 Unmerged 状态（stage 1 为 base，stage 2 为 ours，stage 3 为 theirs），可通过 git ls-files -u 查看。
```

### 4. 常见误区与进阶思考
常见误区：
1. 认为冲突标记是 Git 合并失败时留下的错误信息，需要手动删除。实际上冲突标记是工作区文件中表示冲突区域的文本，是合并算法的输出；解决冲突时，需要编辑文件，保留想要的代码，并删除标记行，然后 git add 标记为已解决。
2. 认为三路合并只需要比较两个分支的当前状态，不需要 base。这是根本性误解。没有 base，就无法判断某行是“新增”还是“删除”，也无法区分“修改”和“保留”。例如，如果仅比较 ours 和 theirs，一行只在 theirs 中出现，无法知道是 theirs 新增的还是 base 中原本就有而 ours 删除了。只有 base 才能提供语义锚点。
思考题：如果两个分支相对于 base 都新增了同一个文件，但内容完全不同，三路合并会产生冲突吗？为什么？提示：想想 Git 对新增文件的处理是否也依赖 base 的缺失状态。
