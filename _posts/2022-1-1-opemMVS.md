---
layout: post
title:  "openMVS"
date:   2022-06-17
categories: 三维重建
---
* content
{:toc}

---
openMVS的框架可由：稠密重建、点云融合、网格生成、网格优化和纹理贴图五部分组成。

![三维重建流程](/img/openMVS/openMVS整体框架示意图.png)

---

## 稠密重建

openMVS的输入是一组图像以及已经计算出的位姿，所以省去了SFM位姿估计的部分。在openMVS的稠密重建中，由以下部分组成：**深度图计算、深度图融合、点云颜色计算**和**点云法线计算**组成。其中**点云颜色计算**和**点云法线计算**一般不计算，因为浪费计算资源。

---
### 深度图计算
![](/img/openMVS/深度图计算框架图.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">深度图计算框架图</div></center>

在openMVS中，深度图计算部分属于重中之重，深度图计算的框架图如上图所示，在这个过程中，用到了两个比较经典实用的算法**PlanSweeping**和**PatchMatch**。其中**PlanSweeping**种用的是**SGM**算法。这两个算法是深度图重建过程的关键。

---

#### 数据准备
openMVS的输入是一系列图像、对应的相机内外参和稀疏点云。因此在稠密重建开始前先准备好重建数据，同时剔除图像数据中无效的、未标定的、被废弃的数据；



