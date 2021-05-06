在hevc中，变换量化之后需要对生成的系数进行编码。编码过程分为两步，扫描和编码。扫描的过程是把二维的系数矩阵变成一维的系数，编码对一维的系数使用cabac进行编码，生成码流。

### TB划分和扫描

hevc的变换单位为TB，size 4x4~32x32。但是，扫描的过程是按照4x4 subBlock来进行的。无论TB size是多少，都需要划分为4x4的subBlock。每个subBlock内的系数需要按照某种顺序扫描，得到一个16个连续的系数，称为系数组CG(coefficient group)。subBlock也会按照某种顺序来遍历，所有CG就形成了最终的一维系数。

上述的“某种顺序”，hevc中提供了三种选择。对角(zig-zag)，水平和垂直。一个TB的扫描从**最后一个系数**开始，首先扫描所在的subBlock，然后按照subBlock的顺序到下一个subBlock，一直到第一个系数。扫描的目的是把幅值相近的系数聚集在一起，cabac可以更好的压缩。一般情况下使用对角扫描，水平预测的情况下会使用垂直扫描会有更好的效果，垂直预测则会使用水平扫描。subBlock的扫描顺序和subBlock内部的系数扫描顺序使用同一个

### 系数编码

得到一维系数之后，需要对系数进行编码。大部分情况是高频系数多为0，低频系数比较密集。因此会首先找到扫描顺序中的第一个非零系数（正向扫描的话就是最后一个系数），编码它的位置。剩下的系数以此编码即可。系数的编码分为**位置信息**和**幅值信息**两个部分

#### 位置信息

这个部分涉及6个语法元素，last_sig_coeff_x_prefix/subfix，last_sig_coeff_y_prefix/subfix。这4个语法元素是为了标识第一个非零系数的位置。codec_sub_block_flag和sig_coeff_flag用于标识后续的系数。

不同TB的size会在行列上划分为不同的区域，4x4，8x8，16x16，32x32会划分为4，6，8，10个区域。prefix指区域的idx，subfix指区域中的偏移量。

确定第一个非零系数的位置之后，就确定了所在的subBlock。在扫描顺序之前的subBlock的系数都为0。而后续的subBlock中是否为全零系数由**codec_sub_block_flag**来确定。如果为0，表示当前subBlock中系数全为0。否则表示至少含有一个非零系数。如果某个subBlock对应的**codec_sub_block_flag**为1，则需要编码**sig_coeff_flag**，用于表示当前位置上的系数是否为0。

因此整个编码过程如下：首先扫描得到一维的变换系数，编码last_dig_coeff的位置信息，得到所在的subBlock。然后按照扫描顺序遍历每一个subBlock，根据subBlock系数是否全为零来编码**codec_sub_block_flag**信息。如果subBlock内存在至少一个非零系数，则按照subBlock内的扫描顺序，编码**sig_coeff_flag**，表示当前位置系数是否为0。然后按照subBlock内的扫描顺序，对于**sig_coeff_flag**不为0的位置编码幅值信息

#### 幅值信息

这个部分涉及4个语法元素，**coeff_abs_level_greater1_flag**，表示当前系数的绝对值是否大于1。**coeff_abs_level_greater2_flag**，表示当前系数的绝对值是否大于2。**coeff_sign_flag**，表示当前系数的符号。**coeff_abs_level_remaining**，表示当前系数绝对值的剩余值。如果系数的绝对值使用abs_level表示，则剩余值的计算如下：
$$
coeff\_abs\_level\_remaining = abs\_level - base\_level \\
base\_level = sig\_coeff\_flag + coeff\_abs\_level\_greater1\_flag + coeff\_abs\_level\_greater2\_flag
$$
一个subBlock内的系数的幅值信息编码分为4步：

1. 编码扫描顺序中前8个非零系数的coeff_abs_level_greater1_flag，后续系数默认为0
2. 编码扫描顺序中第一个abs_level > 1的系数的coeff_abs_level_greater2_flag，后续系数默认为0
3. 编码所有非零系数的符号coeff_sign_flag
4. 编码所有非零系数的剩余值coeff_abs_level_remaining

由于每个非零系数都需要编码符号信息，coeff_sign_flag。为了节省码流，hevc提供了一种**符号数据隐藏（Sign Data Hiding, SDH）**的技术。首先计算subBlock内所有非零系数幅值绝对值之和，然后根据绝对值的奇偶性来判断最后一个系数的符号。奇数判定为负，偶数判定为正。如果不一致，需要对subBlock内的系数进行调整。如果不使用RDOQ，则引入下面的方法：计算原始系数值和反量化系数值之间的差值，对差值最大的值进行修正。

