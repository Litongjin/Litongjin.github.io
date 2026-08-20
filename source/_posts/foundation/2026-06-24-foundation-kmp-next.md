---
title: "每日基础技术总结 · 2026-06-24 · KMP 的 next 数组与失配优化"
date: 2026-06-24 08:00:00
categories: [技术分享]
tags: ["技术分享", "算法与数据结构"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-24 · KMP 的 next 数组与失配优化

## 📚 今日主题

> **KMP 的 next 数组与失配优化**（算法与数据结构）

### 1. 核心概念速览
KMP算法是单模式字符串匹配算法，核心在于利用模式串自身的前缀信息构造next数组（前缀函数），在失配时通过next数组将模式串指针回退到合适位置，而主串指针不回溯，从而将匹配复杂度从O(n*m)降到O(n+m)。next数组本质是失配时的状态转移函数，记录了模式串每个位置失配后应跳转到的下标。它解决了朴素匹配中重复比较已匹配前缀的问题。在计算机体系中，KMP是字符串匹配和自动机理论的基石，也是AC自动机、后缀数组等高级算法的基础。专业工程师必须掌握它，因为它是理解状态机、动态规划、线性时间算法的典型范例，并且在实际工程（如文本编辑器的查找、网络入侵检测、DNA序列比对）中有广泛应用。

### 2. 底层原理剖析
前缀函数pi[i]定义：对模式串P，pi[i]为P[0..i]的最长真前后缀长度（即最长的k<i+1使P[0..k-1]==P[i-k+1..i]）。next数组常用定义为next[i]=pi[i-1]（或next[i]=pi[i]，取决于实现），表示当模式串第i个字符失配时，模式串指针应回退到的位置。构造过程本质上是用模式串自己匹配自己：维护当前已匹配前缀长度j，从i=1开始扫描P，若P[i]==P[j]则j++，否则j回退到next[j]，最终pi[i]=j。失配优化：在计算next时，若P[i]==P[next[i]]，说明跳转后仍会失配，需要继续沿着next链向前跳，得到优化后的nextval数组。这样在匹配重复字符较多的模式串时，避免一次失配后紧接着再次失配，减少比较次数。优化后的nextval构造伪代码：
  next[0] = -1; i=0; j=-1;
  while i < m-1:
    if j==-1 or P[i]==P[j]:
      i++; j++;
      if P[i]!=P[j]: next[i]=j; else next[i]=next[j];
    else:
      j=next[j];
匹配时，主串指针i始终递增，失配时j=next[j]（若j==-1则i++,j=0）。
与前端已有概念的对比：KMP的next数组类似前端中正则表达式引擎在匹配前将模式编译为内部状态机（如JS的RegExp对象），利用预编译结果避免每次匹配都从头解释。相同点是两者都通过预处理构建转移表，使匹配过程直接查询转移；不同点是正则引擎支持通配符、量词等模式，编译成NFA/DFA，可能涉及回溯（如JS正则的灾难性回溯），而KMP专门针对字面量字符串，是确定性线性匹配，不回溯主串。另外，KMP的前缀函数本质上是一个前缀自动机，与前端状态管理（如Redux reducer）不同，reducer是纯函数式状态转换，而KMP的转移表是静态预计算的。

### 3. 基础代码与实战验证
```text
以下是JavaScript实现的KMP匹配与优化next数组构造，无框架依赖：

function buildNextOptimized(pattern) {
  const m = pattern.length;
  const next = new Array(m);
  next[0] = -1; // 哨兵：若第0个字符失配，主串指针后移，模式串从头开始
  let i = 0;    // 主索引，指向模式串当前待计算位置的前一个位置
  let j = -1;   // 当前已匹配的前缀长度（跳转位置）
  while (i < m - 1) {
    if (j === -1 || pattern[i] === pattern[j]) {
      i++;
      j++;
      // 若pattern[i]与pattern[j]相同，则跳转后必然再次失配，所以继续回退到next[j]
      if (pattern[i] !== pattern[j]) {
        next[i] = j;
      } else {
        next[i] = next[j];
      }
    } else {
      j = next[j]; // 回退到更短的前缀，继续比较
    }
  }
  return next;
}

function kmpSearch(text, pattern) {
  if (pattern.length === 0) return 0;
  const next = buildNextOptimized(pattern);
  let i = 0; // 主串指针，只增不减
  let j = 0; // 模式串指针
  while (i < text.length) {
    if (j === -1 || text[i] === pattern[j]) {
      i++;
      j++;
      if (j === pattern.length) {
        return i - j; // 匹配成功，返回起始下标
      }
    } else {
      j = next[j]; // 失配时利用next跳转，主串i不变
    }
  }
  return -1; // 未找到
}

// 验证：console.log(kmpSearch('ABABABABC', 'ABABABC')); // 输出2
注释已说明关键机制。构造next时，i与j的移动实际是模式串的自匹配，next[j]保存了更短前缀的信息。
```

### 4. 常见误区与进阶思考
误区1：混淆next数组的定义。有的教材定义next[i]为“前i个字符的最长相同前后缀长度”，有的定义为“失配时跳转的下标”，且可能从0或-1开始。如果实现时不注意偏移，会导致边界错误或死循环。必须明确：本代码中next[i]表示当模式串下标i处失配时，j应回退到的下标；next[0]=-1是特殊哨兵，代表主串指针前进。
误区2：认为优化后的next数组会改变KMP的渐近复杂度。实际上，优化不影响O(n+m)的复杂度，但能减少在重复字符模式下不必要的比较次数。不要因为优化而忽略对前缀函数本质的理解。
思考题：对于模式串'AAAA'，手动构造优化后的next数组，并模拟在文本串'AAAAB'中匹配时j的跳转序列。解释为什么优化后j直接从3跳到-1，而不是依次经过2、1、0。这验证了优化如何通过链式回退跳过完全相同的字符。
