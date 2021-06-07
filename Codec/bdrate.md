

bdrate是衡量两个编码器版本的性能指标，在相同的图像质量的情况下码率的变化情况。bdrate越小，相同图像质量下压缩率越高，编码器性能越好。

因此，我们需要得到图像质量和码率之间的关系，称为dist-rate曲线。这里的dist可以使用psnr，ssim，vmaf等。进行编码测试之后，一般情况下只能得到若干个dist-rate数据点。计算bdrate分为下面三个步骤：

1. 通过dist-rate数据点，插值得到dist-rate曲线
2. 选定计算范围，[minDist, maxDist]
3. 在[minDist, maxDist]范围内，计算两条曲线的积分。由于希望控制dist来比较rate，因此以dist为x轴，rate为y轴，来计算积分。得到的积分值为两个版本的编码器在[minDist, maxDist]范围内，每个dist指标下码率消耗之和
4. 计算[minDist, maxDist]范围内的rate平均变化，即为bdrate

而第一步计算得到dist-rate曲线时，可以采用不同的插值方法。为了均衡高码率和低码率下对bdrate的贡献，计算曲线时会对rate求log

<img src="videoCodecMeasurement.assets/截屏2020-10-13 下午9.32.32.png" alt="截屏2020-10-13 下午9.32.32" style="zoom:50%;" />



## 插值

### 三次函数

这种方法把[minDist, maxDist]范围内的dist-rate曲线看成是一个三次函数:
$$
y(x) = a + b*x + c*x^2+d*x^3
$$
上述方程有4个参数，因此需要4个测试数据点来获得参数。4个测试点数据可以构成一个线性方程组
$$
\begin{bmatrix}r_0&r_1&r_2&r_3\end{bmatrix} = \begin{bmatrix}a&b&c&d\end{bmatrix} *  \begin{bmatrix}1&1&1&1\\ d_0&d_1&d_2&d_3\\ d_0^2&d_1^2&d_2^2&d_3^2\\ d_0^3&d_1^3&d_2^3&d_3^3\\ \end{bmatrix}
$$


计算插值就是求上面公式中4x4矩阵的逆矩阵，可以使用伴随矩阵的求解方式。不过，这种方法只能使用4个数据点来计算bdrate

### 分段三次函数

bdrate对插值方法进行了改进，使用了分段三次函数插值，即把每个数据点之间的曲线都看作是一个三次函数，规定某些约束条件求出参数。bdrate具体使用的是pchip保形插值算法，规定在两个曲线相交的数据点上，一阶导数值相等，并规定了一阶导数值的计算方法。如果规定一阶导数值和二阶导数值都相等，就是三次样条插值。三次样条插值保证数据点是光滑的，但是曲线形状可能会有扭曲，因此bdrate选择了pchip保形插值。插值算法的具体公式，可以参考文档[1]

分段三次函数插值是分段计算的，因此有一个默认要求。输入的自变量必须是单调递增的，不然公式不成立。但是在编码测试中，在较高或者较低的码率点上，有可能出现dist指标在波动。而bdrate的计算需要以dist为自变量。因此，这种情况下直接使用插值公式是不正确的，需要对dist排序之后再计算

## 计算bdrate

得到插值函数之后，需要在选取的[minDist, maxDist]范围内进行积分计算。得到每个dist下码率消耗之和。anchor记为va，test记为vb
$$
bdrate = 10^\frac{vb-va}{maxDist-minDist} - 1
$$
由于rate在插件计算中取log，最后结果取10为底的指数即可



## 参考

> https://netflixtechblog.com/performance-comparison-of-video-coding-standards-an-adaptive-streaming-perspective-d45d0183ca95
>
> https://tools.ietf.org/id/draft-ietf-netvc-testing-06.html
>
> https://www.mathworks.com/content/dam/mathworks/mathworks-dot-com/moler/interp.pdf



