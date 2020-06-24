最近在使用VTune分析性能时，遇到了一些不明白的概念，这里记录一下。

## concept

- uops

  micro-operation。是指一些宏观指令在cpu执行中转化成的底层的操作。微操作可能是把数据存入一个或多个寄存器，在寄存器或总线之间转移数据，在寄存器上执行算术或者逻辑操作等等。cpu执行一个宏观指令的经典过程是获取指令---解码指令---执行指令，每个宏观指令都会被拆分成多个微操作序列。注意，在现代处理器中，这些微操作可能不会被顺序执行，基于优化的目的cpu可能同时取多条指令，这些微操作可能被重排序，混合，暂存或者并行处理。

  > https://en.wikipedia.org/wiki/Micro-operation

- front end / back end

  现代cpu采用流水线处理方式，通常首先从内存中获取指令，放入指令队列。然后，在指令队列中解码指令到微操作，还会对多个指令的微操作进行混合。然后把微操作给到操作执行单元，算术运算单元，浮点乘法单元等等。现代的cpu一个cycle可以执行多个微操作，因此指令队列在一个cycle中会发送多个解码后的微操作到执行单元。在这个流程中，获取指令和解码指令的部分称为 front end，执行微操作的部分称为back end或者execution core。当一个cycle中有某些微操作无法被执行，此时这个cycle就被浪费了。这个浪费的来源可能是指令没有被及时获取，解码；也有可能是微操作正在等待内存资源。如果是前者就称为front end bound，后者就称为back end bound。如果微操作被顺利执行则为retire。

  > https://software.intel.com/content/www/us/en/develop/documentation/vtune-help/top/reference/cpu-metrics-reference/front-end-bound.html
  >
  > https://software.intel.com/content/www/us/en/develop/documentation/vtune-help/top/reference/cpu-metrics-reference/back-end-bound.html

理解一些概念之后可以更有效地利用VTune来找出代码的运行瓶颈

> https://stackoverflow.com/questions/22165299/what-are-stalled-cycles-frontend-and-stalled-cycles-backend-in-perf-stat-resul

## front end bound

从总体上来讲，指令没有被成功地解码为微操作。back end在空闲状态，没有接受到微操作而引起的cycle浪费。

### Eliminating Branches  

front end中有一个重要的组成模块，branch prediction unit。它是用来进行分支预测的，预测后续的代码会执行到哪些分支。并把它预先放入指令buffer。如果预测错误，则这些指令将不能被执行，从而需要从指令buffer中移除。这种情况频繁发生时将会导致大量的front end bound。因此，尽量减少代码分支。对应到汇编代码，要尽量减少跳转指令的使用，让所有的代码尽量在一个block中。

例如：

```c
X = (A < B) ? CONST1 : CONST2;
```

这就是一种典型的不可预测的分支结构。对应的汇编代码：

```assembly
cmp a, b ; Condition
jbe L30 ; Conditional branch
mov ebx const1 ; ebx holds X
jmp L31 ; Unconditional branch
L30:
mov ebx, const2
L31:
```

其中的cmp的结果决定了执行的block，不太可能被准确预测。因此可以使用别的指令来优化

```assembly
xor ebx, ebx ; Clear ebx (X in the C code)
cmp A, B
setge bl ; When ebx = 0 or 1
; OR the complement condition
sub ebx, 1 ; ebx=11...11 or 00...00
and ebx, CONST3; CONST3 = CONST1-CONST2
add ebx, CONST2; ebx=CONST1 or CONST2
```

对于c/c++代码而言，**尽量减少分支和展开for循环对减少front end bound有帮助的**

### Static Prediction  

### Inlining, Calls and Returns  

当有大量的call return但函数体很小的时候，出现时，使用inline function 有助于减少front end bound。使用宏来代替函数也是一个很好的方向。尤其对于频繁调用的函数

## back end bound

最大的back end bound一般是memory bound，合理利用内存资源对优化back end很重要。

sandy bridge之后，会有多个port读取cache。如下代码

```c
int buff[BUFF_SIZE];
int sum = 0;
for (i=0;i<BUFF_SIZE;i++){
sum+=buff[i];
}
```

每次循环内部的加法会依赖前一次的结果，这样会阻碍back end的并行计算。