HM的scc版本，即SCM是对屏幕内容的编码优化，其中的运动搜索优先使用基于hash匹配的方式，如果没有找到再使用邻域的运动搜索。流程如下：

![HMMotionEstimation](HMMotionEstimation.assets/HMMotionEstimation.png)

### HashSearch

使用了二级hash，分别使用了两种crc。当两种crc hash值相等时，认为找到了相同的block。遍历所有相同的block计算rdo cost。并且当当前图像的qp小于参考图像的qp时，认为是PerfectMatch。

### MotionSearch

HM提供的移动搜索有四种形式：

```c++
enum MESearchMethod
{
  MESEARCH_FULL              = 0,
  MESEARCH_DIAMOND           = 1,
  MESEARCH_SELECTIVE         = 2,
  MESEARCH_DIAMOND_ENHANCED  = 3,
  MESEARCH_NUMBER_OF_METHODS = 4
};
```

其中：

“MESEARCH_FULL”是在一个给定的范围内按照光栅扫码的方式遍历所有的块

“MESEARCH_DIAMOND”和“MESEARCH_DIAMOND_ENHANCED”只有细微的区别，搜索方式相似。搜索过程步长按照2的倍数变化，1，2，4，8直到mv的限制。其中又分为两种具体的方式：

- DiamondSearch:

  以起点为原点，搜索一个菱形的所有点。**包括4个顶点之间连线上的点**

- SquareSearch:

  以起点为原点，搜索周围8个矩形点。