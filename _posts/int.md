---
layout: post
title:  "笔试"
# date:   2022-06-21
categories: 三维重建
---
* content
{:toc}

---
## 蔚来

* 如果在Linux中 vim编辑错了,如何不保存直接退出？
> :q -不保存文件，退出 vim
> :q! -不保存文件，强制退出 vim
> :w - 保存文件，不退出 vim
> :w file -将修改另外保存到 file 中，不退出 vim
> :w! -强制保存，不退出 vim
> :wq -保存文件，退出 vim
> :wq! -强制保存文件，退出 vim
> :e! -放弃所有修改，从上次保存文件开始再编辑


* void型没有形参，但是在调用时传实参
> 不行


* c++ 迭代器的使用
> c++中的vector的迭代器定义`vector<int>::iterator it`
> 迭代器与指针类似，可以通过迭代器改变元素本身

---
## 美团

* HDFS
> 1.高容错性：如果有datanode上副本丢失，可以自动恢复。
> 2.构建成本低：可以放在廉价的机器上。
> 3.适合大数据处理：能够处理TB甚至PB级别的数据。同时能够处理百万规模以上的文件数量。同时能够处理10K节点的规模。
> 4.流式数据访问：一次写入，多次读取，不能修改，只能追加；
> 5.适合批处理：它是通过移动计算而不是移动数据；它会把数据位置暴露给计算框架

* 常用的数据转换方法 [参考](https://blog.csdn.net/weixin_39036700/article/details/102587742?spm=1001.2101.3001.6650.2&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-2-102587742-blog-119810588.t5_download_comparev1&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-2-102587742-blog-119810588.t5_download_comparev1&utm_relevant_index=5)
  * 简单变换
  >简单变换常使用函数变换的方式进行，常见的函数变换包括：开方、平方、对数等
  * 数据规范化
  >		
        标准化
		1.离差标准化--消除量纲（单位）影响以及变异大小因素的影响。（最小-最大标准化）
			x1=(x-min)/(max-min)
		2.标准差标准化--消除单位影响以及自身变异影响。（零-均值标准化）
			x1=(x-平均数)/标准差
		3.小数定标规范化--消除单位影响
			x1=x/10**(k)   k=log10(x的绝对值的最大值

    * 离散化
    >		1.等宽离散化 2.等频率离散化 3.一维聚类离散化

    * 属性构造

* attention的作用

---
## 百度

* 640*480的32位图像，文件大小是多少  [参考1](https://blog.csdn.net/a648929081/article/details/80111255?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-80111255-blog-123715837.pc_relevant_multi_platform_whitelistv3&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-80111255-blog-123715837.pc_relevant_multi_platform_whitelistv3&utm_relevant_index=1)  [参考2](https://blog.csdn.net/m0_64620705/article/details/124441474)
> 在计算机中，1B(byte，字节)=8bit(位)，1bit表示一位;  1024B = 1KB
> 假设一个像素有四种颜色值（r，g，b，a）且每种都为256色，那么每个颜色值需要一个字节8bit来存储，即2的8次方256来存储，那么每个像素点占四个字节即4B。
> 那么一张1024*1024的图形所占字节数为`1024*1024*4=4194304B=4096KB=4MB`
> 但是：
以上计算都是基于（r，g，b，a）四种256色（真彩色）的计算，
这里引入位深度的概念：“位”( bit )是计算机存储器里的最小单元，它用来记录每一个像素颜色的值。图像的色彩越丰富，“位”就越多。每一个像素在计算机中所使用的这种位数就是“位深度”。
继而，（r，g，b，a）四种256色需要8*4位，即32位。那么真彩色图片即为32位深度图片。至于8位、16位（通常分为5位红色和5位蓝色，6位绿色（眼睛对于绿色更为敏感））24位深度图像大小，请自行计算。

* 哈希表链地址法 查找成功or不成功的概率 [参考1](https://www.cnblogs.com/gavanwanggw/p/7307596.html) [参考2](https://www.csdn.net/tags/MtjaUg1sNjk4ODktYmxvZwO0O0OO0O0O.html)
> 注意：链地址法新插入的元素要放到链表的前端，这不仅由于方便。同时还由于新插入的元素最可能不久又被访问。
> 查找成功：(查找每个元素需要的次数之后)/元素总数
> 查找不成功：(每个位置需要查找几次)/链表总长度

* 二叉树的重建
> 先说结论：知道一棵二叉树的先序遍历和中序遍历 or  知道二叉树的中序和后续遍历可以确定该二叉树，但是知道先序和后序是无法确定该树的。


---
## 广联达
> 先入为主的说下，广联达的笔试其实比其余几家都简单，但我做的还是拉胯，必须要记录下。

* 函数的单调区间，函数的导数、sin函数和cos函数的导数

* c++友元函数
* c++private
* c++ char型数组
  ```
  //这个在C++中是对的
  int main(){
    char ch[]="abcdef";
    cout<<sizeof(ch)/sizeof(ch[0]);
    }
  ```
  * 小根堆

**编程**


---
## 美的

* AABB和obb
  > AABB效率高 但精度差
  > OBB效率低 但精度高

* 碰撞检测算法
  > 圆  AABB  obb

* 旋转矩阵
  > 旋转矩阵就是正交矩阵
  > 正交矩阵每一列都是单位向量，并且两两正交。最简单的正交矩阵就是单位阵。
  > 正交矩阵的逆（inverse）等于正交矩阵的转置（transpose）
  > 它的行列式为1 且每个列向量都是单位向量且相互正交，它的逆等于它的转置。

**代码**
> 判断n的阶乘，最后结果的末尾有多少个0
* 判断线段是否和直线相交
  > 用向量的叉乘
  