码率控制的目标是控制编码器输出的码流的大小，通常的做法是通过调整qp来达到控制的目的。因此，简单来说码控就是通过设定合理的qp，来达到码控码率的目的。

## 模型

对于设置合理的qp来达到控制码流大小这个问题，需要一个模型。所有的码控都会使用下面这个简单的模型：
$$
bits = k * \frac{complex}{qscale} \tag{1}
$$
以帧为单位来说，bits可以理解为是一帧的编码之后的码流大小。complex是复杂度，可以认为是残差的大小。qscale可以理解为是量化步长。这个公式很直观，如果一帧越复杂，相同的条件下，码流越大。如果量化步长越大，相同的条件下，码流越小。（为什么使用这个模型，从残差的编码角度）

## 工程实现

模型虽然简单，工程实践上需要解决很多问题，然后需要结合一些其他的方法加以辅助。以下总结在rtc场景下的帧间码控

### Complex计算

由于在进行码控，计算qp的时候，还没有编码。因此，实际的complex是未知的。一种获得complex的方法是，预先进行一次简单的模型选择。即对所有的块进行帧内和帧间预测，得到sad。使用这个sad作为complex的预测。

bytevc1中使用的就是这种方法，基本的模式是以8x8为单位，计算cost。进行intra md，得到最佳的mode的sad。进行inter md，做简单的me，得到最佳的mv，然后计算得到sad+lambda*mvd。比较intra和inter，得到最佳的cost。累计所有8x8块的cost，得到一帧的cost

对于rtc的场景，模型中使用的并不是每一帧的cost，而是使用一种渐消遗忘滤波的方式得到
$$
\begin{aligned}
&costSum = k * costSum + curCost\\
&costCount = k* costCount + 1\\
&cost = \frac{costSum}{costCount}
\end{aligned}
$$
k可以去0.5，这种处理时为了平滑cost的变化。这种平滑之后得到的cost，称为blurredComplex

得到了blurredComplex之后，还有一个参数qcompress。最后公式中使用的complex，如下计算
$$
complex = blurredComplex ^ {1-qcompress}
$$
qcompress是一个控制complex对码控影响的参数，如果为1，则complex恒等于1。则complex对码控无影响。这个值一般被称为rceq

### k的计算

k是一个在编码过程中不断被统计更新的值，更新公式1中的complex为rceq，得到如下：
$$
k = \frac{bits * qscale}{rceq}
$$
因此，在一帧编码结束之后，可以得到当前帧实际的bits，使用的qscale和编码前计算的rceq。从而计算得到k，然后与k的累计值计算平均到最终的k，从而可以再下一帧的计算中使用。

而在实际的编码前的码控计算中，直接使用的是一个rateFactor的值。rateFactor = bits / k。bits是一帧期望的输出码流的大小，可以通过bitrate * frameDuration得到。然后bits也通过与之前的累计值计算平均得到，然后直接更新rateFactor。

这里k和bits的更新也可以使用渐消遗忘滤波的方式

### 计算qp

#### rateFactor QP

1. 计算rceq

   首先计算当前帧的cost，进而得到blurredComplex。结合qcompress参数，计算得到rceq

2. 计算qp

   根据之前编码数据更新的rateFactor和计算得到的rceq，
   $$
   rfQp = qscale2qp(\frac{recq}{rateFactor})
   $$

3. 编码完成之后，更新rateFator

除了上述计算qp的方式之外，还有另外一种qp，crfQp。即const rateFactor Qp

#### crf QP

这种计算与上述方式的差异在于，rateFactor是一个固定值，在初始化是设定好：
$$
constRateFactor = \frac{baseCplx}{qp2qscale(initQp)}
$$
baseCplx是一个固定值，cuNum*80。InitQP是26。crfQp的计算如下：
$$
crfQp = \frac{cost^{1-qcompress}}{constRateFactor}
$$
这种方式计算出来的qp，并不是为了控制输出码流的大小，而是使得每一帧质量保证一致(?)。

### smoothQp

在rtc的场景下，会使用rfQp和crfQp来计算给出一个建议的qp，称为smoothQp。rfQp用来给出建议的qp，crfQp则是对建议的值进行了最大最小值的限制。

- 计算deltaQp

对于rfQp和crfQp，都有一个累计统计值。accRfQp和accCrfQp，当前帧码控时，会计算平均值aveRfQp和aveCrfQP，进而得到delta值
$$
\begin{aligned}
deltaRfQp &= rfQp - aveRfQp \\
deltaCrfQp &= crfQp  - aveCrfQp
\end{aligned}
$$

- 计算qp

$$
qp = aveRcQp + deltaRfQp * weight + 0.5
$$

Weight = 0.5

- 计算qpRange

$$
\begin{aligned}
preCrfQpDelta &= (preCrfQP - curCrfQp) * 2\\
offsetUp &= CLIP3(2.0,   3.0, MAX(preCrfQpDelta, deltaCrfQP * 2))\\
offsetDwn &= CLIP3(-3.0, -2.0, MIN(fPreCrfDelta, fDeltaCrfQP * 2))\\
\\
&minQp = preQp + offsetDwn \\
&maxQp = preQp + offsetUp
\end{aligned}
$$

最后得到smoothQp
$$
smoothQp = CLIP3(minQp, maxQp, qp)
$$

### BitsWindow

那么，smoothQp计算出来的qp是否准确呢？即是否可以达到码控的目标？一般码控会给出一个目标码率bitrate和最大最小码率maxBitrate，minBitrate。控制目标就是每秒的输出码流大小。因此，如果能得到算上当前帧在内的1s内的码流大小，通过这个大小来反馈调整当前帧的qp，则可以很好地控制码率。

已编码输出的帧，码流大小记录即可得到，但是当前帧尚未编码。因此，需要一个qp与bits的预测模型。有了这个模型，通过已编码帧的码流统计值+当前帧码流的预测值就可以得到码流大小了。有了这个模型，我们可以反复微调qp来达到目标码率

#### predictBits

##### predModel

公式1是不是预测模型呢？它确实给出了一个qp与bits之间的关系，但是这个模型的complex是recq，是长时间窗口的加权平均。这个简单模型用于长时间加权平均的qp预测。而目前需要的是预测一帧的bits，因此使用了另外一种模型：
$$
\begin{aligned}
&bits = \frac{a * cost + b}{qscale} \\
&bits = \frac{bits}{1+accumME*0.25}
\end{aligned}
$$
可以看到，相比较公式1，这里使用用了线性模型，多了一个参数。而且，认为模型的预测与实际情况是有误差的，误差比例使用accumAE来表示。0.25是一个调整系数。

为了提高预测的准确性，使用了分段线性模型。即不同的cost值和不同的frameType，对应了不同的模型。因此，这个预测模型是一组线性模型。predModel01[type\][costIdx]。一共使用了22个costIdx。

predModel还有另外一组，即只是按不同的frameType分，并没有使用costIdx。即predModel02[type]。

在每一帧编码完成之后，都会更新predModel：
$$
\begin{aligned}
&a = \frac{bits*qscale-b}{cost}\\
&b = bits*qscale-a*cost
\end{aligned}
$$
先更新coeff，再更新offset

然后计算当前的预测误差：
$$
\begin{aligned}
&err = \frac{\frac{a*cost+b}{qscale}-bits}{bits}
\end{aligned}
$$
使用渐消遗忘滤波的方的方式更新accumME。

对于predModel01[type\][costIdx]和predModel02[type]，会比较两者的accumAE（误差绝对值的累计值）。选择一个最好的模型，用于后续的预测。

##### costBitsModel

除此之外，还有一个统计模型。即统计某种frameType，某个qp，某个costIdx下的frameCost和frameBits。costBitsModel[type\][qp\][costIdx]

当得到smoothQp之后，使用

1. predModel预测得到 bits
2. 对于costBitsModel，使用costIdx，costIdx-1，costIdx+1。得到预测的bits
3. 使用costBitsModel中相邻qp的数据，训练一个predModel模型。预测得到bits

综合这三者的结果，给出smoothQp的预测bits

#### updateWindow

得到smoothQp的预测bits之后，需要更新window。这是使用了两个window，shortTimeWindow(1s)，longTimeWindow(2s)。统计得到对应窗口中的平均码率。

#### updateBrStats

根据到目前为止的平均码率，ST/LTwindow的码率统计值等等，给出一个码率状态。BR_LOW，BR_NORMAL，BR_HIGH。根据这个状态来微调smoothQp。然后重新预测bits，重新更新判断。直到码率状态为normal。

因此BitsWindow是一个重要的码控模块，通过统计码率，预测bits。反复调整qp，直到码率得到控制目标

### 整体流程梳理

1. 计算cost，得到rceq
2. 通过rfQp和cfrQp得到smoothQp
3. smoothQp输入到预测模型，预测当前帧的bits。更新bitsWindow，得到码率状态
4. 根据码率状态，调整smoothQp，回到3。直到码率状态为normal
5. 编码
6. 得到实际的bits，更新rateFactor，predModel，bitsCostModel等