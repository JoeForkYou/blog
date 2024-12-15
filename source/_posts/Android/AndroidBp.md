---
title: AndroidBp
tags: Android
categories: Android
abbrlink: 60717
date: 2024-12-15 10:30:39
---
Android.bp 是 Android Build System 中使用的一种构建描述文件格式，类似于 Makefile，但以更简洁的方式定义构建模块。以下是一个基本的 Android.bp 文件示例，展示了其基本语法和用法。

```bp
# 定义一个库模块
cc_library {
    name: "my_library",  # 模块名称
    srcs: [
        "file1.cpp",      # 源文件
        "file2.cpp",
    ],
    cflags: [            # 编译选项
        "-Wall",
        "-Werror",
    ],
    shared_libs: [      # 依赖的共享库
        "liblog",       # 依赖的库名称
    ],
}

# 定义一个可执行程序模块
cc_binary {
    name: "my_executable",  # 执行文件名称
    srcs: [
        "main.cpp",           # 主源文件
    ],
    static_libs: [
        "my_library",        # 依赖的静态库
    ],
    cflags: [
        "-O2",               # 优化选项
    ],
}

# 定义一个测试模块
cc_test {
    name: "my_library_test",  # 测试模块名称
    srcs: [
        "my_library_test.cpp",  # 测试源文件
    ],
    static_libs: [
        "my_library",           # 被测试的库
    ],
}
```

# 说明
1. **模块类型**：可以定义多种类型的模块，比如 `cc_library`（C++库），`cc_binary`（可执行程序），`cc_test`（测试）等。
2. **name**：每个模块都有一个唯一的名称。
3. **srcs**：指定源文件列表，可以是相对路径。
4. **cflags**：编译标志，用于设置编译时的选项。
5. **shared_libs/static_libs**：分别指定依赖的共享库和静态库。

了解这些基本组件后，你可以根据自己的项目需求，进一步扩展和修改 Android.bp 文件。