SceneChangeDetection:

```c++
typedef struct {//进行场景切换检测的参数
  int32_t iWidth;
  int32_t iHeight;
  int32_t iBlock8x8Width;
  int32_t iBlock8x8Height;
  uint8_t* pRefY;
  uint8_t* pCurY;
  int32_t iRefStride;
  int32_t iCurStride;
  uint8_t* pStaticBlockIdc;//外部传入，预先分配好的内存，存放每个8x8块的 static idc
} SLocalParam;
```

SMbCache:

在一个slice的编码中，会有一个mb loop（WelsMdInterMbLoop），这个loop中每个mb的编码都有用到MbCache，mb编码前初始化MbCache。其中的

```c++
typedef struct TagMbCache {
  struct {
  /* pointer of current mb location in original frame */
  uint8_t* pEncMb[3];//指向编码图像当前mb的位置
  /* pointer of current mb location in recovery frame */
  uint8_t* pDecMb[3];
  /* pointer of co-located mb location in reference frame */
  uint8_t* pRefMb[3];//执行参考图像当前mb的位置
  //for SVC
  uint8_t*      pCsMb[3];//locating current mb's CS in whole frame
//              int16_t *p_rs[3];//locating current mb's RS     in whole frame

} SPicData;
}SMbCache;
```



skip 判别

- 获取当前宏块ABCD位置宏块是否是skip，如果有一个是skip则trySkip设置为true；如果ABC均为skip，则keepSkip设置为true
- 调用"WelsMdInterJudgePskip"函数进行判别
  - 计算当前宏块skip sad的预测值，与mvp计算类似。这里的sad是yuv三通道的sad。
  - 计算当前宏块的sad。
  - 如果sad=0 || sad<sadPred || sad< ref pic对应位置skip sad(如果对应位置是skip)。则判定为skip
  - 否则，进行dct计算，通过dct系数再次判别是可以是skip。
  - 返回bSkip标志
- 如果 bSkip && keepSkip == true，则直接进行skip的编码，结束
- 否则，try intra mode，使用iCostLuma判定使用skip 还是 intra

另外一种情况

如果p16x16宏块的 cbp==0 && mvp == mv && redIndex == 0 则修改mb_type为 skip