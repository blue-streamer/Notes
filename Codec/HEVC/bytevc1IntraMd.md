只有在8x8cu是才会尝试划分pu到NxN，其他size默认pu不划分

md的过程比较简单：

1. 尝试DC和planar

2. 判断subCu是否已经md

   - 如果subCu已经md(无论是不是intra)，尝试mpm和subCuMode(去除mpm)。得到最佳intraMode

   - 否则，尝试mpm和{2, 10, 18, 26, 34}(去除mpm)。然后，找到cost最小的intraMode，如果是角度预测，则在附近搜索最佳。

角度最佳搜索：以当前最优点pos为起点，尝试pos+step和pos-step。step初始化为4，每次右移一位。直到step=0

