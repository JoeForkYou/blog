---
title: AndroidMk基础
date: 2024-11-24 15:49:34
tags: Android
categories: Android
---


Android.mk 文件是 Android 系统中用于描述构建过程的文件，主要用在 Android NDK（Native Development Kit）中。它们是用来定义模块的构建规则和依赖关系。以下是 Android.mk 文件的基础和常用语法：

#  基础结构

1. **变量定义**：
   - 可以使用 `:=`、`+=` 等方式定义变量。
   - 例如：
     ```makefile
     LOCAL_PATH := $(call my-dir)
     ```

2. **模块类型**：
   - 使用 `LOCAL_MODULE` 来指定模块名称。
   - 使用 `include` 来包含其他 mk 文件。

3. **模块的属性**：
   - `LOCAL_SRC_FILES`：指定源文件。
   - `LOCAL_C_INCLUDES`：指定 C/C++ 头文件的搜索路径。
   - `LOCAL_LDLIBS`：链接时需要的库。

#  常见语法

1. **定义静态库**：
   ```makefile
   LOCAL_PATH := $(call my-dir)
   include $(CLEAR_VARS)

   LOCAL_MODULE := my_static_lib
   LOCAL_SRC_FILES := my_static_lib.cpp
   LOCAL_C_INCLUDES := $(LOCAL_PATH)/include

   include $(BUILD_STATIC_LIBRARY)
   ```

2. **定义动态库**：
   ```makefile
   LOCAL_PATH := $(call my-dir)
   include $(CLEAR_VARS)

   LOCAL_MODULE := my_shared_lib
   LOCAL_SRC_FILES := my_shared_lib.cpp
   LOCAL_LDLIBS := -llog

   include $(BUILD_SHARED_LIBRARY)
   ```

3. **定义可执行文件**：
   ```makefile
   LOCAL_PATH := $(call my-dir)
   include $(CLEAR_VARS)

   LOCAL_MODULE := my_executable
   LOCAL_SRC_FILES := main.cpp

   include $(BUILD_EXECUTABLE)
   ```

4. **清理变量**：
   - 使用 `include $(CLEAR_VARS)` 可以清除之前定义的变量，避免冲突。

5. **依赖关系**：
   - 可以使用 `LOCAL_WHOLE_STATIC_LIBRARIES` 来定义静态库的依赖。

#  其他指令

- **链接库**：
  - `LOCAL_LDLIBS` 用于定义需要链接的库，例如 `-lc`、`-lm`等。

- **编译选项**：
  - 还可以使用 `LOCAL_CPPFLAGS` 和 `LOCAL_CFLAGS` 来添加编译选项。

#  总结

Android.mk 文件通过模块定义和构建规则来构建 C/C++ 代码。在编写文件时，需要注意规范和模块间的依赖关系。随着 Android 的发展，渐渐地 CMake 也被广泛使用作为构建系统，但 Android.mk 仍然是一个重要的部分，特别是在 NDK 项目中。
