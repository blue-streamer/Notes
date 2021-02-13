使用Xcode 创建一个opencv c++工程，选择command line tool app

demo中需要使用camera，因此需要获取camera 权限，修改plist文件即可。但是 commad line tool app似乎没有生成info.plist文件

在工程中自己创建一个Info.plist文件，添加上获取Camera权限的字段。然后，在“Build Phases”的“Copy Files”中添加这个文件。这样，Info.plist就会和编译生成的binary放在一起然后生效。

在xcode 中右键打开"product"文件夹下的可执行文件，即可看到binary和Info.plist。在xcode的derivedData目录中