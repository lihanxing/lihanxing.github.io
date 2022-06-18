---
layout: post
title:  "深度图优化"
date:   2022-06-18
categories: 三维重建
---
* content
{:toc}

---
在openMVS中，深度图优化分为两种，单帧深度图优化&帧间优化。

## 单帧深度图优化

![单帧深度图优化部分框架](/img/openMVS/单帧深度图优化部分.png)
单帧深度图优化是openMVS框架中，在进行完SGM/PatchMatch后的内容，执行单帧深度图优化的原因是我们计算出的深度图或多或少会带有一些孔洞或者连通域，单帧深度图优化为的就是对这一部分做优化。
openMVS的单帧深度图优化有两部分：**去除连通域**和**补洞**，对应的函数分别为**RemoveSmallSegments**和**FilterDepthMap**，如上图所示，需要主要的是**去除连通域**操作在**补洞**操作的前面。

---

### 去除连通域

![](/img/openMVS/单帧深度图优化示意图.png)
去除连通域的具体过程如上图所示，蓝色部分为计算得到的深度区域，其中有三处为突兀的深度区域，可以看到它们很小，且与周围的像素不一致。去除连通域的目的就是，将那些突兀的且小的深度块，其像素值重置为0，可以从下面的代码中看到。
```
				// 把无效的像素深度都置为0
				for (unsigned i=0; i<seg_list_count; ++i) 
				{
					depthMap(seg_list[i]) = 0;
					if (!normalMap.empty()) normalMap(seg_list[i]) = Normal::ZERO;
					if (!confMap.empty()) confMap(seg_list[i]) = 0;
				}
```
当然了，对于连通域也有判断，如果某部分的区域确实很大，那就不能简单的理解为连通域，而是正确深度图的一部分，这从nSpeckleSize这个参数也可以看出来。

下面是**RemoveSmallSegments**的全部代码：
```
// filter out small depth segments from the given depth map
// 从给定的深度图中过滤出小的深度段
bool DepthMapsData::RemoveSmallSegments(DepthData& depthData)
{
	const float fDepthDiffThreshold(OPTDENSE::fDepthDiffThreshold*0.7f);
	unsigned speckle_size = OPTDENSE::nSpeckleSize;//判断是否为不连续连通域的阈值?
	DepthMap& depthMap = depthData.depthMap;
	NormalMap& normalMap = depthData.normalMap;
	ConfidenceMap& confMap = depthData.confMap;
	ASSERT(!depthMap.empty());
	const ImageRef size(depthMap.size());

	// allocate memory on heap for dynamic programming arrays
	// 在堆上为动态编程数组分配内存
	TImage<bool> done_map(size, false);
	CAutoPtrArr<ImageRef> seg_list(new ImageRef[size.x*size.y]);
	unsigned seg_list_count;//
	unsigned seg_list_curr;//
	ImageRef neighbor[4];//假如某像素属于连通域

	// for all pixels do
	// 逐像素处理
	for (int u=0; u<size.x; ++u) 
	{
		for (int v=0; v<size.y; ++v) 
		{
			// if the first pixel in this segment has been already processed => skip
			// 如果这个段中的第一个像素已经被处理=>跳过
			if (done_map(v,u))
				continue;

			// init segment list (add first element
			// and set it to be the next element to check)
			// 初始化 分割list
			seg_list[0] = ImageRef(u,v);
			seg_list_count = 1;
			seg_list_curr  = 0;

			// add neighboring segments as long as there
			// are none-processed pixels in the seg_list;
			// none-processed means: seg_list_curr<seg_list_count
			// 只要seg_list中有未处理的像素，就添加相邻的分割块
			while (seg_list_curr < seg_list_count) 
			{
				// get address of current pixel in this segment
				// 取当前像素在这个分割块中的地址
				const ImageRef addr_curr(seg_list[seg_list_curr]);
				const Depth& depth_curr = depthMap(addr_curr);

				if (depth_curr>0) 
				{
					// fill list with neighbor positions
					// 用邻域像素填充list
					neighbor[0] = ImageRef(addr_curr.x-1, addr_curr.y  );
					neighbor[1] = ImageRef(addr_curr.x+1, addr_curr.y  );
					neighbor[2] = ImageRef(addr_curr.x  , addr_curr.y-1);
					neighbor[3] = ImageRef(addr_curr.x  , addr_curr.y+1);

					// for all neighbors do
					// 处理每一个邻域
					for (int i=0; i<4; ++i) 
					{
						// get neighbor pixel address
						// 取邻域坐标
						const ImageRef& addr_neighbor(neighbor[i]);
						// check if neighbor is inside image
						// 确认邻域是否在图像内
						if (addr_neighbor.x>=0 && addr_neighbor.y>=0 && addr_neighbor.x<size.x && addr_neighbor.y<size.y) 
						{
							// check if neighbor has not been added yet
							// 确认邻域是否已经被处理过
							bool& done = done_map(addr_neighbor);
							if (!done) 
							{
								// check if the neighbor is valid and similar to the current pixel
								// 确认邻域是否属于当前分割块
								// (belonging to the current segment)
								const Depth& depth_neighbor = depthMap(addr_neighbor);
								if (depth_neighbor>0 && IsDepthSimilar(depth_curr, depth_neighbor, fDepthDiffThreshold)) 
								{
									// add neighbor coordinates to segment list
									// 如果属于则加到分割块的list中
									seg_list[seg_list_count++] = addr_neighbor;
									// set neighbor pixel in done_map to "done"
									// (otherwise a pixel may be added 2 times to the list, as
									//  neighbor of one pixel and as neighbor of another pixel)
									// 标记邻域已被处理过
									done = true;
								}
							}
						}
					}
				}

				// set current pixel in seg_list to "done"
				// 在seg列表中设置当前像素为“已完成”
				++seg_list_curr;

				// set current pixel in done_map to "done"
				// 标记当前像素已被处理过
				done_map(addr_curr) = true;
			} // end: while (seg_list_curr < seg_list_count)

			// if segment NOT large enough => invalidate pixels
			// 如果分割块大小不够大，就认为是无效的将其剔除
			if (seg_list_count < speckle_size) 
			{
				// for all pixels in current segment invalidate pixels
				// 把无效的像素深度都置为0
				for (unsigned i=0; i<seg_list_count; ++i) 
				{
					depthMap(seg_list[i]) = 0;
					if (!normalMap.empty()) normalMap(seg_list[i]) = Normal::ZERO;
					if (!confMap.empty()) confMap(seg_list[i]) = 0;
				}
			}
		}
	}

	return true;
} // RemoveSmallSegments
```
---
### 补洞
在去除连通域之后，还需要进行的优化步骤是补洞操作，补洞是为了填补上留下来的深度值区域中一些值缺失的区域，使得整个深度值区域能够连续。补洞的示意过程如下图中，最大蓝色区域中的洞那里，再补洞后该部分被补齐。在openMVS的代码中，补洞的逻辑比较好理解，
    具体过程为：在同一y轴上得到洞两侧端点A和B的深度值，计算(depth(A)-depth(B))/(A-B)，得到的结果diff，这个diff就是A和B之间的平均深度差；在补洞过程中，从B到A，依次地址diff，用线性插值的方式将hole补上。
![](/img/openMVS/单帧深度图优化示意图.png)
具体代码如下：
```
// try to fill small gaps in the depth map
// 填充小的洞
bool DepthMapsData::GapInterpolation(DepthData& depthData)
{
	const float fDepthDiffThreshold(OPTDENSE::fDepthDiffThreshold*2.5f);
	unsigned nIpolGapSize = OPTDENSE::nIpolGapSize;
	DepthMap& depthMap = depthData.depthMap;
	NormalMap& normalMap = depthData.normalMap;
	ConfidenceMap& confMap = depthData.confMap;
	ASSERT(!depthMap.empty());
	const ImageRef size(depthMap.size());

	// 1. Row-wise: 处理行
	// for each row do
	for (int v=0; v<size.y; ++v) {
		// init counter
		unsigned count = 0;

		// for each element of the row do处理每一行的每个像素
		for (int u=0; u<size.x; ++u) {
			// get depth of this location
			// 取深度值
			const Depth& depth = depthMap(v,u);

			// if depth not valid => count and skip it
			// 无效跳过，并记录
			if (depth <= 0) 
			{
				++count;
				continue;
			}
			if (count == 0)
				continue;

			// check if speckle is small enough
			// 判断洞是否足够小
			// and value in range
			if (count <= nIpolGapSize && (unsigned)u > count) 
			{
				// first value index for interpolation
				// 第一个要插值的索引
				int u_curr(u-count); //左侧的值
				const int u_first(u_curr-1);
				// compute mean depth
				// 计算洞的两端深度的平均深度
				const Depth& depthFirst = depthMap(v,u_first);
				if (IsDepthSimilar(depthFirst, depth, fDepthDiffThreshold)) 
				{
					#if 0
					// set all values with the average
					const Depth avg((depthFirst+depth)*0.5f);
					do {
						depthMap(v,u_curr) = avg;
					} while (++u_curr<u);						
					#else
					// interpolate values
					// 线性插值
					const Depth diff((depth-depthFirst)/(count+1));
					Depth d(depthFirst);
					const float c(confMap.empty() ? 0.f : MINF(confMap(v,u_first), confMap(v,u)));
					if (normalMap.empty()) 
					{
						do {
							depthMap(v,u_curr) = (d+=diff);
							if (!confMap.empty()) confMap(v,u_curr) = c;
						} while (++u_curr<u);						
					} 
					else 
					{
						Point2f dir1, dir2;
						Normal2Dir(normalMap(v,u_first), dir1);
						Normal2Dir(normalMap(v,u), dir2);
						const Point2f dirDiff((dir2-dir1)/float(count+1));
						do {
							depthMap(v,u_curr) = (d+=diff);
							dir1 += dirDiff;
							Dir2Normal(dir1, normalMap(v,u_curr));
							if (!confMap.empty()) confMap(v,u_curr) = c;
						} while (++u_curr<u);						
					}
					#endif
				}
			}

			// reset counter
			count = 0;
		}
	}

	// 2. Column-wise: 处理每一列同上
	// for each column do
	for (int u=0; u<size.x; ++u) 
	{

		// init counter
		unsigned count = 0;

		// for each element of the column do
		for (int v=0; v<size.y; ++v) 
		{
			// get depth of this location
			const Depth& depth = depthMap(v,u);

			// if depth not valid => count and skip it
			if (depth <= 0) 
			{
				++count;
				continue;
			}
			if (count == 0)
				continue;

			// check if gap is small enough
			// and value in range
			if (count <= nIpolGapSize && (unsigned)v > count) 
			{
				// first value index for interpolation
				int v_curr(v-count);
				const int v_first(v_curr-1);
				// compute mean depth
				const Depth& depthFirst = depthMap(v_first,u);
				if (IsDepthSimilar(depthFirst, depth, fDepthDiffThreshold)) 
				{
					#if 0
					// set all values with the average
					const Depth avg((depthFirst+depth)*0.5f);
					do {
						depthMap(v_curr,u) = avg;
					} while (++v_curr<v);						
					#else
					// interpolate values
					const Depth diff((depth-depthFirst)/(count+1));
					Depth d(depthFirst);
					const float c(confMap.empty() ? 0.f : MINF(confMap(v_first,u), confMap(v,u)));
					if (normalMap.empty()) {
						do {
							depthMap(v_curr,u) = (d+=diff);
							if (!confMap.empty()) confMap(v_curr,u) = c;
						} while (++v_curr<v);						
					} else {
						Point2f dir1, dir2;
						Normal2Dir(normalMap(v_first,u), dir1);
						Normal2Dir(normalMap(v,u), dir2);
						const Point2f dirDiff((dir2-dir1)/float(count+1));
						do {
							depthMap(v_curr,u) = (d+=diff);
							dir1 += dirDiff;
							Dir2Normal(dir1, normalMap(v_curr,u));
							if (!confMap.empty()) confMap(v_curr,u) = c;
						} while (++v_curr<v);						
					}
					#endif
				}
			}

			// reset counter
			count = 0;
		}
	}
	return true;
} // GapInterpolation
```

---
## 帧间优化

帧间深度图优化是在单帧的基础上，做的优化。帧间深度图优化的目的是，使得不同帧之间的深度图更“一致”。
帧间优化的策略有两组
* 对于某一帧深度图中的某一有效深度值，检查其领域帧相同位置的深度值是否一致，统计同一位置深度值一致的领域帧个数，将这个个数与设置的阈值作比较，大于等于该阈值时符合条件保留该深度值，否则丢弃该位置的深度值。
* 对于某一帧深度图中的某一有效深度值，检查该深度值与领域位置的深度值是否一致，一致的话就保存。
**总结：帧间滤波不但判断与领域帧的深度值是否一致，也判断与领域的深度值是否一致**