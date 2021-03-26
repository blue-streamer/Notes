s264为了提高屏幕内容的压缩率，引入了很多特定的编码工具。屏幕内容的有两个主要特点：

- 一帧内或者两帧之间重复的内容会比较多，例如一页文档中会出现很多相似的文字
- 当前帧和参考帧之间可能会出现大范围的整体运动，例如文档的上下滚动和窗口的位置移动

基于这些特点，如果可以准确地找到与当前待编码块相同或者相似的已编码块（可以在当前帧中或者参考帧中），则可以提高压缩效率。为了达到这个目的，引入了基于hash值的匹配算法。

### 基于hash匹配的运动搜索的基本原理

首先选择一个基本单元，例如8x8的block，这个block就是hash匹配的基本单元，后续所有的搜索匹配都是以这个block为单位的。

然后选择一个hash函数，以block的像素为输入可以得到一个hash值。不同的hash函数的选择，会有不同的匹配结果。如果希望是精确匹配，block到hash值的映射需要时一对一的，例如block的所有像素计算crc32，当然这种hash函数会出现碰撞的可能性，需要加以考虑。如果希望是模糊匹配，block到hash值的映射可以是多对一的，例如block的所有像素计算均值。这样所有均值相同的block会被统一筛选出来，集中在一起。

精确匹配可以准确找到完全相同的block，模糊匹配会有一定的容错性。例如，已编码块中不存在完全相同的块，但只有一个像素值存在轻微差距。或者完全相同的已编码块距离当前位置较远，但是相似块距离较近。这种情况下模糊匹配会提供更多的选择，但是需要进一步的遍历来查找，需要额外的计算量。

对于屏幕内容而言，模糊匹配会提供更多的信息，可能会达到更高的编码效率。

### s264的hash匹配实践

以8x8的block为基本单元，使用了模糊匹配的hash函数。hash函数选择如下：

<img src="s264HashMatch.assets/image.png" alt="image" style="zoom:50%;" />

8x8块划分为4个4x4块，Hash 值共16bit，其中：

[bit0-bit4)：4bit：存放梯度纹理信息，整个8x8块梯度的高4bit。

[bit4-bit7)：3bit：4x4块0的均值的高3bit

[bit7-bit10)：3bit：4x4块1的均值的高3bit

[bit10-bit13)：3bit：4x4块2的均值的高3bit

[bit13-bit16)：3bit：4x4块3的均值的高3bit

8x8块的梯度的计算：首先计算每个像素的梯度。每个像素的梯度计算方法为：（水平梯度 + 垂直梯度）>> 1。则整个 8x8 块的梯度为：除去第一行第一列像素，每个像素的梯度的和的均值

这个hash值称为firstHash

确定好最小单位和hash函数之后，需要把hash匹配运用到编码的过程中。当前块如果参考当前帧中的已编码块，需要使用ibc mode。如果参考参考帧中的已编码块，需要使用inter mode。为了可以快速找到可以参考的已编码块，引入hashTable

#### hashTable

hashTable是一个2^16大小的数组，其下标对应hash值，这样可以使用hash值直接索引。由于是模糊匹配，使用链地址法来解决冲突。数组的每个位置存在一个指针，是一个单链表的头结点。链表的每个节点存放一组block信息，其中包括位置posX，posY和二级hash aHash。

节点存放一组block信息是为了减少链表节点个数的创建和访问消耗。当前块计算得到hash之后，需要变量对应的链表，计算sad来寻找最佳的参考块。二级hash存在的目的是为了减少sad的计算，通过二级hash可以快速筛选。

<img src="s264HashMatch.assets/hashTable.png" alt="hashTable" style="zoom:75%;" />

aHash是求取 8x8 块亮度的平均值，比较每个亮度值同平均值的大小，亮度值大于等着均值，则为 1，否则为 0，得到 64个bit，组成 8 个 byte。搜索链表时，首先会计算两个aHash的距离，距离的计算就是两个aHash不同bit的个数。只有距离很小时，才会计算sad。

hashTable的可以用于快速hash匹配搜索，但是建立hashTable需要有较多的计算消耗。对于ibc mode，需要建立当前帧已编码块的hashTable。对于inter mode，需要建立参考帧整帧的hashTable。为了充分利用已编码的信息，从理论上讲需要把每一个像素点为左上角的8x8block的信息都加入到hashTable中。对于1920x1080的图像，则存在(1920-7)*(1080-7)个block，计算量很大。可以引入如下优化点：

- **高纹理区域检测**

  仅仅对复杂纹理区域的block计算hash并插入hashTable。对于平坦区域，被参考的收益并不高。计算一级hash时会计算水平梯度gradX和垂直梯度gradY，如果gradX == 0 或 gradY == 0 或 gradX+gradY < threshHold，则可以判定为低纹理区域。

- **hashTable复用**

  在编码当前帧时，需要建立当前帧已编码block的hashTable。当编码结束之后，当前帧完整的hashTable已经建立完成。因此，当前帧作为后续编码的参考帧时，这个hashTable可以被复用作为编码帧inter mode所需的hashTable

另外一个重要的优化方法是**hashImg**

#### hashImg

如果当前帧与前一帧相比，只有很小的变化。例如，一个窗口的拖动。此时大部分的block是没有变化的，因此前后帧的hashTable变化也比较小。如果可以只计算变化部分的hash并更新的hashTable中，这样的计算量就会很小。但是hashTable是通过hash值来索引的，直接修改比较困难。因此引入hashImg

hashImg是一个与编码图像大小相同的二维数组，每个元素是以该像素点为左上角的8x8block的hash值，包括firstHash和aHash。因此这个二维数组右边的7列和下边的7行没有数据，其余存储的都是hash值。

<img src="s264HashMatch.assets/hashImg.png" alt="hashImg" style="zoom:75%;" />

在编码的前处理模块，有一个motionDetection模块，会检测出当前图像相对参考图像每个8x8block的类型，包括static，scroll和motion。分别对应了静止快，滚动块和运动块(新出现的块)。利用每个block的类型情况，就可以节省大量的计算。

在一帧编码之前，前处理之后，会利用参考帧的hashImg计算出当前的hashImg。在当前帧编码时，会根据当前的编码位置，从hashImg中读取hash值，不断建立和更新hashTable。此时的hashTable用于ibc mode。编码结束之后，hashImg和hashTable会保存下来。当后续编码流程中当前帧被选作参考帧时，hashTable会用于inter mode的me，hashImg辅助计算新的hashImg。

通过参考帧的hashImg辅助计算当前的hashImg以及通过hashImg建立hashTable这两个过程依然是性能热点，为此需要有进一步工程优化。

### 基于hash匹配搜索的整体流程

在完整的编码过程中，需要计算hashImg，编码过程中更新hashTable。流程如下：

<img src="s264HashMatch.assets/hashMatchFlow.png" alt="hashMatchFlow" style="zoom:75%;" />



