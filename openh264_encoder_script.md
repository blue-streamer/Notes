encoder中使用了很多函数指针，在InitFunctionPointers函数中初始化。下面记录一下WelsCodePSlice中用到的函数指针：

pfInterMdBackgroundDecision --> WelsMdInterJudgeBGDPskipFalse

pfSCDPSkipDecision --> WelsMdInterJudgeSCDPskipFalse

sSampleDealingFuncs.pfSampleSad --> WelsSampleSad16x16_c_sse

pfCheckDirectionalMv --> CheckDirectionalMvFalse

pfSearchMethod --> WelsDiamondSearch

pfFirstIntraMode --> WelsMdFirstIntraMode

pfIntraFineMd --> WelsMdIntraFinePartitionVaa

inter

pfInterMd --> WelsMdInterMb 宏块模式决定和编码函数

pfInterFineMd --> WelsMdInterFinePartitionVaa

pfMotionSearch --> WelsMotionEstimateSearch

pfSearchMethod --> WelsDiamondSearch

sSampleDealingFuncs.pfSampleSad[16x16] --> WelsSampleSad16x16_c_sse



