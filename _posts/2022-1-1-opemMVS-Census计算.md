---
layout: post
title:  "代价计算&代价聚合"
date:   2022-06-16
categories: 三维重建
---
* content
{:toc}

---
## 代价计算

Census计算是openMVS中，立体匹配过程做代价计算的策略，或者说是SGM方法做代价计算的策略。原始SGM方法中代价计算是用的(Mutual information)互信息，但是互信息的原理比较复杂且效率较低，所以在实际应用中代价计算一般用Census变换比较多。

![Census变换](/img/2022-6-16/Census变换示意图.png)
从图中我们可以看到，Census变化过程如下：假设左右图都是灰度后的图，我们在左图选择一个3x3大小的窗口，然后以中心点的像素值为基准，周围像素值比起基准大就记为0，比基准小就记为1，所以我们可以得到一个8位的比特串；同理在右图的匹配点待选择区域中选择一个3x3大小的窗口，获得对应的比特串；计算两个比特串的汉明距离，将汉明距离作为代价计算结果。



```
Census计算代码：
// 视差的唯一性约束，一致性检查：主要是判断左右的视差是否相同，剔除错误的视差
// 通过左的视差图，找到每个像素在右影像的同名点像素及该像素对应的视差值，这两个视差值之间的差值若小于一定阈值（一般为1个像素），则满足唯一性约束被保留，反之则不满足唯一性约束而被剔除
void SemiGlobalMatcher::ConsistencyCrossCheck(DisparityMap& l2r, const DisparityMap& r2l, Disparity thCross)
{
	/*
	其中thCross是一个阈值，通过将左右视差加起来，看是否超过该阈值，以此判断左右视差是否一致
	代码中对图中的视差是引用，所以如果视差有问题的话，直接把改点处的视差就改了
	*/

	ASSERT(thCross >= 0);
	ASSERT(!l2r.empty() && !r2l.empty());
	ASSERT(l2r.height() == r2l.height());

	auto pixel = [&](int, int r, int c) 
	{
		Disparity& ld = l2r(r,c);
		if (ld == NO_DISP)
			return;
		// compute the corresponding disparity pixel according to the disparity value
		// 根据视差值计算对应的像素坐标
		const ImageRef v(c+ld,r);
		// check image bounds
		// 确认是否超出图像边界
		if (v.x < 0 || v.x >= r2l.width()) 
		{
			ld = NO_DISP;
			return;
		}
		// check right disparity is valid
		// 确认右视差是否有效
		const Disparity rd = r2l(v);
		if (r2l(v) == NO_DISP) 
		{
			ld = NO_DISP;
			return;
		}
		// check disparity consistency:
		//   since the left and right disparities are opposite in sign,
		//   we determine their similarity by *summing* them, rather
		//   than differencing them as you might expect
		// 因为左右视差本身是相反的d和-d ,所以计算两个视差的查是ld+rd，大于阈值thCross丢弃
		if (ABS(ld + rd) > thCross)
			ld = NO_DISP;
	};
	ASSERT(threads.IsEmpty());
	if (!threads.empty()) 
	{
		volatile Thread::safe_t idxPixel(-1);
		FOREACH(i, threads)
			threads.AddEvent(new EVTPixelProcess(l2r.size(), idxPixel, pixel));
		WaitThreadWorkers(threads.size());
	} else
	for (int r=0; r<l2r.rows; ++r)
		for (int c=0; c<l2r.cols; ++c)
			pixel(-1, r, c);
}
```
---
## 代价聚合
上面代价计算讲完了，现在讲下代价聚合。为什么在openMVS中要做代价聚合呢？这也是我们在立体匹配这一节介绍那里说到的，立体匹配方法总共有三种：
```
全局立体匹配：效果好，但是速度慢
局部立体匹配：速度快，但是效果差
半全局立体匹配：兼顾速度与效果
```
先说如果不做代价聚合的结果就是，代价匹配只考虑局部范围内的相关性，因此对噪声会非常是敏感(这种只考虑局部的待机计算，会出现各自为营的情况)，所以得到的视差结果会非常差。
在openMVS中的立体匹配算法用到了半全局立体匹配算法SGM。因此在做完特征点的代价计算之后，我们需要进行代价聚合操作，使得视差图更具有一致性，这一步有些像前面的选择最佳领域帧，总之是让全局的代价聚合结果达到最优。