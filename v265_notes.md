## DataStruct

### TCtuInfo

ctu 中cu通过链表管理，头结点不存储有效信息cuHead，pCuTail指向当前cu，尾插更新链表。pCuTail初始化为&cuHead。这个链表的顺序是按照划分决策之后的zigzag遍历顺序

cu_group中是预分配好的cu，85个（1+4+16+64），并且分配到cu->subcu中，已完成初始化

iEearlyStop[4] 存储每个depth是否skip

pCtuStoreCurr，存储每个深度的划分情况和划分的cost

pNbor，存储附近block的情况，按照4x4block为单位存储信息

### TCodingUnit

TPredUnit* pu[8]，每个元素都是数组首地址，代表了8中pu的划分方式。2nx2n有一个元素，nxn有4个元素

pMdRt，存放最佳md result，整个md结束之后其中就已经存在最佳模式的 coeff

pTmpMdRt，存放当前正在尝试的md result

### TPredUnit

iInterDir，预测方向。1：前向预测，2：后向预测，3：双向预测

### TLowerResPic

与输入图像对应，存放某些预分析的结果。

pData，存放一张1/2下采样的图像，scc的情况下会采样点，非scc进行插值计算

iSccAttr，scc相关的分析结果，1/2图像上8x8block为单位

iForgroundAttr，前景分析结果，以ctu为单位，1/2图像上以32x32为单位。计算当前图像和参考图像相同位置的sad，大于阈值判定为前景。参考函数isYUVSimiliarYPlane()

## EncodeProcedure

### overall 

CByteVCEncode::encodeFrame函数是一帧的编码入口

```c++
int CByteVCEncode::encodeFrame(){
  TInputPic* pic = m_inputPicManage[bAlphaChannel]->onNewInputPic(pInpic, &DPBForInputPic);
  //new frame incoming, 转换为TInputPic，并放入queue
  m_PreAnalyzeTaskManager->executeTasks(pic);//前处理
  while ((inPic = m_inputPicManage[bAlphaChannel]->getPicTobeEncode(pInpic)) != NULL) {//update queue，rc分析，lookahead参考帧决策等。最后取出待编码帧
		m_taskManage->executeTasks(frameInfo);//编码
  	onFrameFinish(frameInfo);//写码流
  }
}
```

### preprocess

在进行编码之前会有一个预分析的阶段，分析的结果对后续的编码过程有指导意义。预分析相关的数据结构主要是TLowerResPic，相关函数：

```c++
initLowerResPic();//初始化LowerResPic，下采样图像
calcScreenAttrib();//计算scc相关属性
```

scc相关的属性有两个。plane：平坦区域，content：文字区域。这些结果在1/2图像上8x8block为单位进行计算。

- plane

  水平竖直相差一个像素计算sad，其中一个sad<64即为plane

- scc

  统计block像素的直方图，通过像素个数，最大最小像素差距，top5像素个数和等条件得到。参考函数isSccBlock()

如果一个block为plane或者scc，则定义为sccCu。如果一张图像sccCu的个数大于阈值，则设置bSccFrame。此时会关闭deblock filter

### processOneCtu

编码ctu，完成md，trans，quant，filter，writeBs所有流程。整帧的编码是在一个循环中调用这个函数

```c++
void processOneCtu(TAddr* addr){
	initCtu();//初始化，预处理hash
	processCtuMd();//md,trans,quant
  m_loopFilter->Execute();//filter
	m_pSbac->processCtuSbac();// write bitstream
}
```

在hevc中有不同tu的选择，并且精确的md过程需要使用变换量化过的系数来估计Rcost。因此，最佳模式变换量化都在md中进行了。processCtuMd就是调用了processTree完成了整个ctu的md

### processCtuMd

ctuMd过程，调用processTree。按照raster顺序访问的cu都记录在链表中

```c++
void processCtuMd(TCtuInfo* ctu){
	ctu->pCuTail = &ctu->cuHead;// 初始化链表
  processTree(ctu, ctu->cu_group);// 四叉树递归md
  ctu->pCuTail->next = NULL;// 结束链表
}
```

### ProcessTree

这是一个递归调用的函数，用来决策ctu的划分情况。主体流程如下：

```c++
uint32_t processTree(TCtuInfo* ctu, TCodingUnit* const cu)
{
  initCuOnMdStart();// init cu
  if (cu->subCu[0]){
		sub_cost = processTree(ctu, cu->subCu[0]);
		sub_cost += processTree(ctu, cu->subCu[1]);
		sub_cost += processTree(ctu, cu->subCu[2]);
		sub_cost += processTree(ctu, cu->subCu[3]);
  }
	cost = pfProcessCuMd();
  
  if(sub_cost < cost){
    cost = sub_cost;
    updateSplitParam(ctu, cu, sub_cost);
  }else if (cost < MAX_UNIT_COST){
    updateNonSplitParam(ctu, cu, sub_cost);
  }
  cu->uiBestRDCost = cost;
  return cost;
}
```

可以看到首先递归划分到最小的8x8size，然后进行Md，计算cost。再递归返回向上，比较cost与sub_cost。来决策是否进行划分。为了实现不同快速档位，添加了很多参数和快速判别逻辑。

其中有几个重要的bool量，

- **bFromUp2Down**：常规是首先得到4个subCu的cost，然后计算curCu的cost。进行比较。如果设置为true，则会首先计算curCu的cost，然后再subCu。这个逻辑中会有快速跳过subPart的逻辑，
- **tryGoDown**：是否尝试4个subCu
- **tryGoUp**：是否尝试curCu

#### pfEarlySkip

主要check merge mode，如果返回true，则跳过subCu和curCuMd。已经得到bestMode。在非CHECK_MERGE_FULL的情况下，使用**skipFastDecision**函数。

1. 首先会检测globalPalette，应该是有两个globalPalette，每个palette的yuv是都是单色

   ```c++
   if (cu->bIntraPlainCu && ctu->encParam->bSccGloblPalettes && !bRepeatPeriodSkipMode) {
     bEarlySkip = checkGloblPaletteMode(ctu, cu);// check global palette
   
     if (bEarlySkip) {// global palette match!
       uiMinCost = cu->uiBestRDCost;
     }
     fillTmpMdRtForInter(cu, SIZE_2Nx2N);
   }
   ```

2. 其次遍历merge candidate，寻找最小的cost

   ```c++
   ctu->pfGetMergeCands(ctu, pu, ctu->encParam, ctu->frame);// get all merge candidate
   for (pu->iMergeIdx = 0; pu->iMergeIdx < pu->iNumMergeCands; ++pu->iMergeIdx) {
     interpolatePuLuma(pRef[0], ctu->frame, pu, ctu->cache);// MC
   	uiYCost = g_sse_Function[size](cu->pSrc[YUV_Y], pRef[0], MAX_CTB_SIZE, MAX_CTB_SIZE, (1 << cu->iLog2Size));// calc luma cost
     uiSkipCost = uiYCost;
     if (uiSkipCost < uiMinCost) {
       uiMinCost = uiSkipCost;
   	  iBestSkipIdx = pu->iMergeIdx;
   	  switchBestMode(cu, uiSkipCost);
     }
     //early skip of merge cand selection
     if (uiMinYCost < uiDCostTh) {// 开启提前跳出时，uiDCostTh是非零
       bLumaEarlySkip = true;
       break;
     }
   }
   ```

3. 决策skip

   如果globalPalette更好，直接return。否则，check最佳的merge candidate。check的方法是分块计算diff和dct，check系数的最小值与threshold。

   ```c++
   //skip mode is not better than global palette mode, just return
   if (bEarlySkip && (iBestSkipIdx == -1)) {
   	return bEarlySkip;
   }
   pfEarlyskipCheck pEarlySkipCheckF = earlyskipCheck;// check function
   int32_t skipped = pEarlySkipCheckF(YUV_Y);// check Y
   if(!cu->monoChroma && skipped){
     skipped = pEarlySkipCheckF(YUV_U);// check U
   }
   if(!cu->monoChroma && skipped){
     skipped = pEarlySkipCheckF(YUV_V);// check V
   }
   ```

   check的方法对YUV分量分别进行。分块计算diff和dct，check系数的最小值与threshold。**注：这里似乎没有与qp相关，检查残差系数量化之后的情况更合理**

#### 快速决策参数

1. **enSkipCU16Above**

   跳过64和32的cu size，如果depth<2 则不会进行curCuMd，直接划分

2. **enUp2DownJudgeByMaxDepth**

   通过depth的判断来决定是否使用Up2Down的决策逻辑

3. **iTreePartFast**

   

Todo:

pfEarlySkip

storeCodingParam/storeNborInfo

cu的深度预测是通过left，top，topRight，topLeft的深度加权预测得到的，3：3：2：2

#### tCuSplitInfo

在processTree的流程中首先会判断使用up2Down，其中**tCuSplitInfo**中的**enFastGoDown**的统计是进行判断的一个依据

如果使用up2Down：首先计算curCu的md，然后计算subCu的md。中间会有跳过策略

如果使用down2Up：首先计算subCu的md，然后判断使用goUp。其中**tCuSplitInfo**中的**enFastGoUpByCost**的统计值是进行判断的一个依据

tCuSplitInfo在一个slice中随着ctu的编码一直进行统计，会统计不同深度的splitCount和nonSplitCount。然后在当前ctu编码完成后计算出来，以便下一个ctu使用。

**enFastGoDown**：如果nonSplit >= (7/4)split，则设为true

**enFastGoUpByCost**：如果split >= (5/4)nonSplit，则设置为true

### processCuMdInter

checkMerge2Nx2N

checkInterPu2Nx2N

checkInterPu

may ibc

may rdoq

intraMD

#### meInitPoint

mvInitList：zeroMv，hashSearchMv，subCuMvAndparentCuMv or HistMVList，pre-analysisMv

chooseBestMvInitPoint：pu->mvpCands只有两个？

#### motionSearchOneRef

```c++
meInitPoint();
IntMe();
SubMe();
reselectMVP();
```



