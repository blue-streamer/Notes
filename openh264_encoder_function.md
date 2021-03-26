encoder中使用了很多函数指针，在InitFunctionPointers函数中初始化。下面记录一下WelsCodePSlice中用到的函数指针：

pfInterMdBackgroundDecision --> WelsMdInterJudgeBGDPskipFalse

pfSCDPSkipDecision --> WelsMdInterJudgeSCDPskipFalse

sSampleDealingFuncs.pfSampleSad --> WelsSampleSad16x16_c_sse

pfCheckDirectionalMv --> CheckDirectionalMvFalse/CheckDirectionalMv

pfSearchMethod --> WelsDiamondSearch

pfFirstIntraMode --> WelsMdFirstIntraMode/WelsMdScreenIntraMode

pfIntraFineMd --> WelsMdIntraFinePartitionVaa

IntraMbPtr --> WelsMdIntraMbForScreen

IntraMbPtr --> WelsMdIntraMb



WelsMdInterMbLoop

pfInterMd --> WelsMdInterMb 宏块模式决定和编码函数

WelsMdInterSecondaryModesEnc

pfInterFineMd --> WelsMdInterFinePartitionVaa

pfInterFineMd --> WelsMdInterFinePartitionVaaOnScreen

pfMotionSearch --> WelsMotionEstimateSearch

pfSearchMethod --> WelsDiamondSearch

pfSCDPSkipDecision --> WelsMdInterJudgeSCDPskip

sSampleDealingFuncs.pfSampleSad[16x16] --> WelsSampleSad16x16_c_sse

pfWelsSpatialWriteMbSyn --> WelsSpatialWriteMbSyn/WelsSpatialWriteMbSynCabac

pfSetScrollingMv --> SetScrollingMvToMd

pfUpdateMbMv --> UpdateMbMv_c

pfMdBackgroundInfoUpdate --> WelsMdUpdateBGDInfo

pfFillInterNeighborCache --> FillNeighborCacheInterWithoutBGD

Hash

pfCalculateBlockFeatureOfFrame --> SumOf8x8BlockOfFrame_c

pfInitializeHashforFeature --> InitializeHashforFeature_c

pfFillQpelLocationByFeatureValue --> FillQpelLocationByFeatureValue_c

pfCalFirstHash --> CalFirstHash_v2_c

pfCalAHash --> CalAHash

pfVerticalFullSearch/pfHorizontalFullSearch --> LineFullSearch_c

transform

pfQuantizationFour4x4Max --> WelsQuantFour4x4Max_c

pfScan4x4 --> WelsScan4x4DcAc_c

pfScan4x4Ac --> WelsScan4x4Ac_c

pfCalculateSingleCtr4x4 --> WelsCalculateSingleCtr4x4_c

pfDctFourT4 --> WelsDctFourT4_c

pfQuantizationHadamard2x2Skip --> WelsHadamardQuant2x2Skip_c

pfQuantizationHadamard2x2 --> WelsHadamardQuant2x2_c

pfIDctFourT4 --> WelsIDctFourT4Rec_c



Rate control

pRcf->pfWelsRcPictureInit = WelsRcPictureInitGom;
pRcf->pfWelsRcPictureInfoUpdate = WelsRcPictureInfoUpdateGomTimeStamp;
pRcf->pfWelsRcMbInit = WelsRcMbInitGom;
pRcf->pfWelsRcMbInfoUpdate = WelsRcMbInfoUpdateGom;

pRcf->pfWelsRcPicDelayJudge = WelsRcFrameDelayJudgeTimeStamp;
pRcf->pfWelsCheckSkipBasedMaxbr = NULL;
pRcf->pfWelsUpdateBufferWhenSkip = NULL;
pRcf->pfWelsUpdateMaxBrWindowStatus = UpdateMaxBrCheckWindowStatusTimeStamp;
pRcf->pfWelsRcPostFrameSkipping = NULL;





















