原始crash的堆栈都是纯地址的形式，使用dsym文件分析.crash，可以符号化堆栈到具体的行号

dsym文件中包含了app、framework或者dylib的所有符号信息，每个符号地址的偏移量，还有每个函数中的行号信息。crash堆栈中的地址，都可以从dsym文件中获得具体的函数，文件和行号。

简单总结一下dsym中的内容，

https://www.jianshu.com/p/480c0a2ac1a4

https://blog.csdn.net/goldWave01/article/details/90177708