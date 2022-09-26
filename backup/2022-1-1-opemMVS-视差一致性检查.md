---
layout: post
title:  "视差一致性检查"
date:   2022-06-16
categories: 三维重建
---
* content
{:toc}

---

视差一致性检查的目的是保证左右图的视差是一致的，我们用左视差图与右视差图去对应，理论上对应的位置上，视差值除了符号不一样，绝对值应该是一样的。通过左右视差图，左右的对比，相互的去判断彼此的视差是否一致。

```
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