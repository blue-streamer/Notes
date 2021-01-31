在屏幕编码的场景下（iUsageType = SCREEN_CONTENT_REAL_TIME），块的屏幕运动估计使用了与video不同的策略。它综合使用了diamond search, cross search 和feature search。其中feature search是根据哈希匹配的运动搜索。整体流程如下：

### 1、对每个8x8的块，确定运动类型

函数入口：DetectSceneChange

每个8x8的块定义了3种类型：

```c++
typedef enum {
 NO_STATIC,  // motion block
 COLLOCATED_STATIC, // collocated static block
 SCROLLED_STATIC,  // scrolled static block
 BLOCK_STATIC_IDC_ALL
} EStaticBlockIdc;
```

NO_STATIC: 运动块

COLLOCATED_STATIC: 静止块

SCROLLED_STATIC: 滚动静止块

在一帧编码的开始会首先进行预处理，在预处理的流程中会完成每个块类型的确定：

![img](https://bytedance.feishu.cn/space/api/box/stream/download/asynccode/?code=3227f5d211dca9818646af1ea051c0b1_8f118824ce50c961_boxcnymMvmiWat9GCpqBgpAxFih_VU1Brm6ee3wczT7yj0V2KQpSQztiqXya)

在DetectSceneChange中首先做滚动检测，然后根据滚动检测的结果进行场景变化检测。这SceneChangeDetection中确定了8x8块的类型。计算方式比较简单，计算sad然后比较：

- **COLLOCATED_STATIC**：与参考帧完全一致

- **SCROLLED_STATIC**：加上滚动mv，与参考帧完全一致

- **NO_STATIC**：不符合上面两种情况

具体计算在SceneShangeDetection.h中



### 2、vaa计算

函数入口：VaaCalculation

计算每个8x8块的sad



### 3、选择搜索方法，计算feature(哈希)

函数入口：PreprocessSliceCoding

这里一共有3步：

- **设置8x8类型对应的运动搜索的函数**

```c++
pFuncList->pfMotionSearch[NO_STATIC] = WelsMotionEstimateSearch;
pFuncList->pfMotionSearch[COLLOCATED_STATIC] = WelsMotionEstimateSearchStatic;
pFuncList->pfMotionSearch[SCROLLED_STATIC] = WelsMotionEstimateSearchScrolled;
```

- 计算feature

针对选定的参考帧，计算其中8x8块的feature。在PerformFMEPreprocess()函数中完成，主要相关的数据结构为：

```c++
typedef struct TagScreenBlockFeatureStorage {
//Input
uint16_t*  pFeatureOfBlockPointer;   // Pointer to pFeatureOfBlock
int32_t   iIs16x16;    //Feature block size
uint8_t   uiFeatureStrategyIndex;// index of hash strategy
//Modify
uint32_t*  pTimesOfFeatureValue;   // times of every value in Feature
uint16_t**
pLocationOfFeature;    // uint16_t *pLocationOfFeature[LIST_SIZE], pLocationOfFeature[i] saves all the location(x,y) whose Feature = i;
uint16_t*  pLocationPointer;  // buffer of position array
int32_t   iActualListSize;  // actual list size
uint32_t uiSadCostThreshold[BLOCK_SIZE_ALL];
bool    bRefBlockFeatureCalculated; // flag of whether pre-process is done
uint16_t **pFeatureValuePointerList;//uint16_t* pFeatureValuePointerList[WELS_MAX (LIST_SIZE_SUM_16x16, LIST_SIZE_MSE_16x16)]
} SScreenBlockFeatureStorage; //should be stored with RefPic, one for each frame
```

**pFeatureOfBlockPointer**：存储每个8x8块的feature值

**pTimesOfFeatureValue**：每个feature值出现的次数

**pFeatureValuePointerList**：二维数组，每个feature值出现的像素位置，1/4精度单位

- 设置16x16，8x8的搜索方法

在screen的场景下，p块只有两种模式，16x16和8x8。分别设定他们的运动搜索方法

```c++
SetMeMethod (ME_DIA_CROSS, pFuncList->pfSearchMethod[BLOCK_16x16])
SetMeMethod (ME_DIA_CROSS_FME, pFuncList->pfSearchMethod[BLOCK_8x8])
```

其中

16x16使用**WelsDiamondCrossSearch**，首先进行diamond search，然后进行cross search。

8x8使用**WelsDiamondCrossFeatureSearch**，首先进行diamond search，然后进行cross search，最后使用feature search。



### 4、模式选择和预测

入口函数：WelsMdInterMb

- 计算I16x16和P16x16的BestCost，p16x16使用diamond-cross search

- 根据vaa中每个宏块的8x8块sad的均值和方差来决定是否提前终止

```c++
uint8_t uiMbSign = pEncCtx->pFuncList->pfGetMbSignFromInterVaa (&pEncCtx->pVaa->sVaaCalcInfo.pSad8x8[pCurMb->iMbXY][0]);
if (MBVAASIGN_FLAT == uiMbSign) {
 return;
}
```

- 8x8运动搜索

根据每个8x8块的类型，调用不同的运动搜索的方法找到最佳的配置块

WelsMotionEstimateSearch：正常运动搜索，调用WelsDiamondCrossFeatureSearch

WelsMotionEstimateSearchStatic：直接计算cost

WelsMotionEstimateSearchScrolled：加上scrolled mv，计算cost