```c++
pCtx->ppRefPicListExt[pCtx->uiDependencyId];
```

在encoder_context中，每一个空域的层都有不同的参考帧列表

```c++
typedef struct TagRefList {
  SPicture*     pShortRefList[1 + MAX_SHORT_REF_COUNT]; // reference list 0 - int16_t
  SPicture*     pLongRefList[1 + MAX_REF_PIC_COUNT];    // reference list 1 - int32_t
  SPicture*     pNextBuffer;
  SPicture*     pRef[1 + MAX_REF_PIC_COUNT];    // plus 1 for swap intend
  uint8_t       uiShortRefCount;
  uint8_t       uiLongRefCount; // dependend on pRef pic module
} SRefList;

#define MAX_SHORT_REF_COUNT             (MAX_GOP_SIZE>>1)
#define MAX_GOP_SIZE    (1<<(MAX_TEMPORAL_LEVEL-1))
#define MAX_TEMPORAL_LEVEL              MAX_TEMPORAL_LAYER_NUM
#define MAX_TEMPORAL_LAYER_NUM          4
```

短期参考帧的最大个数为4，和时域层数相关。在一个时域分层的周期中，当前帧只能参考最近的一个0层帧之后的帧。遇到一个零层的帧，短期参考帧列表只会保留一个帧

```c++
bool WelsUpdateRefList (sWelsEncCtx* pCtx) {
...
if (keSliceType == P_SLICE) {
    if (pCtx->uiTemporalId == 0) {
    ...
      for (i = pRefList->uiShortRefCount - 1; i > 0; i--) {
          pRefList->pShortRefList[i]->SetUnref();
          DeleteSTRFromShortList (pCtx, i);
        }
      if (pRefList->uiShortRefCount > 0 && (pRefList->pShortRefList[0]->uiTemporalId > 0
                                            || pRefList->pShortRefList[0]->iFrameNum != pParamD->iFrameNum)) 				{
        pRefList->pShortRefList[0]->SetUnref();
        DeleteSTRFromShortList (pCtx, 0);
      }
    }
	}
}
```

目前openh264的screen模式只支持一个参考帧。在preprocess中有参考帧的参考帧选择机制，但是在ref list的管理仅仅使用了简单的划窗，使用最近的一帧作为参考帧。当ref list和preprocess 中的best ref不一致时，以ref list为准并重新计算blockidc

```c++
if (pCtx->eSliceType != I_SLICE) {
      pCtx->pReferenceStrategy->AfterBuildRefList();//检查是否一致，更新blockidx
```

