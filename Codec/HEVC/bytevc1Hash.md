hash计算和搜索的单位是8x8block，使用crc32计算hash值，使用crc32的高16位作为hash索引。后续hashKey代表crc32，hashIdx代表crc32的高16位。

对一帧图像来说，有4个关键的数据结构

- keyMap:

  这是一张map，大小是width * (height + 2 * ctu_size)。存储每个像素点对应的8x8block的hashKey

- maskMap:

  这是一张map，大小是width * (height + 2 * ctu_size)。存储一个bool值，表示当前8x8block是否为行列相同。行列相同的块不加入hash搜索

- header:

  这是一个hashTable，大小是 1<<16，使用hashIdx作为索引，其中每个hashIdx对应的位置存放的是等于当前hashIdx的block位置(使用左上像素位置标记)。这个block是按照编码扫描ctu的顺序中最近的一个。像素位置使用x + y*stride来表示。

- hashMap:

  这是一张map，大小是width * (height + 2 * ctu_size)。存储与当前位置block的hashIdx相同的block位置。这个block是按照编码扫描ctu的顺序中距离当前block最近的一个

### 建立过程

在ctu编码前，遍历当前ctu中所有的block。每个block有如下处理：

- 判断是否需要加入hash 搜索。如果需要，从keyMap中取出对应的hashKey
- 计算得到hashIdx
- 取出hashIdx中存储的pos0，更新为当前的pos1。
- hashMap位置当前pos1中存放pos0。

可以看出，header和hashMap共同组成了一个尾插建立的链表，header中使用hashIdx来索引，存放最近的block pos。hashMap中存放的是某个block pos在链表中对应的下一个pos。

### 使用过程

得到当前待搜索的block位置，从对应的keyMap中取出hashKey，计算hashIdx。从对应的header和hashMap中遍历，比较hashKey来判断是否完全相同。如果hashKey相同则搜索成功，计算cost。