x264中有多种码率控制的方式，大致分为single pass和mulit pass两种。single pass比较适合RTC场景，其中有CQP，CRF，ABR三种。

1. CQP：固定QP
2. CRF：固定码控因子
3. ABR：平均码率，控制平均码率达到设定值

除此之外还有VBV(video buffer verifier)，ABR+VBV控制可以实现CBR，恒定码率控制

这里主要介绍ABR的算法，ABR中主要有帧级的码控，宏块级的码控（AQ）两部分组成：

## 帧级码控

1. 计算当前帧的satd.

   这一步在lookahead之后，每个宏块都对参考帧做了小范围的ME(p 帧)或者帧内预测(I 帧)，然后得到的satd。

2. 计算blurred_complexity.

   blurred_complexity就是satd的平均值：
   $$
   blurred\_complexity[i] = \frac{cplxsum[i]}{cplxcount[i]}
   $$
   

   $cplxsum$是satd的加权平均:
   $$
   cplxsum[i+1] = cplxsum[i]*0.5 + satd[i] \\
   cplxcount[i+1] = cplxcount[i]*0.5 + 1
   $$
   

3. 计算qscale(即$\lambda$).

   x264有一个参数，qcompress。代表了帧的复杂度对码控的影响。这里有一个相关的变量：
   $$
   rceq = blurred\_complexity^{(1-qcompress)}
   $$
   

   qscale的计算公式如下：
   $$
   qscale = \frac{rceq}{rate\_factor}
   $$
   

   计算出来的qscale会根据码控的情况进行调整：
   $$
   qscale = qscale * overflow \\
   overflow = min(max(1.0+\frac{total\_bits - wanted\_bits}{abr\_buffer}, 0.5), 2.0)
   $$

4. 计算qp，编码

5. 更新rate_factor

   在x264中有这样一行注释，“qscale = linearized quantizer = Lagrange multiplier”。这个线性量化器的意思应该是这个量和输出的码率是线性关系。rate_factor就是线性关系的计算结果。

   
   $$
   rate\_factor = \frac{wanted\_bits}{actual\_size} * \frac{rceq}{actual\_qscale}
   $$
   

   其中wanted_bits是根据时间戳和码率计算出来的一帧期望的码率，actual_size是一帧的编码实际的输出。这两个值都是累计值。

   actual_qscale是当前帧根据实际编码qp的统计平均值，rceq是当前帧编码前计算的复杂度相关的量。设qc = qscale/rceq，则：

   
   $$
   qc_{curr} = \frac{actual\_bits}{wanted\_size} * qc_{prev}
   $$
   

   可以认为qscale/rceq和bits成正比，使用累计计算的rate_factor来预测计算下一次的qscale。调整依据是当前的累计实际编码码率和期望编码码率的比值，所以ABR控制的目标是一段时间的累计码率，等效为平均码率。可以看到如果qcompress设置为1，则rceq恒等于1，复杂度则不会影响码控。