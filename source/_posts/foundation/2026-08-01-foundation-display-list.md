---
title: "每日基础技术总结 · 2026-08-01 · 绘制中的显示列表（Display List）与栅格化"
date: 2026-08-01 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-01 · 绘制中的显示列表（Display List）与栅格化

## 📚 今日主题

> **绘制中的显示列表（Display List）与栅格化**（前端底层与计算机基础）

### 1. 核心概念速览
显示列表（Display List）是渲染引擎内部维护的一张有序、不可变（或半不可变）的绘制指令表，每条指令描述一个原子绘制操作（如填充矩形、绘制文本、裁剪、变换），指令的输入是几何/样式数据，输出是交给光栅化器的最终命令序列。栅格化（Rasterization）是将这些矢量指令转换为像素的过程：通过扫描转换（scan conversion）把几何覆盖的像素区域求交，并对每个像素做着色、深度测试和混合，最终写入帧缓冲。

它解决的问题是立即模式（immediate mode）的两大缺陷：一是指令执行与场景变更强耦合，导致任意微小变更都需要重绘全部内容；二是绘制上下文与线程模型绑定，无法跨帧复用。显示列表将“场景描述”与“像素生成”解耦：场景变更只更新显示列表中的对应条目，未变更的条目直接重放；且显示列表可以在任意线程（合成器线程、栅格线程池）被重放，主线程不必参与像素生成。

在整个计算机图形学体系中，显示列表处于“场景组织”与“像素生成”之间的中间层：上承布局树/场景图，下接光栅化器与帧缓冲。专业工程师必须掌握它，因为它直接决定渲染性能的上限——显示列表的失效粒度决定重绘面积，光栅化代价决定像素填充率。前端性能问题（主线程长任务、图层爆炸、滚动卡顿）的根因都可以归因到这两个环节。

### 2. 底层原理剖析
显示列表的构建与执行是两层分离的流程。

构建阶段：渲染引擎遍历布局树（或场景图），对每个节点按绘制顺序（painting order）生成指令。指令是扁平化的，不保留树结构，只保留几何、变换矩阵、绘制属性和裁剪边界。关键点：指令是“已解决”的，即样式计算、级联、继承、层叠上下文已经在指令生成前完成，指令内部只含可直接执行的数值。

存储与失效：显示列表按图层（layer）分段。每个图层持有自己的显示列表，图层内的指令跨帧复用；当样式/布局变更时，只将受影响图层的显示列表标记为 dirty，并增量更新，未 dirty 的图层直接重放旧指令。

重放与栅格化：显示列表执行器逐条读取指令，调用光栅化器。光栅化的核心是扫描转换：对矩形/路径/三角形，计算其覆盖的像素区间（如通过边缘函数/半平面法），对覆盖像素做覆盖率采样（抗锯齿）、颜色插值、深度测试与混合，写帧缓冲。GPU 光栅化中，显示列表被编码为命令缓冲（command buffer），顶点数据经顶点着色器变换后，由硬件光栅化单元生成片段，片段着色器计算最终颜色。

与前端已有概念的对比：

- DOM 与显示列表：DOM 是语义树，可被 JS 查询和修改；显示列表是渲染引擎的最终绘制指令，不可被 JS 直接访问。DOM 的修改必须经过样式/布局计算后才能映射为显示列表变更。可以把 DOM 理解为“高层的、可变的场景描述”，显示列表是“低层的、已编译的绘制代码”。

- 虚拟 DOM 与显示列表：虚拟 DOM 是协调阶段的纯 JS 数据结构，目的是减少对真实 DOM 的变更次数；它不参与绘制指令生成。虚拟 DOM diff 优化的是“DOM 操作的次数”，而不是“绘制成本”。显示列表优化的是“像素生成的重复率”。

- 类比 Java interface 与 TS interface：Java 的 interface 是运行时存在的（字节码层面有类型信息，可通过反射查询），TS 的 interface 在编译后完全擦除，是编译期契约。同理，DOM 是运行时实体（可查询、可事件监听），显示列表是引擎内部的瞬时产物（从外部不可见），虚拟 DOM 是协调期的临时结构（存在于 JS 堆中，只在提交阶段有意义）。三者的生命周期和可见性完全不同。

### 3. 基础代码与实战验证
```text
// 渲染引擎核心管线：布局 → 显示列表构建 → 光栅化
function renderFrame(layoutTree, viewport) {
  // 1. 构建显示列表：递归遍历布局树，生成绘制指令序列
  const displayList = [];
  buildDisplayList(layoutTree, displayList);
  
  // 2. 栅格化：逐条执行指令，把矢量命令转为像素
  const framebuffer = new Framebuffer(viewport.width, viewport.height);
  rasterize(displayList, framebuffer);
  
  // 3. 提交帧缓冲到合成器（最终上屏）
  compositor.submit(framebuffer);
}

function buildDisplayList(node, list) {
  // 每条指令都是"已编译"的原子操作，不含任何样式计算
  list.push({ op: 'save', bounds: node.bounds });          // 保存当前绘制状态（裁剪区域、变换矩阵）
  list.push({ op: 'clip', rect: node.bounds });            // 设置裁剪矩形，防止子节点溢出
  
  if (node.backgroundColor) {
    // fillRect 指令：只记录矩形坐标与颜色，不做任何绘制
    list.push({
      op: 'fillRect',
      rect: node.bounds,
      color: node.backgroundColor
    });
  }
  
  if (node.isText) {
    // drawText 指令：记录字形索引与位置，字形的轮廓将在栅格化时被扫描转换
    list.push({
      op: 'drawText',
      font: node.font,
      glyphs: node.glyphIndices,
      position: node.textPosition
    });
  }
  
  // 深度优先遍历子节点，子节点的指令追加在父节点指令之后（painting order）
  for (const child of node.children) {
    buildDisplayList(child, list);
  }
  
  list.push({ op: 'restore' });                            // 恢复之前保存的绘制状态
}

function rasterize(displayList, framebuffer) {
  for (const cmd of displayList) {
    switch (cmd.op) {
      case 'fillRect': {
        // 扫描转换核心：矩形在像素网格上的覆盖区域求交
        const x0 = Math.floor(cmd.rect.x);
        const y0 = Math.floor(cmd.rect.y);
        const x1 = Math.ceil(cmd.rect.right);
        const y1 = Math.ceil(cmd.rect.bottom);
        for (let y = y0; y < y1; y++) {
          for (let x = x0; x < x1; x++) {
            // 覆盖测试：判断像素中心点是否落在矩形内（几何 → 像素）
            if (x >= cmd.rect.x && x < cmd.rect.right &&
                y >= cmd.rect.y && y < cmd.rect.bottom) {
              framebuffer.setPixel(x, y, cmd.color);       // 写入颜色值到帧缓冲
            }
          }
        }
        break;
      }
      case 'drawText': {
        // 字形轮廓（矢量路径）经扫描转换填充像素，再按覆盖率计算抗锯齿颜色
        for (const glyph of cmd.glyphs) {
          const contour = glyph.getContour();              // 获取字形轮廓路径
          rasterizePath(contour, cmd.position, framebuffer); // 路径填充 → 像素
        }
        break;
      }
    }
  }
}

这段代码直观地验证了显示列表的本质：指令不含任何计算逻辑，光栅化是独立的纯函数（输入指令，输出像素）。关键洞察：只要显示列表不变，栅格化结果就不变——因此可以跨帧缓存栅格化结果（即位图/纹理），这就是浏览器缓存绘制结果、避免重绘的理论基础。
```

### 4. 常见误区与进阶思考
误区一：把显示列表视为一种“缓存优化手段”。实际它是渲染架构的基石。显示列表的存在让渲染引擎可以在多个线程/进程上并行重放绘制命令（如 Chrome 的栅格化线程池按 tile 分块并行光栅化），并且允许合成器线程独立于主线程完成合成。缓存只是它的附带收益，其本质是“将绘制过程从场景变更中解耦”。

误区二：认为“光栅化只发生在 GPU”。现代浏览器同时存在 CPU（软件光栅化）和 GPU 光栅化。Canvas 2D 的 Skia 后端默认是软件光栅化；Chrome 在低端设备或驱动异常时会回退到软件栅格化；离屏 canvas、某些 filter 效果也会强制走 CPU。GPU 光栅化的瓶颈是纹理上传带宽和 draw call 数量，CPU 光栅化的瓶颈是像素填充率。性能诊断时不能默认假设绘制一定发生在 GPU。

思考题：CSS 中 transform: translate(100px, 0) 触发的动画可以在合成器线程独立完成，不触发主线程的布局与绘制（显示列表不失效）；而修改 top: 100px 会触发完整管线。请从“显示列表的失效粒度”与“栅格化结果的可复用性”两个维度，解释两者性能差异的底层原因。
