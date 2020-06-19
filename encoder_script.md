encoder中使用了很多函数指针，在InitFunctionPointers函数中初始化。下面记录一下WelsCodePSlice中用到的函数指针：

pfInterMd --> WelsMdInterMb 宏块模式决定和编码函数

pfInterMdBackgroundDecision --> WelsMdInterJudgeBGDPskipFalse

pfSCDPSkipDecision --> WelsMdInterJudgeSCDPskipFalse

pfMotionSearch --> WelsMotionEstimateSearch 运动搜索估计

sSampleDealingFuncs.pfSampleSad --> WelsSampleSad16x16_c_sse

pfInterFineMd --> WelsMdInterFinePartitionVaa

pfCheckDirectionalMv --> CheckDirectionalMvFalse

pfSearchMethod --> WelsDiamondSearch

pfFirstIntraMode --> WelsMdFirstIntraMode

pfIntraFineMd --> WelsMdIntraFinePartitionVaa

pfInterFineMd --> WelsMdInterFinePartitionVaa