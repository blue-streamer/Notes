​	最近在做了一些代码优化，主要目的是让程序运行的更快，cpu消耗更少。在图像编解码和处理中，会有很多计算密集型的任务。因此，对这些代码的优化是很有意义的。

​	有一点需要注意。“让程序运行的更快”和“cpu消耗的更少”有时候并不是等价的。例如，使用多线程技术。如果根据任务合理使用多线程来完成计算，显然会加速。但是，cpu需要进行的运算并没有减少，相反，任务的拆分和线程间的同步还需要消耗更多的资源和cpu运算。多线程的好处在于可以更好地发挥多核机器的能力，让一个任务可能分配到不同的核上同时执行。cpu负载更为均衡。

​	多线程是一个重要优化手段，需要根据任务的不同合理设计分配，这里不多展开。下面总结一下从改变代码结构本身来优化的方法。一段代码的执行主要包括两部分操作：cpu读写数据和cpu进行运算。**优化时要从最耗时的代码段或函数入手，这些地方是代码的瓶颈。使用工具（如vtuns）找到这些地方是第一步也是最重要的一步**。

- 优化cpu运算：
  - 浮点运算转化为定点运算
  - 使用simd
  - 对图像进行padding，减少边界条件判断
- 优化cpu读写：
  - 改变内存排布，高效利用cache

### 定点化

​	浮点运算就是有浮点类型的参与的运算，浮点数的加法和乘法相比较整数而言都需要更多的指令周期。所谓的**浮点运算转化为定点运算**指的就是使用整数运算和移位来模拟浮点运算。

​	比如高斯模糊这个算法，选择好模板大小之后，需要根据公式计算出系数。然后选择可以接受的精度位数，变为整数，并保证整数系数的和为2的整数倍（这个过程可能需要微调系数，在保证算法效果可接受的前提下）。最后移位。例如，一维系数为[0.25, 0.5, 0.25]，此时可以转化为[2，4，2]。对应的像素与其相乘，最后的结果右移3位即可。这样需要浮点相乘的计算转化为了整数相乘和移位。

### simd

