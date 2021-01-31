cache是cpu与memory之间的缓存，cpu访问内存空间时并没有直接从内存里读取，而是从cache中读取。写入也是一样的。所以，cpu之和cache进行数据的交互。

## cache的组织结构：

### 整体结构

当cpu访问一段数据，但是cache中并没有时，就发生了cache miss。此时，cpu进行等待，机器中的其他单元负责把memory中的数据放入cache。不同level的cache大小和访问时间不太一样。现代的cpu都会有3级cache，L1/L2/L3。访问L3一般需要几十个指令周期，L2十几个指令周期，L1几个指令周期。大小也不相同，L1一般几十k，L2几M。

L1 cache分为 L1d(data)和L1i(instruction)，每个逻辑核有独自的L1d，同一个物理核的两个逻辑核共享L1i和L2，一组逻辑核共享L3，如下图

![image](cache.assets/image-1484845.png)



## cache line/cache set

在cache中，管理数据的最小单位是cache line。一般为64Byte，早期的cpu使用过32Byte。例如，一个32KB大小的Lid Cache，其中有512个cache line。一个4MB大小的L2 Cache，其中有65536个cache line。

如果cache需要和memory交换数据，则交换的最小单位也是cache line。例如，cpu 需要一个数据，但是cache中没有缓存，则需要先从memory中load。此时会直接load一段64Byte的数据，其中包含所需的数据。这个操作很简单，对应的数据地址的低6位会被mask为0，然后从内存中取数据放入cache中的某个cache line。

cache line在某个level的cache中是通过，cache set来组织的。例如，一个4M大小的L2 Cache，通常会被组织成16组。则一组中有4096个cache line。每个cache line会有一个tag，这个tag用来标记它其中存放的数据。

当cpu需要内存中的数据时，cpu会使用数据的内存地址来搜索cache。以32位的地址为例：

<img src="../../Desktop/截屏2021-01-31 下午3.55.30.png" alt="截屏2021-01-31 下午3.55.30" style="zoom:50%;" />

地址分为3段

Tag：标记

Cache Set：cache set(组)索引

Offset：数据在cache line中的偏移

首先使用S段来查找cache set，然后使用T段来与cache line的tag做对比。如果找到了，最后使用O段来计算偏移，从cache line中找到需要的字节。如下图所示：

<img src="../../Desktop/截屏2021-01-31 下午9.04.43.png" alt="截屏2021-01-31 下午9.04.43" style="zoom:50%;" />

也有一种cache line的组织方式，s段长度为0。可以认为所有的cache line在同一个set。这种情况下，需要更多的comparator。这种形式适合比较小的cache size，一般TLB会使用这种结构。

### cache read/write

cache读写的最小单位是cache line。当读行为发生时，有两种不同的策略

exclusive cache：直接把数据从memory 加载进入L1d，不经过L2。当有新的数据进入L1d，cache line需要换出时，把这个cache line放入L2。如果还有需要放入L3，直到memory。

Inclusive cache：数据加载进入L1d是通过L2，进入L2通过L3。当有新的数据进入L1d，cache line需要换出时，直接覆盖。L2中仍然保留了覆盖的cache line。

exclusive cache 读入新数据更快，而inclusive cache 换出数据更快。AMD使用exclusive，Intel使用Inclusive



当一个数据要写入内存时，对应的cache line会首先被加载进cache，然后修改对应的cache line。修改后的cache line有两种不同的处理方式

Write-through：cache line会直接被写入内存，然后这个cache line直接可以再次使用

Write-back：cache line不会被马上写入内存，而是被标记为dirty。当有数据进来，需要使用这个cache line时再写入内存。

Write-through可以很好的保证多核和多线程之间的数据一致性，但是会浪费带宽。Write-back性能更好，但是一致性需要特殊处理。正常情况下，cache 都是使用Write-back。数据一致的问题，可以参看"3.3.4 Multi-Processor Support"



## 如何优化cache

### memory align

由于memory和cache进行数据交换是以cache line来进行的，因此，数据的内存对齐就很有必要了。如果程序要访问一个数组的全部数据，int array[16]。此时，这个数组一共64个字节，一个cache line。如果数据对齐，只需要一次load就可以在cache 中得到所有数据，如果不是对齐的则需要两次

分配在栈和全局区域的内存可以使用__attribute((aligned(64)))，分配在堆上的内存可以使用posix_memalign。具体的对齐修饰和对齐分配可以查阅资料

### cache bypass

从cache的正常读写来看，数据都是首先load进入cache，然后在被写回内存或者换出。当数据读入之后只被使用一次，短期不再使用，或者数据产生后短期不再使用，这种数据称为"Non-temporal aligned"(NTA)数据。例如，访问一个大数组的每个元素，只使用一次。此时，每个数据都通过cache是比较低效的。可以使用"stream"指令，cpu直接访问数据而不会破坏cache。

write：_mm_stream**指令，直接把数据写入内存而不进入cache。会考虑使用write-combine来尽可能减少带宽的使用

read：_mm_stream_load_si128指令，使用了一个streaming load buffer。最小的单位也是cache line。

### cache access

cache line size和L1d size一般情况下不同的cpu大致相同。优化的策略就是：尽可能使用已经在cache中的数据。

数据cache line size对齐，设计结构体尽可能在cache line size内，并且load之后尽可能使用。

L1d中的数据尽可能使用，局部化

6.2.1中的矩阵乘法很好地展示了这一点

### cache prefetch

硬件层面的prefetch已经广泛地应用在了L1 cache和L2 cache中，如果访问连续的内存，如数组遍历或者memset。此时，硬件会自动perfetch。另外，现代的指令执行pipeline会打乱指令，这样数据的load和指令的执行可能同时进行。

但是，硬件的prefetch有个限制。prefetch的数据不能跨越page。这样可能会造成缺页，从而影响性能。

软件层面，提供了void _mm_prefetch(void *p,enum _mm_hint h)。这条指令执行后p所在的64Byte对齐的内存会被加载进入cache。不同的hint代表了不同的加载策略。可以查看intel 指令文档

由于存在硬件prefetch，并且cache的具体操作大部分都是由cpu来管理的。software prefetch 需要很好地设计，插入才能起作用。不同的机器会有不同的表现。可以参看讨论

> https://stackoverflow.com/questions/48994494/how-to-properly-use-prefetch-instructions

### cache associativity

> http://igoro.com/archive/gallery-of-processor-cache-effects/

这个blog中很好的展示了cache大小对程序性能的影响，其中“example 5: Cache associativity”是指具体cache的结构对性能的影响

通常情况cache的结构“N-way set associative cache”，即cache line会分成x组，每组中有n个cache line。当每个cache line属于某个组时，可以使用n个中的任何一个空闲的cache line。

例如，4M大小的L2 cache，如果是16way的cache 结构。则每个组中有个16个cache line，一个组的大小为16*64=1024B，一共有4M/1024 = 4096组。则，某个字节地址的某12位来确定映射到哪个组，一个连续的64Byte属于同一个cache line。如果我们固定index cache组的12位地址和确定cache line 大小的低6位地址，即如果我们固定了18位地址，则这18位地址相同的数据会竞争16个cache line。因此，当循环中访问了262144（2^18）倍数地址的次数超过了16次，就会出现cache miss。