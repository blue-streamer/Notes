深度编码

深度编码和深度学习以及图像处理基础较差，不通过 在网易主要做基于屏幕内容的编码优化，一般内容会有2%的码率减少，运算速度慢2-3倍。 Q：深度编码项目相比已有的google的图像编码的改进与创新？ A：加了channel attention的机制，全局信息优化 Q：non-local模块的特点？ A：全局信息，而不是局部特征值的 A：缺点，是运算速度比较慢，网络难以收敛。 Q：时域的深度编码？ A：p帧，通过光流找当前帧和参考的帧，光流图进行编码，再warp，讲的也不是很清楚 Q：光流是怎么产生？ A：不清楚，pwcnet，不清楚 Q：fsrcnn的优点，特点 A：不需要 l1 loss和l2 loss的区别？ 说不出来 Q：perceptional loss？ 说不出来 Q：prelu和relu的区别 回答错误 Q：batch normalization的定义和作用？ 说不清楚



ROI

对非ROI区域进行降噪，去除高频分量，这样会降低高频分量，降低码率

宏块的ac 能量计算

https://arxiv.org/pdf/1606.01299.pdf

https://zhuanlan.zhihu.com/p/219925964





 MPEG/DNNVC and JVET/AG11

Catch up with SOTA performance, i.e., as reported in Cheng 2020 CVPR

https://zhuanlan.zhihu.com/p/219925964



rdo paper

Affine 266/av1 LU分解

Obmc



Summary, ratecontrol

CInputPicManage::scenecutDetect LTR

summary，统计学习，主要总结决策树，逻辑斯特回归（贝叶斯），朴素贝叶斯

Lookahead  qa  cutree

paper 1. ml partition 2. Invers transform

为什么使用bits = k * cplx/qscale的模型？1. 从残差编码的角度理解，2. 统计实际编码数据，进行分析，3. qscale，qstep与lambda关系 4. 模型是否可以改进

