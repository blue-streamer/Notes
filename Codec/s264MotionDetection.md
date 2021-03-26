为了提高屏幕内容的压缩率，s264在openh264的基础上改进了运动搜索算法。对于屏幕的整体运动有很好的检测效果，显著地提高了压缩率。整体运动的检测是在hash search的基础上完成的。hash search会给出当前block在参考帧中相同的block的位置，表示成mv的形式。然后把所有block的mv尝试聚合，得到整体的运动检测结果。具体过程主要分成3个步骤：

- 初始化block类型
- 获得整体运动 mv
- 修改block类型

下面详细叙述各个步骤

### 初始化block类型

block大小选定为8x8，每个block都会有一个类型和mvList。这个mvList存放当前block的参考位置。block为8x8的大小主要是考虑到264的宏块大小，处理的复杂度等因素。mb大小为16x16，8x8的最小单位有更多的自由度，处理的复杂度也不会太高。

把图像划分为8x8的block，通过当前帧和参考帧做差得到每个block的sad。block类型分为三种：**static**，**motion**，**scroll**。初始化阶段根据sad是否为设置为**static**和**motion**

### 获得整体运动mv

这一步是通过每个motion block的hash search的结果，通过聚合得到整体运动。

hash search简单来说是对参考帧的每个8x8block(步长为1)计算hash值，并以hash值作为索引，建立一个hash table，记录下了每个block的位置。search的时候只需要计算当前block的hash值，查找hash table即可得到mv。

为了减少计算量，当block的属于低纹理，则不进行hash search，判定为flat block。hash table中也不会有相关的信息。低纹理的判断是通过block水平和垂直梯度之和来进行的。

为了获取整体运动mv，设计了**frequencyMvList** 和 **aMvList**。分别为16和2048的大小。可以理解为**frequencyMvList**是当前出现最高频率的mv，**aMvList**是出现过的mv。这两个list中记录了每个mv的值和出现的次数。对于每个block，都有一个**blockMvList**，存放每个block完全相同的参考位置mv。遍历每个block，不断更新**frequencyMvList**和**aMvList**，最终得到结果。这个过程如下：

1. search block mv

   遍历block，对于**motion** && **aMvList** block，首先搜索 **frequencyMvList**，如果命中则添加到**blockMvList**中。否则进行hash search，搜索结果添加到**blockMvList**中

2. vote frequency mv

   每个block得到**blockMvList**之后，遍历所有的mv，进行投票。新增**aMvList**或增加对应mv次数，有必要更新**frequencyMvList**

最终得到的**frequencyMvList**中就是当前出现次数最多的16个mv，最后对**frequencyMvList**排序。过滤掉频次较少的mv，得到整体运动的mvlist，**globalMvList**

整体流程如下：

<img src="screenDetection.assets/getGlobalMv-6480157.png" alt="getGlobalMv" style="zoom:75%;" />

### 修改block类型

获得**globalMvList**之后，需要修改block的类型。这一步需要做两个事情

- 修改相应的motion 到 scroll 
- 在某些整体运动的部分包括背景，此时整体运动的部分中会有一些block 判断为static。需要把这些static修改为scroll

scroll类型就是整体运动的mv指向的参考帧block与当前的block相同，或者整体运动的mv存在于**blockMvList**中。整体分为3步

#### 正向遍历block

这一步尝试把所有的motion block变为scroll，由于flat block没有hash search的信息。策略略有不同

- motion flat block，检查left和top block。如果是scroll，参考mv，尝试修改为scroll。否则，遍历**globalMvList**，尝试修改为scroll
- motion not flat block，遍历**globalMvList**，尝试修改为scroll
- 经过上述修改策略，当前block为scroll情况下，尝试修改left和top的static block。如果mv命中，static修改为scroll

整体流程如下：

<img src="screenDetection.assets/changeBlockType.png" alt="changeBlockType" style="zoom:75%;" />

#### 反向遍历block

正向遍历是从左上到右下的变量，反向遍历正好相反。从右下到左上。这样是为了使得scroll的信息更好地传播到static和motion block。例如有些整体运动的区域中纹理复杂的block集中在右侧或者下方。上述正向变量很难将scroll mv传播到左上方区域。

反向遍历策略如下：

- motion flat block，检查right和bottom block。如果是scroll，参考mv，尝试修改为scroll。
- 经过上述修改策略，当前block为scroll情况下，尝试修改right和bottom的static block。如果mv命中，static修改为scroll

整体流程如下：

<img src="screenDetection.assets/changeBlockType1.png" alt="changeBlockType1" style="zoom:75%;" />

#### 多次遍历block

这一步主要是用于修改static block的类型，使得整体运动的检测更加准确。每次遍历修改策略比较简单：

- 多次正向遍历block，如果当前block是scroll，并且left或top的block的类型为static。尝试修改为scroll

多次遍历，直到遍历的最大次数或者修改的block个数小于阈值，流程如下：

<img src="screenDetection.assets/changeBlockType2.png" alt="changeBlockType2" style="zoom:75%;" />

