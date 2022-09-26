---
layout: post
title:  "图像校正"
date:   2022-06-16
categories: 三维重建
---
* content
{:toc}

---

在openMVS中，图像校正这一步处于SGM算法的第一步。下面将详细介绍下图像校正的原理。

---
### 三角测量
![](/img/openMVS/三角测量示意图.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">三角测量示意图</div></center>

如上图所示，在理想情况下，两个摄像机平面精准的位于同一平面，且行对齐，两个光轴严格平行。
根据三角关系我们可以推导出depth = fT/dis，其中dis就是视差

---
### 对极几何

![](/img/openMVS/对极几何示意图.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">对极几何示意图</div></center>

在真实世界中很难满足上述三角测量中完美的模型，大多数一对图像的关系还是如对极几何示意图这般。
在对极几何约束关系中：
极平面：两相机坐标原点与空间中点p，构成极平面。
极点：左右图像的坐标原点投影到对方像平面上的点
极线：极平面与两个像平面的交线。
极线约束关系：给定图像上的一个特征点，它在另一幅图像上的匹配视图一定在对应的极线上。
有了极线约束，我们就将两幅图像上的特征点匹配从二维搜索优化成了一维搜索，即只需要在极线上搜索就行。

---
### 极线校正
![](/img/openMVS/极线校正示意图.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">极线校正示意图</div></center>

通过上面的讨论，我们已经知道了三角测量可以算出深度、极线约束可以做特征匹配，那么通过极线校正可以将这两者联系起来——即我们可以通过极线校中，从满足极线约束的图像对中恢复对应点在三维空间中的空间信息。
在极线校正过程，将两幅图像在数学上恢复平行关系(三角测量中的那种关系)，然后通过极线约束去寻找匹配的特征点，再利用三角测量计算校正后该点的深度值，再将其转换校正前(即真实情况下的三维空间)的空间位置。

---
### 总结
openMVS中SGM里的极线校正用的就是上述的思想去做，这部分的具体推断可在我的OneNote笔记中看。