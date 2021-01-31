h264中的预测mv和skip mv的计算有点绕，这里总结一下

首先相对当前宏块有四个位置，A，B，C，D。A左侧宏块，B上方宏块，C右上宏块，D左上宏块。在推到mvp和skip mvp时，存在一个宏块“不可用”的语境，这里是指相应位置的宏块不存在。例如第一行的宏块，B，C，D位置就不存在

对于使用帧内预测的宏块，refIdx=-1，mvx=0，mvy=0。对于skip宏块，refIdx=0，mvx=skipMvx，mvy=skipMvy

### Skip mv预测

以下任意条件为true，skip mv=(0,0)

- A不可用
- B不可用
- A的refIdx=0，且mv=(0,0)
- B的refIdx=0，且mv=(0,0)

否则，使用mvp的计算方式得到skip mvp

### mvp计算方式

16x8和8x16有定义好方式，其余的使用中值预测

中值预测第一步：确定ABC的状态

如果C不可用，使用D来代替C

如果BC均不可用，且A可用，则refIdxBC=refIdxA，mvBC=mvA

中值预测第二步：计算

如果ABC有且仅有一个(设为N)，refIdxN=refIdxCur。则mv=mvN

否则使用中值计算。取A,B,C三者的mvx的中值为mvx，mvy的中值为mvy。注意这个中值是指(mvxA+mvxB+mvxC - min(mvxA,mvxB,mvxC) - max(mvxA, mvxB, mvxC))。**如果ABC中只有两个refIdx等于refIdxCur，也是计算三者的中值**。同理，如果ABC三个refIdx都不等于refIdxCur，也是计算中值