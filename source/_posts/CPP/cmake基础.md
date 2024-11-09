---
title: cmake基础
tags: cpp
abbrlink: 14555
date: 2024-11-10 02:33:36
---

# 1 概述

选自百度百科:

CMake是一个跨平台的安装（[编译](https://baike.baidu.com/item/编译/1258343?fromModule=lemma_inlink)）工具，可以用简单的语句来描述所有平台的安装(编译过程)。他能够输出各种各样的makefile或者project文件，能测试[编译器](https://baike.baidu.com/item/编译器/8853067?fromModule=lemma_inlink)所支持的C++特性,类似[UNIX](https://baike.baidu.com/item/UNIX/219943?fromModule=lemma_inlink)下的automake。只是 CMake 的[组态档](https://baike.baidu.com/item/组态档/4812025?fromModule=lemma_inlink)取名为 CMakeLists.txt。Cmake 并不直接建构出最终的软件，而是产生标准的建构档（如 Unix 的 Makefile 或 [Windows](https://baike.baidu.com/item/Windows/165458?fromModule=lemma_inlink) [Visual C++](https://baike.baidu.com/item/Visual C%2B%2B/1811800?fromModule=lemma_inlink) 的 projects/workspaces），然后再依一般的建构方式使用。这使得熟悉某个[集成开发环境](https://baike.baidu.com/item/集成开发环境/298524?fromModule=lemma_inlink)（IDE）的开发者可以用标准的方式建构他的软件，这种可以使用各平台的原生建构系统的能力是 CMake 和 SCons 等其他类似系统的区别之处。

linux 下系统直接安装

```
sudo apt install cmake
```

查看cmake版本

```shell
cmake --version
cmake version 3.16.3

CMake suite maintained and supported by Kitware (kitware.com/cmake).
```

我们一般都是通过cmake生成对应的makefile后来运行make。

# 2 单个文件

只有单个main.cpp的时候可以用如下CMakeLists.txt文件

```cmake
# cmake最低版本需求
cmake_minimum_required(VERSION 3.10)

# 工程名称
project (demo)

# 设置C标准还是C++标准
set(CMAKE_C_STANDARD 11)

add_executable(demo
        main.cpp)
```

# 3 多个文件夹

但是实际开发中玩玩会更加复杂。

比如:下面每个模块都有自己的文件夹,并且每个文件夹下都有自己对应的头文件和源文件。

```shell
.
├── abstract
│   ├── inc
│   │   └── abstractModel.h
│   └── src
│       └── abstractModel.cpp
├── blueEar
│   ├── inc
│   │   └── blueEarModel.h
│   └── src
│       └── blueEarModel.cpp
├── CMakeLists.txt
├── CmakeOut
├── main.cpp
├── out
└── README.md
```

这个时候我们可以看如下的内容。

```cmake
# cmake最低版本需求
cmake_minimum_required(VERSION 3.10)

# 工程名称
project (demo)

# 设置C标准还是C++标准

#set(CMAKE_C_STANDARD 11) #c标准
set(CMAKE_CXX_STANDARD 11)#c++标准11

# 设置可执行文件输出路径
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/out)

# set (EXECUTABLE_OUT_PATH ${PROJECT_SOURCE_DIR}/out)

#
include_directories (   abstract/inc/
                        blueEar/inc/
                        )

aux_source_directory (abstract/src/ abstract_path)
aux_source_directory (blueEar/src/ blueEar_path)

add_executable(demo
        main.cpp
        ${abstract_path}
        ${blueEar_path})
```

可以看到如上的内容。

通过include_directories将相关的头文件包含进来

```cmake
include_directories
使用的格式如下:
include_directories (   abstract/inc/
                        blueEar/inc/
                        )
```

另外通过aux_source_directory将相关的cpp文件都加载到对应的变量中

```cmake
aux_source_directory

#使用的格式如下:
aux_source_directory (abstract/src/ abstract_path)
aux_source_directory (blueEar/src/ blueEar_path)
```

```cmake
# 设置可执行文件输出路径.会输出到项目文件下的out目录下
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/out)
```

cmake过程中会生成大量的中间文件。

其中的一种做法可以跟我上面一样建立一个 CmakeOut文件夹.

然后进入到这个文件夹中,运行

```
cmake ..
```

这样就会把编译的中间文件生成到这个目录中,不至于破坏工程的目录。


