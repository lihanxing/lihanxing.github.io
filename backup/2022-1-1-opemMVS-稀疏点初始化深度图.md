---
layout: post
title:  "稀疏点初始化深度图"
date:   2022-06-22
categories: 三维重建
---
* content
{:toc}

---

在openMVS中，做完图像校正后，就要进行深度图初始化了。在这里使用输入的稀疏点对深度图做初始化，由于openMVS的输入数据包含的SFM重建到的稀疏点，所以这里不讨论怎么由图像序列得到稀疏点，只探究如何通过稀疏点初始化深度图。

![](/img/openMVS/稀疏点初始深度图.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">稀疏点初始深度图</div></center>

首先，我们稀疏点投影到对应帧，然后对其三角化，将三角化内平面插值得到完整的深度图，如上图所示。但是这样只得到了三角化区域内的深度值，三角化外区域只能利用已有深度值的最大-最小值范围内随机的去初始化，当然这样会非常不准确。这个过程如下图的左图，右图所示。

![](/img/openMVS/初始化深度图结果.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">深度图结果</div></center>


为了对这些三角化外的区域附上更准确的深度值，我们采用如下的策略：

* 给未三角化区域插入4个角点，这样的话，整张图像都在三角化区域内。
* 为了使新加入的区域的深度值被尽可能合适的初始化，我们找含某一角点的三角面的领域（也就是说领域不包含角点，这是与包含角点的面相邻）3个，然后用这3个领域面的深度信息与初始化角点的深度值。

整张图像初始化的深度图如上图的右图所示。


![](/img/openMVS/稀疏点初始化深度图_角点.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">角点</div></center>

最后说一下这些深度值是怎么初始化到图像中的，可以看到深度图结果的最右侧是一些列稀疏点，这些点其实就是深度值，因此这些稀疏点组成了三角化区域，然后通过这3个点的深度值，线性的对三角区域进行深度插值。角点区域也是一样，通过上面说的方法给4个角点模拟深度值，然后也是用线性插值的方式给新加进来的区域赋予深度值。

然后通过初始化的深度图，利用图像校正部分的知识，将深度图转为视差图。
