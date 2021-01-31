MacOS上的很多动态依赖是使用framework的，如果编译过程中依赖了某个framework，引用了其中的头文件。需要在编译阶段做如下改动：

- 在include的搜索路径上添加该framework的头文件路径，一般为.../xxx.framework/Headers。使得编译可以通过

- 在cxxflags中添加"-F framework_path"，framework_path是framework的所在路径。

  如果不添加"-F"，include递归包含到framework内部的头文件时，会出现查找不到文件的问题。具体原因还不清楚



-std 

这个编译选型决定了编译时使用的c++语法标准，"-std=c++11"使用c++11，"-std=gnu++11"C++11标准和GNU扩展特性。

-stdlib

指定使用的c++标准库

-isystem

增加路径到系统include所有路径