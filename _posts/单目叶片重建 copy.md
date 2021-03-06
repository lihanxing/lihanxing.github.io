---
layout: post
title:  "基于单幅图像的植物叶片三维重建"
# date:   2022-06-21
categories: 三维重建
---
* content
{:toc}

---

#### 背景
植物叶片的三维重建在影视、游戏和植物学领域具有重要的应用价值，但传统的植物叶片三维重建方法往往基于三维扫描仪、深度相机等外部设备，这些设备往往**价格昂贵**且**操作不便**，**对于普通用户不友好**。

随着智能设备的普及，单目图像的采集越来越方便。如果能通过单目图像对植物叶片进行高效重建，对于植物学领域以及图形学领域都有一定的应用价值，因此本研究旨在提出一种基于单幅图像的植物叶片高质量三维重建方法。


---
#### 方法
利用单幅图像中的梯度场信息作为线索，通过离散几何处理操作对叶片的三维表面进行 恢复。再根据叶片的几何形态结构推断叶片的深度场信息，并对叶片的宏观结构进行调整使其更逼真， 最终与表面细节进行融合，实现基于单张图像的植物叶片高质量三维重建。

![](/img/项目示意图/叶片重建流程图.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">流程图</div></center>

---
#### 效果图
![](/img/项目示意图/1.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">可视化结果1</div></center>

![](/img/项目示意图/2.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">可视化结果2</div></center>

![](/img/项目示意图/3.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">可视化结果3</div></center>

