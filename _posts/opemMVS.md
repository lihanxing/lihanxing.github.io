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

openMVS的输入是一组图像以及已经计算出的位姿，所以省去了SFM位姿估计的部分。在openMVS的稠密重建中，由以下部分组成：深度图计算、深度图融合、点云颜色计算和点云法线计算