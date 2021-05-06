AppEncTop::encodeOneSeq

CByteVCEncode::encodeFrame

### Encoder entrance 

CCtuEncTask::execute

CCtuEncTaskWpp::execute

### Ctu entrance

CCtuEnc::processOneCtu

### Write bitstream

CCtuSbac::processCtuSbac

### function pointer

initSCCFunctions:

g_calcSccUniform2x2_func = calcUniformgenius2x2;

g_calcSccUniformRow_func = calcUniformRow;

g_calcSccUniformCol_func = calcUniformCol;

g_calcSccUniformMask_func = calcUniformMask;

g_fastCrcNormal8x8_func = fastcrc32_c;

g_getHashIdx_func = getHashIdx



pfProcessCuMd --> processCuMdIntra/processCuMdInter

pfMotionSearch --> motionSearchP/motionSearchB

pfEarlySkip --> skipFastDecision

pfGetMergeCands --> GetMergeCandsForP/GetMergeMvpCandsForP_SingleRef

Ibc:

processCuMdIbc

pfIbcSearch --> motionSearchIbc

Palette: 

processCuMdPalette

pfGeneratorFromSource --> palette_generator_from_source

g_calcSSD_2D_diffUV --> calcSADForPaletteI_core

g_calcSSD_2D --> calcBestIndicesAndSSD_Core

g_checkEscape --> checkEscape

Intra:

pfDecideLumaModeBySad --> decideBestLumaModeBySadFast

Sse:

g_sse_range_Function --> sse_range_c

g_sse_Function --> sse_c



### param

屏幕内容编码的配置参数为：Preset--**ByteVC1PRESET_ULTRAFAST**，Usecase--**ByteVC1USECASE_SCC**。首先调用fillDefaultCfgs配置搜索参数，然后使用preset和usecase来修改参数

enSkipCU16Above, 0: normal processing, 1: enable skip16 processing，skip 64x64 and 32x32 cu size(0 and 1 depth)

iEarlySkipCheckCUD, 0:disable; 1b(flag) + 1b(TL0) + temporal_level: enable minimal cu depth restriction for early skip. b0: enable or disable

bOnlySkipForLargeCU: false, only check skip mode for large CU sizes

enUp2DownJudgeByMaxDepth: true

enMergeTuDecision: false, enable merge tu decision

bFastInit: false, fast Ctu/Cu initialization

iMergSADTh: 0, threshold of mergeSAD for fast choosing merge as the better mode rather than inter, and skipping ME

enUp2DownJudgeByMaxDepth: 1

iTreePartFast: 0, 0:disable; 1: enable fast subPart; 2: enable fast encoding for current cu according to cu depth; 3: both. bit0: fast subPart，bit1: ，bit2: 64x64size special. 这里的1,2,3应该是1,2,4。bit0,bit1,bit2设置为1。"fast subPart" 通过一个subPart的cost来判断

cuGoUpLevel: 0, 0: veryfast; 1: medium; 2: full; (< 2: enable fast do bigger cu when going down to up)

cuGoDownLevel：0，

subCuEarlySkipRatioX/subCuEarlySkipRatioDownToUpX：快速subPart使用，X=1,2,3。每次得到subCu的cost之后，使用判断是否停止尝试subCu

meInitMVDist，初始化mv的距离阈值，插入一个新的mv到mvInitPointList中时，需要检查当前mv和list中已有的mv的distance。如果小于这个阈值不插入

enCbp：EnContentBasedPrune，enable content-based pruning for Nx2N/2NxN PU shapes

enInterAmp：AsymmetricMotionPartition

enBitCnt：rdo中是否使用bitcnt来计算Rcost

iEarlySkipMode：0

fastGoDown:0，whether to enable fast go down method

enFastDoIntraJudge:true

intraRmdLevel：INTRARMD_LEVEL_FAST

enSimpIntraMD：false

enIntra8x8Use4x4Modes：false

lessIntraModes：false

enFastDoIntraJudge：true

iInterSadFactor：19