---
layout: post
title:  "基于单幅图像的物体稠密点云重建"
# date:   2022-06-21
categories: 三维重建
---
* content
{:toc}

---

#### 背景
生活中，我们不仅能仍单视角图像预测目标三维形状，而且还能捕捉它细节信息，但对计算机来说，单视角图像重建目标的高分辨率三维结构形状是一项较有意义且极具挑战的工作。

随着自动驾驶以及SLAM的发展，如果机器人能通过单目相机对周围环境中物体的三维结构进行有效的估计，这不仅在**成本上更有优势**，同时也能够**辅助机器人对路径规划、避障等算法进行优化**，使机器人**更好的感知周围的三维空间**。


---
#### 方法

为了重建出高分辨率的稠密点云，我们提出一种多阶段的稠密点云重建网络，算法需要先重建出稀疏点云，然后在此基础上重建出稠密 3D 点云。整体流程图如下所示：
![](/img/项目示意图/稠密点云重建流程图.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">总体流程图</div></center>



基于图像的点云生成网络由图像编码器和点云解码器组成。图像编码网络将输入的单张RGB图像映射到嵌入式空间，获得潜在的2D隐式形状特征。再通过点云解码器生成对应特征的稀疏点云。同时为了充分利用单张图像中的有限信息，通过构建注意力机制的特质提取模块，对2D图像中的通道维度与空间维度的语义特征进行提取。基于图像的三维点云重建过程如下所示：
![](/img/项目示意图/RGB24096.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">3D点云重建网络</div></center>


为了更真实的表达物体的三维形态结构，对重建出的三维点云进行稠密化处理，我们构建了基于编码器结构的稠密点云生成网络使得稀疏点云稠密化。点云稠密化过程如下图所示：
![](/img/项目示意图/4096216384.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">稠密点云生成网络</div></center>

---
#### 效果图
![](/img/项目示意图/单目点云可视化图.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">可视化结果</div></center>

----
#### 对比结果
下图为本文方法与主流方法在公共数据集上的可视化对比结果：
![](/img/项目示意图/单目点云对比.png)

同时我们与主流方法做定量比较，采用Chamfer distance(CD)(和Earth move’s distance(EMD)作为评估指标，定量指标值越小表示重建结果的误差越小，即重建结果越准确。
![](/img/项目示意图/单目点云定量对比.png)
**通过可视化结果与定量结果可以看出，我们的方法较准确的从单张图像中重建出对应的稠密点云。**