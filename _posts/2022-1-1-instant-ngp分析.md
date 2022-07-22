## instant-ngp
instant-ngp是今年NVIDIA在SIGGRAPH 2022中的项目，由于其"5s训练一个Nerf"的传奇速度，受到研究人员的关注。下面对其做简单介绍，也作为自己学习的记录。

#### 背景
传统基于全连接的神经网络已经能解决很多问题，比如MLP结构(*PointNet、Nerf等*)，但是这种全连接的神经网络在训练和评估中代价很大。同时在基于深度学习的图形学任务中，每个工作都针对自己特定的task，设计不同的网络结构，这样的缺点是这些方法只限制在特定的任务上，同时这些工作在优化整个网络的过程中，需要对整个网络进行优化，这加大了网络的花费。

### 方案
作者提出一种基于多分辨率的哈希编码方案，这个方案与task无关，是一种通用的方案(改方案能用在多种不同任务中)，改方案只由参数*T*和期望的分辨率*N*<sub>mak</sub>配置，实验结果表明该方案在多项任务上达到了不错的效果。该方案与任务无关的**自适应性**和**效率**的关键是哈希表的多分辨率层次结构。
### Adaptivity
作者将网格级联的映射到相应的固定大小的特征向量数组。在粗分辨率下，从网格点到数组条目有一个1：1的映射。在精细分辨率下，数组被视为一个哈希表，并使用空间哈希函数进行索引，其中多个网格点为每个数组条目起别名。
这种哈希碰撞将导致训练梯度达到平均水平，这意味着最大的梯度(与损失函数最相关的梯度)将占主导地位，因此哈希表将自动对最重要且细节最丰富的部分做优先考虑。同时与之前的工作不同，在instant-ngp的训练期间不需要对数据结构进行系统性更新。
### Efficiency
哈希表的查找是O(1),同时哈希表可以并行的查找。

### MultiResolution Hash Encoding
![](/img\instant-ngp\Hash_encoding_parameters.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;"> Hash_encoding_parameters table</div></center>


![](/img\instant-ngp\multiresolution_hash_encoding_示意图.png)
<center>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;"> Illustration of the multiresolution hash encoding in 2D</div></center>

一个全连接网络 *m*(y;$\phi$)，对输入的y进行编码*y*=enc(x;$\theta$)，因此在instant-ngp中，既有可训练的权值参数$\phi$，也有可训练的编码参数$\theta$。这些参数被分布为L个层，每层中包含T个维度为F的特征向量。上面的表1为哈希编码参数，**只有哈希表的大小*T*和期望的分辨率*N*<sub>mak</sub>需要被设置(作为超参数 )**。

在这个过程中，每个层是独立的(*如上图中的红蓝两层，分别代表不同层级。红色代表分辨率较高的体素网格、蓝色代表分辨率较低的体素网格*)，同时存储网格顶点所代表的特征向量，每一层的分辨率是最糙分辨率与最佳分辨率之间的几何级数。[*N*<sub>min</sub>,*N*<sub>mak</sub>]。*N*<sub>mak</sub>代表了数据中的最好细节。

由上面的陈述我们知道参数被分为L个层，按照表1给出的超参数可知L被设置为16，即体素的分辨率有16个等级，那么分辨率层级从低到高的变化公式为：

![](/img/instant-ngp/层级分辨率变化公式.png)

其中b被认为是成长参数，作者将其设置为1.38到2之间。对于L层中的某个层l，输入的坐标点对应的分辨率应该为$\lfloor $x<sub>𝑙</sub>$ \rfloor$:=$\lfloor x* $ *N*<sub>𝑙</sub> $ \rfloor$,$\rceil $x<sub>𝑙</sub>$\rceil$:=$\lceil x* $ *N*<sub>𝑙</sub> $ \rceil$。

对于较粗糙的网格，不需要T个参数，它的参数量应该为(*N*<sub>𝑙</sub>)<sup>d</sup>&le;T，这样以来映射就可以保证一对一的关系，对于精细网格，使用空间哈希函数*h*:*Z*<sup>d</sup>&rarr;*Z*<sub>T</sub>将网格索引到数组。本文中空间哈希函数的定义为

![](/img/instant-ngp/空间哈希函数.png)

运算符$\bigoplus$表示异或运算，$\pi$<sub>i</sub>表示独一无二大小的大素数，改计算在每一维产生线性同余排列，以解除维度对哈希值的影响。最后根据x在体素中的相对位置，实现体素内角点特征向量的线性插值，插值的权重为*W*<sub>𝑙</sub>:=*x*<sub>𝑙</sub>-$\lfloor $x<sub>𝑙</sub>$ \rfloor$(我猜这里应该代表可视化图中，线性插值的部分，但还是有些不理解)。

根据以上对哈希编码的陈述，可以理解这个过程在L层中可以独立的执行，不会相互干扰。这样的话每一层的插值向量以及辅助输入(位置+视角方向)被串联以产生y，y是由输入的enc(x;𝜃)到MLP m(y;Φ)。 

### 神经网络训练参数
为了优化GPU的缓存，作者逐级的处理对应的哈希表，当处理一批输入时，首先计算多分辨率哈希编码的第一级，接着是第二级，依次类推。所以在整个过程中，只有少量的哈希表需要驻留在GPU缓存中，节省了GPU的内存。
