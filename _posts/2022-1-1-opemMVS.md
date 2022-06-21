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

openMVS的输入是一组图像以及已经计算出的位姿，所以省去了SFM位姿估计的部分。在openMVS的稠密重建中，由以下部分组成：**深度图计算、深度图融合、点云颜色计算**和**点云法线计算**组成。其中**点云颜色计算**和**点云法线计算**一般不计算，因为浪费计算资源。**深度图计算、深度图融合**是稠密重建中的关键。

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
openMVS的输入是一系列图像、对应的相机内外参和稀疏点云。因此在稠密重建开始前先准备好重建数据，同时剔除图像数据中无效的、未标定的、被废弃的数据。在数据准备过程做了以下几件事：
* 剔除无效图像
* 对图像的分辨率进行调整，使其满足最大分辨率和最小分辨率的要求
* 根据重新设置的分辨率，对图像的相机参数进行调整

---
#### 领域帧选择
![](/img/openMVS/领域帧选择框架图.png)
领域帧选择阶段就是给每个帧选择对应的领域帧，为后续的匹配做准备，合适的领域帧选择会提升重建效果。在代码中的领域帧选择部分分为两部分：**为每个帧选择潜在的领域帧**，即为每个帧选出所有符合条件的领域帧；以及**为每个帧选择最佳的领域帧**。

----
##### 为每个帧选择潜在的领域帧
选择潜在领域帧的过程，参考的论文是"Multi-View Stereo for Community Photo Collections".
对于参考帧R，我们希望找到对于R而言足够好的领域帧V。具体的选择标准为：共视点f在两个图像V和R之间的夹角为Wn，f在两个图像中分辨率的相似性是Ws，f在两个图像中覆盖面积的最小值为area，因此参考帧R和领域帧V的分数为：
        ![](/img/openMVS/潜在领域帧计算公式.png)
        Wa的定义如:![](/img/openMVS/潜在领域帧计算公式1.png)，其中a为共同特征Vi与Vj与共视点f的角度。该公式的意义是减少小于角度amax所带来的影响，amax设为10。该公式的现实意义是，**如果这个角度越小，说明这两个帧之间的变化就越小，也就越无法从这两个帧中推断出对重建有效的信息，因为基线就越小**。当然，如果这个角度越大，那就更无法推断出有效信息，**因为角度过大时的几乎無共视特征**。

总结一下，领域帧选择的三个条件：
* 角度：两个view靠的越近，越不能提供一个足够大的baseline(基线)去重建高精度的模型，通过共视点夹角间接的判断两个view之间基线是否足够大。
* 分辨率：如果两个图像的分辨率差异过大， 那么肯定会影响立体匹配。所以选择领域帧时，两个帧的分辨率要尽可能一致。
* 共视面积：重建时，共视点的面积也是越大越好。
总之就是，潜在领域帧的选择过程中，收到共视点夹角，领域帧分辨率以及共视区域面积这3个因素影响：共视点f在两个图像(V,R)的夹角(fV与fR组成的夹角)；邻域帧R与当前帧V的分辨率是否接近；共视点在图像中覆盖的面积area。通过以上三个条件对每个候选领域帧打一个分数score，分数越大越合适。

```
PS：其实这一步完了后还有个领域帧滤波，但是这个操作要么不用(因为很容易把所有的领域帧都给过滤掉)，要么设置一个很宽松的阈值。或者直接选取我们领域帧中，前几个分数最大的帧，因为我们最后得到的帧都有分数，那分数越大说明越合适。
```

----
##### 为每个帧选择最佳的领域帧
在上一步我们为每一帧选择了它的所有潜在领域帧，同时这些领域帧与该帧的匹配度从大到小有个socre。
如果我们粗暴的直接选每一帧中，它的潜在领域帧里score最大的帧，这种领域帧选择策略会使每一帧只考虑到自己的局部，而没有考虑到全局。比如说A视图选择了B视图，C也选了B、D也选了B…，，这样的话就很难覆盖整个场景，这种领域帧选择策略不是我们想要的。