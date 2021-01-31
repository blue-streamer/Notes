### 使用c++生成python的扩展模块

python 可以允许用户自己使用c/c++生成某个功能模块在python中调用，这种功能模块称为c/c++库的绑定(bindings)。在python中bindings库表现为一个正常的modules，使用import导入。bindings对python提供一些接口来调用底层c/c++的功能。

![bindings](sip&PyQt5.assets/bindings.png)

bindings实际上是一个按照某种接口规则生成的动态库，在windows上是xxx.pyd文件，mac上是xxx.so文件。把这种生成的文件放在python文件的加载路径里面，import xxx即可。import xxx的过程实际上是进行主动加载动态库的过程，在mac上会调用dlopne函数。而c/c++ library也要提供动态库的形式。bindings会依赖c/c++动态库，此时具体提供功能的library也会一起加载进入python进程。

c/c++ library也可以是静态库，此时生成bindings时会把c/c++的源代码一起带包进去

### SIP

sip是一种生成bindings的工具，它由Qt公司开发并开源。python中的Qt模块(比如PyQt5)就是使用sip生成的，查看PyQt5 package的安装目录，会看到Qt相关的动态库。

> https://riverbankcomputing.com/software/sip

目前sip有两个常用的版本，sip4和sip5。生成bindings的方法并不兼容，推荐使用sip5，在PyQt6之后，sip4的方法将被废弃。这里简单介绍一个sip4的使用过程。简单使用例子：

> https://www.jianshu.com/p/34088714381e

sip 4 Reference Guide：

> https://www.ics.uci.edu/~dock/manuals/sip/sipref.html#introduction

主要步骤：

- 下载sip源码，编译安装

  查看reference，会有详细说明

- 编写c/c++源文件生成动态库或静态库

- 编写sip文件

  xxx.sip文件中定义了暴露给python的接口。具体语法参见 reference。

- 调用sip命令，生成相关的文件

  sip会根据xxx.sip文件生成一个xxx.sbf文件，这个文件定义了构建的目标和文件。此外会生成一些需要的源文件和头问题。这些文件需要和c/c++ library一起编译

- 生成Makefile

  最终的bindings通过make来生成，sip提供了一个makefile生成的python class。使用这个class可以方便地生成makefile，这个makefile需要接受xxx.sbf中的信息。对于sip工程的配置，也有一个sipconfig的modules。可以把sip调用，sip配置和生成makefile一起在一个configure.py脚本中进行

- make，生成bindings

注意，如果c/c++ library是动态库，则需要在import的过程中可以找到并正确加载所有的依赖。

### 基于PyQt5的bindings

如果当前的bindings是基于PyQt5来开发的，或者说c/c++ library依赖了Qt。在生成bindings时有些地方需要特殊注意。在调用sip命令时，需要注意这个sip module的限定名称：

"-n name     the qualified name of the private copy of the sip module"

推荐使用PyQt5.QtCore.PYQT_CONFIGURATION。这个dict提供了必要的sip指令参数。否则的话，在import xxx会遇到"RuntimeError: the PyQt4.QtCore module failed to register with the sip module"。参考：

> https://phabricator.kde.org/D15091

