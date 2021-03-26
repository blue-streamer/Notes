openH264中的码控模式有4种：**Quality**，**Bitrate**，**BufferBased**，**TimeStamp**

Quality和BufferBased模式重点在控制编码质量，而Bitrate和TimeStamp模式重点在于控制输出码率。**timeStamp**模式比较适用于rtc场景：

timeStamp模式的码控主要分为两部分：**Frame level** 和 **Gom level**：

- Frame level，主要有两部分：
  1. 分配帧级码率。时域分层的权重 + BufferFullness反馈调节，设定每一帧的期望码率
  2. 帧级QP计算。以期望码率为输入，通过R-QP模型计算QP
- Gom level，主要有两部分：
  1. 分配Gom级码率
  2. 调整Gom QP



## Frame Level

### Init

openH264的码控以GOP为单位，这个GOP是指时域分层结构中的。3层时域，则每个GOP有4帧。0层1帧，1层1帧，2层2帧。编码器初始化之后会根据时域分层的参数，设定每一层的码控参数。主要有**权重**，**QP范围**。

例如，设置时域分层为3层，QP为[24,28]。则权重比例：40%:30%:15%（40+30+15+15=100）。QP为[24 + (n<<1), 28 + (n<<1)]，n取0，1，2。

### UpdateBitrateFps

openH264的码控有两个重要的输出参数，**bitrate**和**framerate**。根据这两个参数计算出bitsPerFrame和GopBits。计算出

- 不同层每一帧的码率最大最小值，

  minBits = minRatio * weight * GopBits，maxBits = maxRatio * weight * GopBits。默认设置minRatio=0.55，maxRatio=1.5

- BufferSkipSize,

  bitrate * bufferSkipRatio。bufferSkipRatio 默认0.5

### DecideTargetBits

计算每一帧的期望编码Bits。重要的参数：bufferTh = bufferSkipSize - bufferFullness，表征编码器输出和信道之间虚拟buffer的充盈度。

如果 bufferTh < 0 

targetBits = minBits

否则，这里I帧和P帧的逻辑略有不同

- P Frame

  targetBits = GopBits * weight。

  maxTh = bufferTh / 2，minTh = bufferTh * 2 / framerate。低码率情况下(framerate<8)，minTh = bufferTh / 4

- I Frame

  targetBits = bitsPerFrame * idr_ratio。idr_ratio 默认设置为 4，低码率情况下(framerate < idr_ratio + 1)，idr_ratio = 1

  maxTh = bufferTh * 3 / 4，minTh  = bufferTh * 2 / framerate。低码率情况下(framerate<8)，minTh = bufferTh / 4

targetBits 约束在[minTh，maxTh]

### CalculateQp

这里openH264使用简单的R-QP模型，定义frameCmplex为当前帧与参考帧之间的sad，frameBits为实际编码输出的大小，QStep为$\lambda$。R-QP模型为：
$$
frameCmplex = \alpha * frameBits * QStep
$$
即一帧编码的大小与QStep的乘积和frameCmplex成正比。这个模型和x264中ABR的模型是一致的，不同点有两个

- x264提供了一个qcompress来调整frameCmplex对模型的影响程度
- x264中的frameCmplex计算更加精确，在编码前进行了帧内预测(I frame)和me(p frame)

使用这个模型，计算QP分为三步：

1. iCmplexRatio = iFrameCmplex / iFrameCmplexMean

   计算当前复杂度和统计平均复杂度的比率，并限制在[0.8，1.2]之间。iFrameCmplexMean是已经编码帧的加权统计值

2. iQStep = iLinearCmplex * iCmplexRatio / targetBits

   使用帧复杂度的比例来计算QStep。iLinearCmplex是已编码帧的frameBits * QStep 的加权统计值。

3. iQStep -> QP

得到QP之后需要计算minFrameQP和maxFrameQP来进行约束，这个约束保证两帧之间的QP不会有太大的变化，并且考虑到不同时域层的区别。最后约束在当前时域层的maxQP和minQP之间。注意这里的lastQP是上一帧实际编码QP的统计平均值。

minFrameQP = clip3(lastQP - frameDeltaQPLower + DeltaQPTemporal, minQP, maxQP)

maxFrameQP = clip3(lastQP + frameDeltaQPUpper + DeltaQPTemporal, minQP, maxQP)

注意上述R-QP模型统计值，iFrameCmplexMean和iLinearCmplex。I帧和P帧进行区别统计



### 编码和Gom层调节

使用得到QP对图像进行编码，如果开启Gom码控，则进行的调整。

### UpdatePictureInfo

1. 更新QP和FrameBits的统计值
2. 更新R-QP模型中的iFrameCmplexMean和iLinearCmplex。更新的策略比较简单，和历史值加权平均

$$
iFrameCmplexMean = 0.2 * curFrameCmplex + 0.8 * iFrameCmplexMean \\
iLinearCmplex = 0.2 * (FrameBits * QStep) + 0.8 * iLinerCmplex
$$

3. 更新bufferFullness，bufferFullness += FrameBits

### UpdateFps

使用1s的时间窗口，更新Fps的统计值

### FrameDelayJudgeTimeStamp

在下一帧送入编码器之后，进行跳帧判断：

1. 根据时间戳计算两帧间信道的发送bits，sendBits = bitrate * timeInv
2. 更新bufferFullness，bufferFullness -= sendBits
3. 如果bufferFullness > bufferSkipSize，则码控判断进行跳帧



## Gom Level

Gom 是group of mb的缩写，为了弥补帧级码控和R-QP模型的误差，实现更加精确的码率控制。在完成Frame QP计算之后，以Gom为单位来微调QP。Gom的控制是通过当期分配的目标码率和实际生产码率之间差别，调整后续Gom的码率分配和QP设置。Gom的划分为一行MB的整数倍，分辨率越大Gom也会越大。720p的Gom是2行MB。

需要注意的是Gom是在一个Slice中的，如果当前编码开启了多线程，编码了多个Slice。则Gom会被关掉

### Init

开始Gom码控之前会初始化一些参数，targetBitsSlice = sliceMbNum * BitsPerMb。第一个Gom使用Frame QP

### GomTargetBits

每个Gom开始时会计算当前Gom的期望码率，首先计算iLeftBits = targetBitsSlice - frameBitsSlice。frameBitsSlice是当前编码实际生产的bits。

- 如果 iLeftBits < 0：

  iGomTargetBits = 0

- 否则，平均分配或者按照每个Gom的sad权重进行分配

### Gom编码

第一个Gom使用Frame QP，后续的Gom QP会在上一个Gom编码结束后微调

### CalculateGomQp

评估当前实际编码输出bits和期望值之间的差距，调整下一个Gom QP

计算两个重要参数：

- iLeftBits = targetBitsSlice - frameBitsSlice

  当前slice剩余的bits

- iTargetLeftBits = iLeftBits + iGomBitsSlice - iGomTargetBits

  iGomBitsSlice为当前Gom实际编码输出的bits，iTargetLeftBits则是当前Gom编码之后期望的剩余bits，即当前Gom输出的bits，期望与实际相等的情况。

计算iBitsRatio = iLeftBits / iTargetLeftBits，调整QP。这里使用的模型依然是Frame Level里的R-QP模型：Bits * QStep = $\alpha$Complexity。这里认为Gom之间的Complexity相同，因此bits与QStep成反比，因此可以根据iBitsRatio计算出调整后的QP。



## 代码注释

timeStamp mode：

pRcf->pfWelsRcPictureInit = WelsRcPictureInitGom;
pRcf->pfWelsRcPictureInfoUpdate = WelsRcPictureInfoUpdateGomTimeStamp;
pRcf->pfWelsRcMbInit = WelsRcMbInitGom;
pRcf->pfWelsRcMbInfoUpdate = WelsRcMbInfoUpdateGom;

pRcf->pfWelsRcPicDelayJudge = WelsRcFrameDelayJudgeTimeStamp;
pRcf->pfWelsCheckSkipBasedMaxbr = NULL;
pRcf->pfWelsUpdateBufferWhenSkip = NULL;
pRcf->pfWelsUpdateMaxBrWindowStatus = UpdateMaxBrCheckWindowStatusTimeStamp;
pRcf->pfWelsRcPostFrameSkipping = NULL;



```c++
typedef struct TagWelsRc {
  int32_t   iBufferSizeSkip;//frame skip buffer size
  int64_t   iBufferFullnessSkip;//当期buffer的充盈度，通过统计实际编码输出和计算信道输出得到的。iBufferFullnessSkip > iBufferSizeSkip表明超编过多，进行skip
  int32_t   iSkipFrameNum;//统计skip frame num
  int32_t   iFrameDqBits;//当前帧编码的实际bits
  int32_t*   pGomCost;//每个Gom的宏块实际编码的LumaCost统计值 
}
```

```c++
typedef struct TagRCSlicing {
  int32_t   iComplexityIndexSlice;
  int32_t   iCalculatedQpSlice;
  int32_t   iStartMbSlice;
  int32_t   iEndMbSlice;
  int32_t   iTotalQpSlice;//slice QP统计值
  int32_t   iTotalMbSlice;
  int32_t   iTargetBitsSlice;//slice 分配的bits
  int32_t   iBsPosSlice;
  int32_t   iFrameBitsSlice;//slice实际消耗的bits，统计值
  int32_t   iGomBitsSlice;//当前Gom实际消耗的bits，统计值
  int32_t   iGomTargetBits;//当前Gom分配的bits
  //int32_t   gom_coded_mb;
} SRCSlicing;
```

