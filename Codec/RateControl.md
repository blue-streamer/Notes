video codec的码率控制一般分为两个步骤来完成：目标码率分配，计算qp

264/265提案中的方案和相应的代码实现都是按照这两个大步骤来完成的。**目标码率分配**是指对相应的编码目标分配码率，如GOP，pic，CTU(mb)等等。**计算qp**是指得到目标码率之后，根据相应的模型来计算qp。这一步不同的标准(264/265)甚至不同的codec实现都可能有差异

## 目标码率分配

简化考虑认为相邻的GOP之间相对独立，所以GOP的分配比较简单，按照码率、帧率(时间戳)和GOP的长度来计算每个GOP的bits即可

GOP中不同帧的分配问题，如果以整个GOP中图像失真之和最小为目标，理论上存在最优解。但是实践上会根据帧类型或者时域分层分配不同的权重。

在目标码率分配阶段，更重要的一个问题是怎么样才能控制输出的码率准确，这在RTC的场景中尤为重要。因此很多提案和论文都引入了flow buffer的机制，可以起到缓冲码率波动和更精确地控制码率输出的作用

### flow buffer

整体思想是假定在编码器的输出和信道之间存在一个buffer，编码器的输出是填充buffer，信道的发送是消耗buffer。而码率控制算法的目的就是使得这个buffer的充盈度B~c~

<img src="RateControl.assets/截屏2021-02-28 下午4.37.41.png" alt="截屏2021-02-28 下午4.37.41" style="zoom:25%;" />

T(n)是第n个图像编码完之后输出的码流大小，u(n)/F~r~ 是这一帧信道发送的码流大小，u为码率，Fr为帧率。buffer的实时容量使用B~c~(n)来表示。
$$
B_c(n+1) = B_c(n) + T(n) - \frac{u}{F_r}
$$
码率控制的目标就变成了控制B~c~(n)为一个恒定的值或者在某个范围内，然后计算出某个图像等效分配的码率。

一般情况下第一个GOP会初始化buffer的容量为B~s~/8，当前GOP的buffer初始化值是前一个GOP结束时buffer的容量。每个GOP控制目标都是结束时的buffer容量为B~s~/8，这样每个GOP都还使用了自己时间段内的信道容量。B~cpre~ 表示上一个GOP结束时的buffer容量，TB表示整个GOP分配的码率，R表示当前GOP剩余的码率，则：
$$
R = TB + B_s/8 - B_{cpre}
$$
设I帧的权重为$w_i$，P帧的权重为$w_p$。GOP中有一个I帧和$N_p$个P帧。编码I帧和P帧之后，会对buffer的大小产生影响$Delta(P)$和$Delta(I)$：
$$
Delta(I) = \frac{w_i}{w_i + w_p * N_p} * R - \frac{u}{F_r} \\
Delta(P) = \frac{w_p}{w_i + w_p * N_p} * R - \frac{u}{F_r}
$$
设编码第n帧之前buffer的目标大小为target(n)和实际大小$B_c(n)$，target(n)是按照上述理想情况下的Delta，buffer容量的变化
$$
target(n+1) = target(n) + Delta
$$
引入控制系数$\gamma$。则第n帧的buffer控制算法码率：
$$
f0(n) = \frac{u}{F_r} + target(n+1) - \gamma*target(n) + (\gamma - 1)*B_c(n)
$$
如果计算适当的QP和编码之后，$f0(n) = T(n)$则可以看到：
$$
B_c(n+1) = B_c(n) + f(n) - \frac{u}{F_r}
$$
有如下关系：
$$
B_c(n+1) - target(n+1) = \gamma*(B_c(n) - target(n))
$$
$\gamma$的取值范围为(0,1)，这样就可以控制buffer的充盈度了。

上述的控制方法可以实现输出码率的控制，但是会导致不同帧之间的码率分配不再符合预设定的权重关系($w_i, w_p$)。设$T_r(n)$是编码第n帧之前，当前GOP的实际剩余码率，按照权重分配，当前帧的分配比例为$\alpha$。例如，如果GOP中没有B帧，则可以认为剩下的P帧平均分配剩余的码率。根据权重比例得到的帧级的码率分配：
$$
f1(n) = \alpha * T_r(n)
$$


因此，最后帧级的码率分配会综合flow buffer和权重分配的结果：
$$
f(n) = \beta*f0(n) + (1 - \beta)*f1(n)
$$




如果码控出现很大的偏差，需要在进行跳帧的操作。如果
$$
B_c(n) >= 0.8B_s
$$
则跳过这一帧，此时buffer的更新公式变为：
$$
B_c(n+1) = B_c(n) - \frac{u}{F_r}
$$




## 计算QP

QP的计算会使用一个R-QP模型，根据第一步分配好的码率来计算即可。264的JVT-G012中使用了一个二次模型，265的JCTVC-K0103中使用了$R-\lambda-QP$ 模型。

### JVT-G012

JVT-G012中的二次模型是根据理论模型推到出来的，设R是码率，H为残差之外的编码信息，M是MAD(mean absolute difference)。则公式为：
$$
R = c2*(\frac{M}{QP})^2 + c1*\frac{M}{QP} + H
$$
可以看到公式中，R是通过码率分配得到的结果。但是，M和H是未知的。这就是264码控中的一个悖论，码控计算QP需要知道当前的M和H，但是M和H在RDO之后才能得到，而RDO需要QP作为输入参数。因此，提案中使用可一个线性预测的方法，使用前一帧的MAD~pre~来预测当前帧的MAD~cur~：
$$
MAD_{cur} = a1 * MAD_{pre} + a0
$$
H也可以使用预测的方法，或者在高码率情况下忽略。

可以看到上述的两个模型中存在未知系数c2，c1，a1，a0。a1和a0比较简单，设置初始值为1和0，在每一帧编码之后拟合更新。c2和c1的值和视频序列特性相关，可以使用相似的视频序列预回归得到初始值，然后编码过程选择一个窗口来更新拟合系数。需要注意的是，当场景发生较大变化时，需要减少之前系数对后续系数的影响。

### JCTVC-K0103

265的JCTVC-K0103中的$R-\lambda-QP$ 模型是建立了一个$R-\lambda$关系，而$\lambda-QP$的关系是已知的:
$$
\lambda = \alpha * bpp^\beta
$$
bpp(bits per pixel)，这个值也是可以从码率分配值得到的。其中$\alpha$,$\beta$值是前一帧的编码结束后的更新值。即使用了某个$\lambda$编码之后，可以得到实际bpp，然后更新参数：
$$
\lambda_{new} = \alpha_{old} * (bpp_{actual})^{\beta_{old}} \\
\alpha_{new} = \alpha_{old} + \theta_{\alpha} *(ln\lambda_{old} - ln\lambda_{new})*\alpha_{old} \\
\beta_{new} = \beta_{old} + \theta_{\beta} *(ln\lambda_{old} - ln\lambda_{new})*lnbpp_{actual} \\
$$
如果实际编码的$bpp_{actual}$太小，例如大部分CU都是skip，则使用如下方式更新：
$$
\alpha_{new} = 0.96*\alpha_{old}\\
\beta_{new} = 0.98*\beta_{old}
$$
其中的$\theta_{\alpha}$和$\theta_{\beta}$分别设为0.1和0.05，$\alpha$和$\beta$的值会限制在[0.05,20]和[-3.0,-0.1]之间。



可以看到265中$\lambda-QP$模型并没有引入残差的信息，而这个变量显然会影响R。可以认为，参数$\alpha, \beta$已经包含了残差的预测，在JVT-G012的提案中MAD也是通过已编码的数据预测出来的。



## 亚帧级码率控制

上述两步可以理解为帧级的码率控制，通过分配码率和计算QP得到的是整帧的QP。为了实现更为精准的控制，H264和H265都提出了亚帧级的控制。

**JVT-G012**中提出了一个Basic Unit Layer，可以是宏块，宏块组或者slice。控制方法分为三步：

##### 1、为当前的basic unit 分配码率

当前帧剩余的码率$f_{rb}$，当前帧剩余的basic unit个数$N_{ub}$。预测当前basic unit残差之外的预测信息的码率$m_{hdr}$（可以使用已编码的basic unit和前一帧的信息来预测，也可以根据更多信息做的更精确）
$$
R = \frac{f_{rb}}{N_{ub}} - m_{hdr}
$$

##### 2、计算QP

1. 第一个 basic unit 使用前一帧的所有basic unit QP的平均值，$QP_{apf}$
2. $f_{rb}<0$的情况，设前一个basic unit的QP为$QP_{pb}$，则$QP_{cb}=QP_{pb} + DQP$。$DQP$为自定义的QP增量，1或者2。
3. 如果不是前两种情况，使用MAD预测和二次R-D模型来计算QP，得到$QP_{cb}$。其中的MAD模型应该与basic unit对应

最后，计算得到的QP进行平滑：
$$
QP_{cb} = \max(QP_{pb}-DQP, \min({QP_{cb}, QP_{pb}+DQP})\\
QP_{cb} = \max(QP_{apf} - \Delta, \min(QP_{cb}, QP_{apf} + \Delta))
$$

##### 3、更新$QP_{apf}$

编码完所有的basic unit之后，更新$QP_{apf}$



**JCTVC-K0103**中也有CTU级的码率控制，大体类似。$\lambda-QP$模型是CTU级别的，使用相同层距离最近图像的相同位置的CTU数据更新值。最后加上一些限制：

1. 相邻CTU之间的$lambda$在[$\frac{1}{2^{1/3}}$, $2^{1/3}$]
2. 当前CTU和所属图像之间的$lambda$在[$\frac{1}{2^{2/3}}$, $2^{2/3}$]



可以看到，亚帧级的码率控制在码率分配上都是平均分配，QP的计算上更加细致，不同的位置使用不同的模型。