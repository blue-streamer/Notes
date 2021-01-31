openh264 中提供时域分层的功能。每一帧都有一个对应的时域层，一个视频序列可以分为最多4层。丢掉上面的层，不影响下一层的正常解码和播放。这样在编码完成之后可以实现不解码而直接调整视频帧率的效果，有利于弱网情况下的视频实时传输。

## 时域分层结构

时域分层的结构如下，以3层为例：

![temporal_layer](temporal layer in openh264.assets/temporal_layer-5484533.png)

### GOP

​	时域分层中有一个gop的概念，这个gop与普通意义中的gop不同。时域分层的帧是一种周期性的结构，属于同一个周期的帧组成一个gop。如上图，coding index 0~3 是第一个gop，4~7 是第二个gop。

### Temporal layer

​	开启时域分层之后，每一帧都会有一个temporal layer的概念。属于同层的帧可以在传输的过程中，统一丢弃。不影响低于这个一层的所有帧的解码。以3层为例：丢掉第2层，1层和0层可以正常解码。丢掉第2层和第1层，0层可以正常解码。

​	时域分层的一个gop中，每一层的帧数等于低于这一层的所有层的帧数之和。例如，4层时域分层，帧数比例为4:2:1:1。3层时域分层，帧数比例为2:1:1。这样，丢掉一层之后，帧率变为原来的1/2。

### 参考关系与frame num

​	上图中，每个帧上表明了frame num。根据h264标准，当前frame num等于前一个参考帧的frame num+1。因此frame num如图所示。确定参考帧的规则为：**小于等于当前层的帧中最近的一个参考帧**。参考队列的调整通过slice head中的reorde来通知解码器。



## 相关代码

​	在openh264中，有时域分层和空域分层两个概念。空域分层有两种实现范式，svc和avc。目前encoder没有使用openh264中的空域分层。spatial layer为1，设置simulcastAVC为false，即使用单层的svc。这样和标准的h264没有区别。

​	代码中的sDependencyLayers和iCurDid指空域分层，TemporalLevel和iCurTid指时域分层。时域分层和参考队列操作主要流程如下：

- PrepareEncoderFrame

  决定帧类型，获取当前帧的temporal layer。代码如下
  
  ```c++
  int32_t GetTemporalLevel (SSpatialLayerInternal* fDlp, const int32_t kiFrameNum, const int32_t kiGopSize) {
    const int32_t kiCodingIdx = kiFrameNum & (kiGopSize - 1);
  
    return fDlp->uiCodingIdx2TemporalId[kiCodingIdx];
  }
  ```
  
  kiFrameNume是一个递增的coding index，从idr帧开始。gopsize和uiCodingIdx2TemporalId是初始化时设定好的。对gop取余，计算出当前帧在gop中的index，进而计算出temporal layer。

- InitFrameCoding

  更新poc和frame num。poc使用简单的递增+2的策略，frame num根据上一帧的priotity来决定是否递增。

  ```c++
  void UpdateFrameNum (sWelsEncCtx* pEncCtx, const int32_t kiDidx) {
    SSpatialLayerInternal* pParamInternal = &pEncCtx->pSvcParam->sDependencyLayers[kiDidx];
    bool bNeedFrameNumIncreasing = false;
  
    if (NRI_PRI_LOWEST != pEncCtx->eLastNalPriority[kiDidx]) {
      bNeedFrameNumIncreasing = true;
    }
  
    if (bNeedFrameNumIncreasing) {
      if (pParamInternal->iFrameNum < (1 << pEncCtx->pSps->uiLog2MaxFrameNum) - 1)
        ++ pParamInternal->iFrameNum;
      else
        pParamInternal->iFrameNum = 0;    // if iFrameNum overflow
    }
  
    pEncCtx->eLastNalPriority[kiDidx] = NRI_PRI_LOWEST;
  }
  ```

  eLastNalPriority表征了上一帧是否为参考帧，来调整frame num。Tid会决定NalRefIdc：

  ```c++
  if (iCurTid == 0 || pCtx->eSliceType == I_SLICE)
        eNalRefIdc = NRI_PRI_HIGHEST;
      else if (iCurTid == iDecompositionStages)
        eNalRefIdc = NRI_PRI_LOWEST;
      else if (1 + iCurTid == iDecompositionStages)
        eNalRefIdc = NRI_PRI_LOW;
      else 
        eNalRefIdc = NRI_PRI_HIGHEST;
  ```

- BuildRefList

  构建当前帧的参考列表，ctx中有一个IWelsReferenceStrategy的抽象类，不同的派生类会有不同的策略。在屏幕共享中使用CWelsReference_Screen。WelsBuildRefList中时域分层相关的主要是短期参考帧相关的

  ```c++
  for (i = 0; i < pRefList->uiShortRefCount; ++ i) {
          SPicture* pRef = pRefList->pShortRefList[i];
          if (pRef != NULL && pRef->bUsedAsRef && pRef->iFramePoc >= 0 && pRef->uiTemporalId <= kuiTid) {
            pCtx->pCurDqLayer->pRefOri[pCtx->iNumRef0] = pRef;
            pCtx->pRefList0[pCtx->iNumRef0++] = pRef;
            WelsLog (& (pCtx->sLogCtx), WELS_LOG_DETAIL,
                     "WelsBuildRefList pCtx->uiTemporalId = %d,pRef->iFrameNum = %d,pRef->uiTemporalId = 										%d",pCtx->uiTemporalId, pRef->iFrameNum, pRef->uiTemporalId);
            break;
          }
  ```

- UpdateRefList

  当前帧重构之后，更新reflist

  ```c++
  if (eNalRefIdc != NRI_PRI_LOWEST) {
        if (!pCtx->pReferenceStrategy->UpdateRefList()) {
          WelsLog (pLogCtx, WELS_LOG_WARNING, "WelsEncoderEncodeExt(), WelsUpdateRefList failed. ForceCodingIDR!");
          pCtx->iEncoderError = ENC_RETURN_CORRECTED;
          break;
        }
      }
  ```

  在updateRefList中主要会有两个操作，更新参考帧列表和更新对应的原图列表
  
  更新参考帧列表：
  
  ```c++
  for (iRefIdx = pRefList->uiShortRefCount - 1; iRefIdx >= 0; --iRefIdx) {//把当前重构帧放在最前面
  	pRefList->pShortRefList[iRefIdx + 1] = pRefList->pShortRefList[iRefIdx];
  }
  pRefList->pShortRefList[0] = pCtx->pDecPic;
  pRefList->uiShortRefCount++;
  ...
  if (keSliceType == P_SLICE) {
      if (pCtx->uiTemporalId == 0) {
        if (pCtx->pSvcParam->bEnableLongTermReference) {
          ...
        }
        for (i = pRefList->uiShortRefCount - 1; i > 0; i--) {//如果是0层，只留下当前帧，其余reset
          pRefList->pShortRefList[i]->SetUnref();
          DeleteSTRFromShortList (pCtx, i);
        }
        ...
      }
    }
  
  ```
  
  更新对应的原图列表
  
  ```c++
  void CWelsPreProcess::UpdateSrcList (SPicture* pCurPicture, const int32_t kiCurDid, SPicture** pShortRefList,
                                       const uint32_t kuiShortRefCount) {
    SPicture** pRefSrcList = &m_pSpatialPic[kiCurDid][0];
  
    //pRefSrcList[0] is for current frame
    if (pCurPicture->bUsedAsRef || pCurPicture->bIsLongRef) {
      if (pCurPicture->iPictureType == P_SLICE && pCurPicture->uiTemporalId != 0) {
        for (int iRefIdx = kuiShortRefCount - 1; iRefIdx >= 0; --iRefIdx) {//put an unuse frame at first place
          WelsExchangeSpatialPictures (&pRefSrcList[iRefIdx + 1],
                                       &pRefSrcList[iRefIdx]);
        }
        m_iAvaliableRefInSpatialPicList = kuiShortRefCount;// src list is align with ref list
      } else {
        WelsExchangeSpatialPictures (&pRefSrcList[0], &pRefSrcList[1]);
        for (int32_t i = MAX_SHORT_REF_COUNT - 1; i > 0  ; --i) {
          if (pRefSrcList[i + 1] != NULL) {
            pRefSrcList[i + 1]->SetUnref();
          }
        }
        m_iAvaliableRefInSpatialPicList = 1;
      }
    }
    (GetCurrentOrigFrame (kiCurDid))->SetUnref();// set next src pic slot unref
  }
  ```
  
  

## 时域分层和多参考帧

在openh264中有两个list，srcList和refList

srcList：前处理中使用，存放原始帧。对应**CWelsPreProcess.m_pSpatialPic**，初始化大小为2 + WELS_MAX (iHighestTemporalId, 1) （AllocSpatialPictures）

refList：编码使用，存放重建帧。对应**SRefList.pShortRefList**，初始化大小为iMaxNumRefFrame+1（InitDqLayers）

由于时域分层的存在，因此所有的参考关系都会满足时域分层的要求。refList的大小会保证时域分层的参考关系得到满足

```c++
if (pCfg->iNumRefFrame == AUTO_REF_PIC_COUNT)
        pCfg->iNumRefFrame = WELS_MAX (1, pCfg->uiGopSize >> 1);
```

如果强行设置过小的RefFrame，会导致时域分层的参考关系失败。可以设置大于gop/2的参考帧的长度，但是，并没有更复杂的参考帧选择算法。

在一个gop中，会遍历所以符合时域分层要求的参考帧，计算每个符合要求的帧与当前帧的差别，选择最好的。这个过程在DetectSceneChange中完成（GetAvailableRefList），使用srcList的原始帧。而gop之间，0层帧使用了简单的策略，只参考前一个0层的帧。当一个0层的帧编码完成之后，会清空refList（WelsUpdateRefList）

DetectSceneChange选择完参考帧会记录下来（SaveBestRefToVaa）。refList寻找参考帧的方法很简单，满足时域分层参考关系中最近的一个帧（BuildRefList）。当refList和srcList选择的参考帧不一致时，会以refList为准（UpdateBlockStatic）。因此，srcList的算法并没有被用到！