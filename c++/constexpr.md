Constexpr是c++11/14的关键字，看上去和const有点像，但是用法有区别：

const并不能代表“常量”，它仅仅是对变量的一个修饰，告诉编译器这个变量只能被初始化，且不能被直接修改（实际上可以通过堆栈溢出等方式修改）。而这个变量的值，可以在运行时也可以在编译时指定。

constexpr可以用来修饰变量、函数、构造函数。一旦以上任何元素被constexpr修饰，那么等于说是告诉编译器 “请大胆地将我看成编译时就能得出常量值的表达式去优化我”。

因此，如果成员变量中的const值，使用constexpr修饰就可以直接在声明的时候给到初始化值，只要初始化值编译期可以确定。而使用const修饰，非int和enum类型并不能声明时给到初始化值

https://www.jianshu.com/p/34a2a79ea947