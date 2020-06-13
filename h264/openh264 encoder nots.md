# openh264 encoder nots

openh264编码的主函数是WelsEncoderEncodeExt
```
WelsEncoderEncodeExt
{
	pFbi->uiTimeStamp = GetTimestampForRc()//计算当前时间
	
	// perform csc/denoise/downsample/padding, generate spatial layers
	  iSpatialNum = pCtx->pVpp->BuildSpatialPicList (pCtx, pSrcPic);
	//预处理，产生空域分层
	
	pCtx->pFuncList->pfRc.pfWelsUpdateMaxBrWindowStatus()
	//更新最大的码率检测窗口，TODO
	
	InitBitStream (pCtx)//初始化码流
	
	if (!pSvcParam->bSimulcastAVC) 
	{
	    eFrameType = PrepareEncodeFrame (pCtx, pLayerBsInfo, iSpatialNum, iCurDid, iCurTid, iLayerNum, iFrameSize,
                                     pFbi->uiTimeStamp);
        ////检测当前的码率状态，并决定帧的类型
        if (eFrameType == videoFrameTypeSkip) 
        {
            pFbi->eFrameType = videoFrameTypeSkip;
            pLayerBsInfo->eFrameType = videoFrameTypeSkip;
            return ENC_RETURN_SUCCESS;
        }
	}
	    
	
	while (iSpatialIdx < iSpatialNum) 
	{
	...
        
        InitFrameCoding (pCtx, eFrameType, iCurDid);
        //初始化pParamInternal中的参数
        
        pCtx->pVpp->AnalyzeSpatialPic (pCtx, iCurDid);
        //预处理，场景切换检测和自适应qp计算
        
        WelsInitCurrentLayer();
        //初始化SDqLayer参数
        
        pCtx->pVpp->AnalyzePictureComplexity();
        //TODO
        
        pCtx->pFuncList->pfRc.pfWelsRcPictureInit();
        //调用WelsRcPictureInitGom，初始化
        
        PreprocessSliceCoding();
        //根据帧内容类型的不同，初始化不同的编码函数指针
        
        if (SM_SINGLE_SLICE == pParam->sSliceArgument.uiSliceMode)
        {
            SetSliceBoundaryInfo();//设置当前slice的大小
            WelsCodeOneSlice();//编码slice
            WelsEncodeNal();//写码流
        }
        else if (SM_SIZELIMITED_SLICE == pParam->sSliceArgument.uiSliceMode)
        {}
        else//other multi-slice uiSliceMode
        {}
        //根据不同的slice划分策略，执行不同的逻辑
        
        pCtx->pFuncList->pfRc.pfWelsRcPostFrameSkipping
        //编码结束，判断当前帧是否跳过。未实现
        
	}
	
	
}
```





openh264中提供动态和非动态的slice划分模式，动态划分是为了更好的兼顾网络丢包率高的情况。控制当前slice大小尽量接近但不超过传输层payload的大小。这样可以保证一个包一个slice，当前丢包不会影响后面slice的解码。<br>
slice编码函数设置
```
// 1st index: 0: for P pSlice; 1: for I pSlice;
// 2nd index: 0: for non-dynamic pSlice; 1: for dynamic I pSlice;
static const PWelsCodingSliceFunc g_pWelsSliceCoding[2][2] = {
  { WelsCodePSlice, WelsCodePOverDynamicSlice }, // P SSlice
  { WelsISliceMdEnc, WelsISliceMdEncDynamic }    // I SSlice
};
```
以WelsCodePSlice()为例，一个slice的编码主函数为
```
WelsMdInterMbLoop
{
...
    for (;;) // for every mb
    {
        //step(1): set QP for the current MB, 根据不同的码控模式有不同操作，通常是RcCalculateGomQp()
        pEncCtx->pFuncList->pfRc.pfWelsRcMbInit (pEncCtx, pCurMb, pSlice);
        
        //step(2). save some vale for future use, initial pWelsMd。初始化模式选择的相关变量
        WelsMdIntraInit (pEncCtx, pCurMb, pMbCache, kiSliceFirstMbXY);
        WelsMdInterInit (pEncCtx, pSlice, pCurMb, kiSliceFirstMbXY);
        
        //step(3). 进行具体的模式选择和编码。宏块编码的大部分工作在这个函数中完成，对于P slice是 WelsMdInterMb()
        pEncCtx->pFuncList->pfInterMd (pEncCtx, pMd, pSlice, pCurMb, pMbCache);
        
        ...
        
        //step(8): 更新码率控制 gom。统计每个宏块编码后的cost和bits
        pEncCtx->pFuncList->pfRc.pfWelsRcMbInfoUpdate (pEncCtx, pCurMb, pMd->iCostLuma, pSlice)
    }
}
```

"Md" 指的应该是mode decision，决定宏块类型。在这中间会加入一些编码前处理的信息对编码过程的优化

pfInterMd() 即WelsMdInterMb() 宏块模式决定和编码函数