s264为了提高屏幕内容编码的压缩效率，引入了很多新的编码工具和编码算法。例如palette，ibc，smooth，globalMv等等。palette，ibc和smooth等是帧内编码工具，帧间编码除了运用上述工具之外，主要有以下改进：

- 基于hash匹配的运动搜索算法，可以更有效地检测出大范围的运动和更精确地匹配。与ibc复用相同的hash
- 检测屏幕内容的整体运动，充分利用skip和globalMv来提高压缩率

其中主要改造了前处理和编码流程

### prerprocess

根据设计，前处理模块需要完成如下任务：

- 丢弃连续相同的帧
- 选择最佳的参考帧，并判定是否场景切换
- 判定每个block的类型，并给出mv list
- 检测是否存在整体运动，并得到globalMv list

由于hash搜索都需要在原图上进行。因此，前处理模块维护了一个srcList。list的大小与最大参考帧个数相关。对于丢弃相同的帧，这个比较简单。当前帧和前一帧计算sad，如果完全相同则丢弃。

对于屏幕内容，经常前后帧大范围的相同，或者整体运动。因此，block可以分为三个类型：

- **static**：相同位置完全静止的块
- **scroll**：整体运动的块
- **motion**：不属于上面两种，可以认为是新出现的块

如果当前帧相对于一个参考帧，motion块非常多可以认为差异比较大，而如果static和scroll块比较多可以认为是一个好的参考选择。因此，参考帧选择和场景切换判断的逻辑也比较简单。对每个可用的参考帧做运动检测，判定block类型。从中选择motion block最少的帧作为最佳参考帧。如果可用的参考帧相比当前帧都有比较大的变化，可以认为发生了场景切换。此时可以考虑使用I帧。

因此给定一个参考帧，相对于这个帧对当前帧进行运动检测是前处理模块的关键

#### 运动检测

所谓运动检测就是要判定当前帧与参考帧之间的相对运动情况，在检测过程中会给出每个block的**type**，**blockMvList**等信息，以及整帧的**globalMvList**（如果存在整体运动）。如果以选定的参考帧作为实际的编码参考帧，这些信息会用于编码过程。因此，运动检测的输入：

- **当前帧**和**候选参考帧**

运动检测的输出：

- block info：**type，globalMv，blockMvList，similarMvList**

  **type**用于指明当前block的类型，如果当前block为scroll，会有对应的**globalMv**

  **blockMvList**是hash搜索的结果，运动检测是基于hash匹配的运动搜索。这mv_list中存放了所有的相同的参考块的运动矢量

  **similarMvList**也是hash匹配的运动搜索的搜索结果。由于使用了模糊hash，因此会得到相似的参考块

  这些所有信息都会保存在**blockInfo**中，用于后续的编码

- frame info：**globalMvList**

  如果存在整体运动，globalMvList存放的是当前帧所有的整体运动矢量

在运动检测的过程中，最为重要的是基于hash匹配的运动搜索和基于hash搜索的整体运动检测，具体流程可以参见文档。在此之后，可以的到block type，blockMvList，globalMvList等重要信息。而对于有效信息比较少的block，例如motion block且blockMvList为空，此时会进行相似块搜索得到**similarMvList**。

#### 整体流程

整个前处理模型需要进行丢帧判断，基于运动检测的参考帧选择，场景切换判定，hashImg更新等工作，整体流程如下：

<img src="s264InterEncode.assets/preprocessFlow.png" alt="preprocessFlow" style="zoom:75%;" />



注：目前s264中只使用了一个可用参考帧，因此motion detect和calc hashImg只会进行一次。如果有多参考帧选择，calc hashImg应该放在choose best reference frame之后。使用最佳参考帧的hashImg来计算当前的hashImg

### encode

在编码流程中的主要改进是使用先处理得到的blockInfo信息，充分利用motion detection和hash search的结果。主要流程如下：

1. blockInfo 信息辅助进行模式选择

   利用block **type**，**globalMv**等信息辅助选择编码模式。如果成功，直接进行编码

2. 模式选择

   对当前宏块进行编码模式选择，利用**blockMvList**信息辅助运动估计，尝试包括ibc在内的各种帧内预测模式

3. 编码

   根据模式选择的结果进行编码

4. 更新hashTable

   编码结束之后，通过hashImg中当前宏块的hash信息更新hashTable。用于后续宏块的ibc搜索

整体流程如下：

<img src="s264InterEncode.assets/s264InterEncodeFlow.png" alt="s264InterEncodeFlow" style="zoom:75%;" />

#### blockInfo 信息辅助进行模式选择

对当前宏块的4个block类型进行检测，如果都是static或者globalMV 相同的scroll。此时可以认为最佳的mv已经得到，则可以跳过后续模式选择的过程。首先计算predMv，尝试skip。否则尝试p16x16或者globalMv两种种编码模式，选择最优。

注意，由于所有的前处理都是在原图luma上进行的，而参考帧是重构图像。因此需要注意：**chroma一致性**问题和**qp参数的变化**

<img src="s264InterEncode.assets/scdSkip.png" alt="scdSkip" style="zoom:75%;" />

#### 帧间预测模式选择

帧间预测的模式选择主要是进行运动估计，这里主要使用p16x16，p16x8，p8x16和p8x8 四种编码模式。由于screen场景中可能会出现视频，因此需要把传统的diamond search和基于hash匹配的运动搜索进行结合。

hash匹配的运动估计结果在preprocess中已经得到，存放在blockMvList中。而hash匹配的运动估计是基于block的，也就是得到的是8x8block的运动估计结果。这个结果需要进一步做改进：

- 对每个block增加diamond search

  对于diamond search来说，搜索起始点的选择十分重要。因此，添加ABCD四个位置的mv作为mvc。结合hash search和diamond search，得出每个block的最佳运动估计结果

- 对宏块中每个block的运动估计结果做融合

  一个宏块中的4个block，可能给出了4个不同的mv。尝试更大的编码块如p16x8，p8x16和p16x16。尝试找出最佳的编码模式和mv

在此外再添加p16x16的diamond search，最后得到帧间模式选择和运动估计

<img src="s264InterEncode.assets/s264InterModeDecision.png" alt="s264InterModeDecision" style="zoom:75%;" />



#### 帧内预测模式选择

帧内预测模式中，使用了I16x16，smooth，palette，ibc，I4x4等模式。较为平坦的块会使用I16x16，smooth。颜色个数比较少的情况下，palette有比较好的RDO性能。细节较多的区域I4x4比较合适。对于可以参考当前slice已编码的区域，ibc有很好的压缩率。

整个模式选择主要是通过RDO来选择的，在此基础上会根据宏块特征来决定是否尝试某种模式和一些提前跳出的模式：

- 分析宏块特征，决定是否尝试某些模式

  计算颜色个数，颜色较少时可以尝试palette

  计算梯度，梯度较大时可以尝试ibc

  计算4x4块亮度均值的方差，方差较大时可以尝试I4x4

- 当某种模式cost足够小时，提前跳出

具体来讲，首先尝试I16x16和smooth模式。计算颜色个数，判断是否尝试palette。计算4x4block的方差，判断是否尝试4x4。计算梯度，判断是否尝试ibc。每个模式之后都会进行提前跳出判断。

#### 编码和更新hashTable

模式选择结束后，进行编码。然后从hashImg中读取当前宏块位置的hash信息，更新hashTable