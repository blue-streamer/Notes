hevc中的palette是一种没有tu的编码方式，使用三种数据在解码端建立重构像素：

1. palette entry：这是一个一维数组，其中存放不同的颜色。如果是yuv的格式，每种颜色由三个分量组成
2. palette index：这是一个与block大小相同的二维数组，其中存放的是palette entry中的idx(0~size-1)。表明这个位置的重构像素使用entry中的idx标识的像素。如果idx=size，说明重构像素不使用palette entry，使用escape value
3. escape value：这是一个与block大小相同的二维数组，其中存放的是对应位置的escape value(如果使用)。escape value提供了一个在palette模式中编码palette entry之外的值的方式。escape value是一个经过量化的值，解码端反量化之后得到重构数据



bytevc1中的palette编码分为simple palette和common palette。当前block只有一种颜色时，称为simple palette。当depth < 3时，只使用simple palette。common palette编码流程主要分为如下流程：

1. 生成source，luma取一半，4个数据取对角两个
2. 统计直方图，聚类。把相近的颜色聚合成一类。
3. 对聚类的结果进行排序，生成palette entry。对生成的palette entry 寻找最佳的预测
4. 决策每个像素的index和escape value

