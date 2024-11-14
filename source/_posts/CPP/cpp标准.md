---
title: '''cpp标准'''
tags: cpp
categories: cpp
abbrlink: 52398
date: 2024-11-14 22:47:09
---
# 1 概述

‌C++标准‌是C++编程语言的规范，由国际标准化组织（ISO）制定。
C++标准的发展历程可以追溯到1998年,
当时ISO/IEC 14882:1998标准被发布，这被认为是第一个C++标准，常被称为C++98。
随后，C++标准经历了多次更新和修订，
包括C++03（2003年）、C++11（2011年）、C++14（2014年）和C++17（2017年）。最新的C++标准是C++20，于2020年发布，引入了许多新特性，如概念（concepts）、范围库（ranges）、协程（coroutines）等。此外，C++23标准也在2023年确定，但目前支持完整的编译器较少。
C++标准的发展历程

‌C++98‌：1998年发布的第一个C++标准，常被称为C++98。
‌C++03‌：2003年发布的修订版，增加了对自动存储期变量的支持等新特性。
‌C++11‌：2011年发布的版本，增加了lambda表达式、自动类型推导等功能。
‌C++14‌：2014年发布的版本，增加了基于范围的for循环、constexpr等功能。
‌C++17‌：2017年发布的版本，增加了结构化绑定、文件系统库等功能。
‌C++20‌：2020年发布的版本，引入了概念（concepts）、范围库（ranges）、协程（coroutines）等新特性。
‌C++23‌：2023年确定的版本，目前支持完整的编译器较少。
目前按照我接触的标准来说.
市面上大部分项目都是以C++11/14/17/20为主, 而C++98/03则是少数.  这主要是因为一些老项目的历史原因, 也有一些公司的项目使用C++98/03, 这也是C++标准的发展历程.
本文先介绍个大概,后续再介绍C++11/14/17/20的详细新特性。
# 2 C++11 新特性

C++11标准引入了许多新特性，以下是一些重要的特性及其代码示例：

1. **自动类型推导（auto）**
   ```cpp
   auto x = 1;      // x 被推导为 int
   auto y = 2.5;    // y 被推导为 double
   ```

2. **范围for循环（for each）**
   ```cpp
   std::vector<int> vec = {1, 2, 3, 4, 5};
   for (auto& val : vec) 
   {
       std::cout << val << " "; // 输出每个元素
   }
   ```

3. **新类型：nullptr**
   ```cpp
   int* p = nullptr; // nullptr 是类型安全的空指针
   ```
   早期一直用的是NULL, 后来发现NULL是int类型, 所以就引入了nullptr, 它是一个空指针常量, 类型安全, 避免了类型转换错误。这使得代码更加安全和可读性更高。

4. **右值引用和移动语义**
   ```cpp
   class MyClass {
   public:
       MyClass() { std::cout << "Constructor" << std::endl; }
       MyClass(MyClass&& other) { std::cout << "Move Constructor" << std::endl; }
       MyClass(const MyClass& other) { std::cout << "Copy Constructor" << std::endl; }
   };

   MyClass obj1;
   MyClass obj2 = std::move(obj1); // 使用移动构造函数
   ```

5. **lambda表达式**
   ```cpp
   auto lambda = [](int x) { return x * x; };
   std::cout << lambda(5); // 输出25
   ```
   **注意lambda是最后使用发射的,在用QT的时候,我经常会使用connect函数, 它会自动生成一个lambda表达式, 这个时候要注意局部变量和全局变量的生命周期，如果在外层申明了一个局部变量，在lambda表达式中使用这个变量，就会出现未定义行为导致程序崩溃。**


6. **智能指针（std::unique_ptr和std::shared_ptr）**
智能指针单独开一个章节说明
   ```cpp
   std::unique_ptr<int> p1(new int(10)); // 独占所有权的智能指针
   std::shared_ptr<int> p2(new int(20)); // 共享所有权的智能指针
   ```

7. **线程支持库（std::thread）**
   ```cpp
   void threadFunction() 
   {
       std::cout << "Thread is running" << std::endl;
   }

   std::thread t(threadFunction);
   t.join(); // 等待线程结束
   ```

8. **静态断言（static_assert）**
   ```cpp
   static_assert(sizeof(int) == 4, "Size of int is not 4 bytes!");
   ```

9. **变长模板（Variadic templates）**
   ```cpp
   template <typename... Args>
   void print(Args... args) {
       (std::cout << ... << args) << std::endl; // 使用折叠表达式
   }

   print(1, 2, 3, "text"); // 输出: 123text
   ```

10. **枚举类（enum class）**
    ```cpp
    enum class Color { Red, Green, Blue };
    Color c = Color::Red; // 强类型枚举
    ```

以上是C++11引入的一些主要新特性及其简单示例，这些特性极大地增强了C++的功能和灵活性。

# 3 C++14 新特性

## 1. 二进制字面量

C++14 引入了二进制字面量，允许使用 `0b` 或 `0B` 前缀来表示二进制数字。

```cpp
#include <iostream>

int main() {
    int binaryNum = 0b101010; // 二进制 101010 等于十进制 42
    std::cout << "二进制 101010 的十进制值是: " << binaryNum << std::endl;
    return 0;
}
```

**注意事项**: 使用二进制字面量时，必须在编译器开启 C++14 标准的情况下编译代码。

## 2. 泛型 Lambda 表达式

在 C++14 中，Lambda 表达式支持模板参数，可以使用 `auto` 作为参数类型。

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> nums = {1, 2, 3, 4, 5};
    auto print = [](auto n) { std::cout << n << " "; };

    for (const auto& num : nums) {
        print(num); // 调用 泛型 Lambda
    }
    std::cout << std::endl;
    return 0;
}
```

**注意事项**: 泛型 Lambda 可能在某些老旧的编译器上不被支持，请确认编译器版本。

## 3. std::make_unique

C++14 引入了 `std::make_unique` 来简化 `std::unique_ptr` 的创建，避免手动使用 `new`。

```cpp
#include <iostream>
#include <memory>

int main() {
    auto ptr = std::make_unique<int>(42); // 创建一个 unique_ptr 并初始化
    std::cout << *ptr << std::endl;
    return 0;
}
```

**注意事项**: 使用 `std::make_unique` 可以防止内存泄漏，但是请确保使用 C++14 或更高版本编译。

## 4. 返回类型推导

C++14 允许推导函数的返回类型，可以使用 `auto` 关键字。

```cpp
#include <iostream>
#include <vector>

auto add(int a, int b) {
    return a + b; // 返回类型自动推导
}

int main() {
    std::cout << "3 + 5 = " << add(3, 5) << std::endl;
    return 0;
}
```

**注意事项**: 使用返回类型推导时，确保函数体简单，编译器能够清晰推导出返回类型。

## 5. std::shared_timed_mutex 和 std::shared_lock

C++14 引入了 `std::shared_timed_mutex` 和 `std::shared_lock`，支持更灵活的多线程锁。

```cpp
#include <iostream>
#include <thread>
#include <shared_mutex>

std::shared_timed_mutex mutex;

void read() {
    std::shared_lock<std::shared_timed_mutex> lock(mutex);
    std::cout << "Reading data" << std::endl;
}

void write() {
    std::unique_lock<std::shared_timed_mutex> lock(mutex);
    std::cout << "Writing data" << std::endl;
}

int main() {
    std::thread t1(read);
    std::thread t2(write);

    t1.join();
    t2.join();

    return 0;
}
```

**注意事项**: 当在多线程环境下使用锁时，确保正确地管理锁的生命周期，避免死锁和资源竞争。

## 总结

C++14 引入了多项新特性，增强了语言的灵活性和表达能力。在使用这些特性时，请注意兼容性和编译器支持情况，以确保代码的可移植性和稳定性。
