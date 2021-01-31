## 整体模块

openh264中编码之前有一个重要的前处理模块，这个模块中会进行滚动检测，场景切换检测，复杂度分析等等。这些结果会指导后续的编码过程。**CWelsPreProcess::BuildSpatialPicList**函数为每一帧编码前的预处理入口

主要的代码逻辑在CWelsPreProcess中，两个派生类CWelsPreProcessVideo/Screen，应用在不同的场景上。

```c++
class CWelsPreProcess {
 public:
 private:
 	SPicture*        m_pLastSpatialPicture[MAX_DEPENDENCY_LAYER][2];
 	SPicture*        m_pSpatialPic[MAX_DEPENDENCY_LAYER][MAX_REF_PIC_COUNT + 1];
 	SPicture*        m_pSpatialRefPic[MAX_DEPENDENCY_LAYER];
 	SPicture*        m_pRef_saved;
 	int32_t           m_iAvaliableRefInSpatialPicList;
 };
```

上面列举了几个重要的成员变量，这些都是framebuffer。首先，需要明确的一点是这里**所有的frame都是原始图像**，不是重构图像。所有前处理都是在视频序列中的原图像上进行的。对于同一个空域层，有如下几个buffer

```c++
SPicture*        m_pLastSpatialPicture[2];
SPicture*        m_pSpatialPic[MAX_REF_PIC_COUNT + 1];//长度为参考图像个数+1,idx=0为当前图像
SPicture*        m_pSpatialRefPic;//当前帧的参考帧
SPicture*        m_pRef_saved;//参考帧缓存
```

在screen下，最重要的是m_pSpatialPic。存储当前帧和不同时域层的参考帧

### 初始化

在**CWelsPreProcess::AllocSpatialPictures**函数中会初始化，m_pSpatialPic的大小为2+最大时域层数+长期参考帧个数，idx=0为当前帧，从1开始后续均为参考帧。

### 更新

在当前帧编码完成之后会调用**CWelsPreProcess::UpdateSrcList**函数，更新m_pSpatialPic。保证idx=0为丢弃的buffer，可以用作存储下一个原始帧。

其余的buffer，在screen没有时域分层时似乎并没有起作用

## ScrollDetection

### 检测原理

在preprocess中检测当前帧相对于参考帧是否存在滚动，如果存在滚动则给出ScrollFlag和ScrollMv。后续场景检测中会使用ScrollMv判断某个block是否为滚动静止，在运动搜索的过程中会尝试ScrollMv。

目前的滚动检测只有垂直方向上的滚动检测，ScrollMv.iMvX=0。垂直滚动检测的做法比较简单，在当前的图像中寻找一条固定宽度的线段，在参考图像中寻找相同的线段。如果找到了并且在上下一定区域内像素值都相同。则判断为滚动，给出ScrollMv。

### 具体实现

设图像的宽度为W，高度为H。滚动检测区域的宽度为W-(1/8)H，设为regionW。高度为H，设为regionH。整个搜索会在regionW * regionH范围内进行。搜索线段的宽度为(1/6)regionW。把检测区域的宽度方向三等分，在每个区域中的中心的(1/6)regionW中进行垂直方向搜索。即下图中的绿线区域内：

![ScrollDetection](openh264Preprocess.assets/ScrollDetection-3094063.png)

每个绿色区域内又在垂直方向分为上中下3个区域内进行搜索，具体高度划分的区域可以参考代码，选择不同的Yoffset，在searchH=(7/8)H的垂直范围内搜索。因此，一帧的滚动检测会在9个区域中进行，每个区域的检测步骤如下：

- 在区域中选择一条相对复杂的线段，不同的像素值超过3个，或者有多次像素值的变化。作为搜索标志
- 在参考帧对应的给定的垂直范围内进行搜索
- 如果找到相同的线段，则搜索相邻的上下区域。如果也相同则判定为滚动，给出ScrollMv

代码实现：ScrollDetecion.cpp