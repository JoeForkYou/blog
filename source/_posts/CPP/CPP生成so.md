---
title: CPP生成so
tags: cpp
categories: cpp
abbrlink: 18533
date: 2024-12-14 17:53:22
---


在C++中生成共享库（.so文件）并使用它，涉及到以下几个步骤。假设我们要创建一个简单的共享库并在一个主程序中调用它。

# 1: 创建共享库

1. **创建头文件**（例如 `example.h`）:

```cpp
#ifndef EXAMPLE_H
#define EXAMPLE_H

#ifdef __cplusplus
extern "C" {
#endif

void hello();

#ifdef __cplusplus
}
#endif

#endif // EXAMPLE_H
```

2. **创建实现文件**（例如 `example.cpp`）:

```cpp
#include <iostream>
#include "example.h"

void hello() {
    std::cout << "Hello, World!" << std::endl;
}
```

3. **编译生成共享库**:

使用以下命令编译 `example.cpp` 为共享库：

```bash
g++ -fPIC -shared example.cpp -o libexample.so
```

- `-fPIC` 表示生成位置无关代码（Position Independent Code），适合共享库。
- `-shared` 指定生成共享库。

# 2: 使用共享库

1. **创建主程序文件**（例如 `main.cpp`）:

```cpp
#include "example.h"

int main() {
    hello();
    return 0;
}
```

2. **编译并链接共享库**:

使用以下命令编译 `main.cpp` 并链接到生成的共享库：

```bash
g++ main.cpp -L. -lexample -o main
```

- `-L.` 指定当前目录为查找库的目录。
- `-lexample` 指定链接名为 `libexample.so` 的库（省略 `lib` 和 `.so`）。

3. **运行程序**:

在运行时，可能需要设置 `LD_LIBRARY_PATH` 环境变量，以便程序能够找到共享库。例如：

```bash
export LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH
./main
```

### 总结

通过上述步骤，你可以成功创建并使用一个简单的共享库。在实际应用中，可能会有更多的复杂性，比如处理多个源文件，或者使用构建工具（如 CMake）来管理构建流程。希望这个示例对你有所帮助！