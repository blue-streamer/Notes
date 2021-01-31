

1、添加编译选项，忽略mv range 的限制

2、提取宏块特征来判断使用哪种模式更合适，高纹理（hash cross palette）,低纹理（diamond 4x4）,平坦（16x16）。低纹理（图片，视频）跳过hash和cross，以及外围的merge。使用diamond search 以及亚像素搜索

3、优化skip，可以有更多的块skip，增加try skip的尝试

4、会出现hash search，diamond search和cross search都失效的情况，当文字区域有非水平竖直移动且画面亮度有变化

5、P mode和I mode之间的选择有问题，尝试完整的rdo cost

6、P mode之间的模式选择，由于变换额外的消耗，也会出现问题。尝试使用SATD代替SAD来做残差大小的判断



2，3属于在现有的算法基础上需要的改进

4 需要新的搜索算法或者mvc选择方法的支持（当给出一个很好的搜索起点时，可能会得到比较好的结果）

5，6属于模式选择的问题，并没有找到“最好”的模式



进一步提高压缩率的方法：

tsm

解决问题4

使用更为灵活的块划分方式

引入merge和amvp





整体运动判断正确，skip mv预测正确的情况下。skip的色度skip判别，无法跳过。应该可以再放宽一些“stop at dc”。原图和重构图的sad和qp相关，如果设置一个阈值，这个阈值需要和qp相关。因此，在scroll中判断色度，编码过程中不需要加sad和系数的判断。



codec 原理

1 motion search的方法，有效性

2 RDO过程，为什么qp 与lambda 相关。这个公式的有效性和哪些因素相关

如何设计实验得到 qp与lambda的拟合关系。如果要改变distorition 的衡量指标

3 码控的过程，



