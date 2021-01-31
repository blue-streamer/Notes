RDO

2、“现有基于块平移模型的快速运动估计/补偿技术均是根据自然视频的运行向量统计特点（如中心偏置特性）所设计的”

理解RDO过程，看编码器

查看h264提案，RDO 模型

JVT-O079

学习 HM r lambda qp 模型

码率控制

https://slidesplayer.com/slide/17101924/

深度编码

深度编码和深度学习以及图像处理基础较差，不通过 在网易主要做基于屏幕内容的编码优化，一般内容会有2%的码率减少，运算速度慢2-3倍。 Q：深度编码项目相比已有的google的图像编码的改进与创新？ A：加了channel attention的机制，全局信息优化 Q：non-local模块的特点？ A：全局信息，而不是局部特征值的 A：缺点，是运算速度比较慢，网络难以收敛。 Q：时域的深度编码？ A：p帧，通过光流找当前帧和参考的帧，光流图进行编码，再warp，讲的也不是很清楚 Q：光流是怎么产生？ A：不清楚，pwcnet，不清楚 Q：fsrcnn的优点，特点 A：不需要 l1 loss和l2 loss的区别？ 说不出来 Q：perceptional loss？ 说不出来 Q：prelu和relu的区别 回答错误 Q：batch normalization的定义和作用？ 说不清楚



ROI

对非ROI区域进行降噪，去除高频分量，这样会降低高频分量，降低码率

B. Li and J. Xu, *Hash-Based Motion Search*, document JCTVC-Q0245, JCT-VC, Valencia, Spain, Mar. 2014.

MulticoreWare. (2015). *x265 1.8 Release*. [Online]. Available: https://bitbucket.org/multicoreware/x265/downloads

W. Xiao, B. Li, and J. Xu, *Bottom-Up Hash Value Calculation and* Validity Check*, document JCTVC-W0078, JCT-VC, San Diego, CA, USA, Feb. 2016.



cabac 上下文模型

信息熵

学习

https://arxiv.org/pdf/1606.01299.pdf

https://zhuanlan.zhihu.com/p/219925964

