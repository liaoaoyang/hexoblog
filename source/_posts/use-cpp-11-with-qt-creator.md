date: 2014-12-28 00:00:00
title: 使用QtCreator配置C++ 11开发调试环境
categories: Try
tags: [C++,QtCreator,CMake]
---

编译[`lwan`][22]的时候顺道看了下MinGW，发现64位版本也更新到了[`4.9`][23]，感觉自己很久没更新过Windows下的C++环境了，也想体验一下C++ 11，顺手升级了一下。

<!-- more -->

自己一直使用[`QtCreator`][24]作为C++开发环境，这个IDE个人很喜欢，自动补全功能相当方便，之前在开发Qt程序的时候这几乎是最好的选择，现在也开始出现了收费版本，但是仍然有社区版可用。目前CLion还没有放出正式版本，VS感觉又太重，QtCreator感觉还是一个很不错的选择。

QtCreator的是支持[`CMake`][25]的，正好自己想要学习CMake，于是打算选用CMake作为构建部分的工具。

以下是简要的配置步骤：

**1.安装[`MinGW64`][23]**

MinGW现在使用安装器的形式进行安装，下载完成一个很小的安装器后进行安装，期间可能是比较漫长的下载过程。不再多说。

**2.安装CMake**

一样是常规的安装步骤。

**3.QtCreator配置**

首先是配置编译器路径信息：

在`工具`->`选项`中进行配置：

![配置编译器][3]

之后配置GDB路径信息：

![配置GDB][4]

还需要配置上CMake路径信息：

![配置CMake][5]

上述步骤完成之后根据之前起好的别名（即上图中的Name字段），选择组成构建套件：

![配置构建套件][2]

**4.项目级配置**

首先自然是常规的新建项目了：

![新建项目][1]

在项目类型这里，需要选择使用CMake构建的纯C++项目：

![新建使用CMake的纯C++项目][6]

建立完成之后选择好保存的位置

![确定项目保存位置][7]

![确认项目信息][8]

之后开始配置Debug构建目录

![配置Debug构建目录][9]

之后生成CMake相关文件，这里需要指定构建版本为Debug版本，即在参数中指明`-DCMAKE_BUILD_TYPE=Debug`:

![设置CMake参数为Debug构建模式][10]

之后项目会生成一个`main.cpp`和一个`CMakeLists.txt`文件：

![新项目文件][11]

在`CMakeLists.txt`文件中增加如下配置，告知检查编译器是否支持C++ 11。

```
include(CheckCXXCompilerFlag)CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)if(COMPILER_SUPPORTS_CXX11)	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")elseif(COMPILER_SUPPORTS_CXX0X)	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")else()	message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")endif()

```

之后可以对项目这一构建模式由默认的all重命名为Debug，重新对项目执行CMake操作。

![重命名为Debug][12]

![重新执行CMake操作][13]

完成上述操作之后，Debug对象的构建配置工作基本完成，可以通过如下代码检验是否能够支持C++ 11的语法：

```
#include <iostream>#include <vector>#include <algorithm>using namespace std;int main(){        vector<int> v = {5, 2, 1, 4, 3};        sort(v.begin(), v.end());        for(int i: v)            cout << i << " ";        cout << endl;        return 0;}
```

![编辑代码][14]

之后可以尝试构建Debug对象（点击左下角锤子图标）：

![构建Debug对象][15]

尝试运行：

![运行Debug对象][16]

发现能够成功运行，那么至少配置Debug对象这一步就没有问题了。

**5.调试**

既然生成了Debug对象，那么调试功能自然不能少，尝试打一个断点：

![断点调试][17]

可以看出，在断点处查看局部变量等核心调试功能已经能够正常工作，单步、步入等功能也都可以正常工作。

**6.发布程序**

构建了Debug对象，那么作为生成产物的Release对象自然也不能少。

可以在添加一个构建模式：

![添加Release构建模式][18]

Debug对象与Release对象会通过CMake生成不同的配置，需要将二者放到不同的目录中：

![配置Release构建目录][19]

CMake参数上的区别在于需要指明`-DCMAKE_BUILD_TYPE=Release`:

![配置CMake的Release参数][20]

之后的构建运行同Debug。

![构建运行Release对象][21]

（CentOS下升级GCC参考了[`这里`][26]）。

以上。





[1]: https://blog.wislay.com/wp-content/uploads/2014/12/1.png
[2]: https://blog.wislay.com/wp-content/uploads/2014/12/2.png
[3]: https://blog.wislay.com/wp-content/uploads/2014/12/3.png
[4]: https://blog.wislay.com/wp-content/uploads/2014/12/4.png
[5]: https://blog.wislay.com/wp-content/uploads/2014/12/5.png
[6]: https://blog.wislay.com/wp-content/uploads/2014/12/6.png
[7]: https://blog.wislay.com/wp-content/uploads/2014/12/7.png
[8]: https://blog.wislay.com/wp-content/uploads/2014/12/8.png
[9]: https://blog.wislay.com/wp-content/uploads/2014/12/9.png
[10]: https://blog.wislay.com/wp-content/uploads/2014/12/10.png
[11]: https://blog.wislay.com/wp-content/uploads/2014/12/11.png
[12]: https://blog.wislay.com/wp-content/uploads/2014/12/12.png
[13]: https://blog.wislay.com/wp-content/uploads/2014/12/13.png
[14]: https://blog.wislay.com/wp-content/uploads/2014/12/14.png
[15]: https://blog.wislay.com/wp-content/uploads/2014/12/15.png
[16]: https://blog.wislay.com/wp-content/uploads/2014/12/16.png
[17]: https://blog.wislay.com/wp-content/uploads/2014/12/17.png
[18]: https://blog.wislay.com/wp-content/uploads/2014/12/18.png
[19]: https://blog.wislay.com/wp-content/uploads/2014/12/19.png
[20]: https://blog.wislay.com/wp-content/uploads/2014/12/20.png
[21]: https://blog.wislay.com/wp-content/uploads/2014/12/21.png
[22]: http://lwan.ws
[23]: http://mingw-w64.sourceforge.net/download.php#mingw-builds
[24]: https://qt-project.org
[25]: http://www.cmake.org
[26]: http://ilovers.sinaapp.com/article/centos%E4%B8%8B%E5%AE%89%E8%A3%85gcc-481


