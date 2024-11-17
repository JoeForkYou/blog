---
title: 【CPP基础】一
tags: cpp
categories: cpp
abbrlink: 47760
date: 2024-11-17 20:06:49
---
# 概述
这是对c++的泛式编程的梳理
c++的基础就那边点,无非就是流程控制和对象继承等
剩下的就是STL的内容。
所以我觉得还是有必要深究下来，一来就是我这个人的性格是这样的.
学习链接:
https://www.learncpp.com/

# 基本
## 数据类型
- 整型: int, long, long long
- 浮点型: float, double, long double
- 字符型: char, wchar_t
- 布尔型: bool
- 指针型: pointer, reference
- 数组型: array
- 枚举型: enum
- 结构体型: struct
- 类型: class

| 类型     | 关键字  |
| -------- | ------- |
| 布尔型   | bool    |
| 字符型   | char    |
| 整型     | int     |
| 浮点型   | float   |
| 双浮点型 | double  |
| 无类型   | void    |
| 宽字符型 | wchar_t |

补充表格

| 类型               | 位    | 范围         |
| ------------------ | ----- | ------------ |
| char               | 1字节 | -128~127     |
| unsigned char      | 1字节 | 0~255        |
| signed char        | 1字节 | -128~127     |
| int                | 4字节 | -2^31~2^31-1 |
| unsigned int       | 4字节 |              |
| signed int         | 4字节 |              |
| short int          | 2字节 |              |
| unsigned short int | 2字节 |              |
| signed short int   | 2字节 |              |
| long int | 8字节 ||
|signed long int|8字节||
|unsigned long int|8字节||
|float|4字节||
|double|8字节||
|long long|8字节||
|long double|16字节||

typedef 使用格式如下

```c++
typedef type newname;
typedef int INT;
INT INI16;
```

## 类型转换

### 静态转换

静态转换是将一种数据类型的值强制转换为另一种数据类型的值。

静态转换通常用于比较类型相似的对象之间的转换，例如将 int 类型转换为 float 类型。

静态转换不进行任何运行时类型检查，因此可能会导致运行时错误。

```cpp
int i = 10;
float f = static_cast<float>(i); // 静态将int类型转换为float类型
```

### 动态转换

动态转换通常用于将一个基类指针或引用转换为派生类指针或引用。动态转换在运行时进行类型检查，如果不能进行转换则返回空指针或引发异常。

```c++
class Base {};
class Derived : public Base {};
Base* ptr_base = new Derived;
Derived* ptr_derived = dynamic_cast<Derived*>(ptr_base); // 将基类指针转换为派生类指针
```

完整代码如下

```c++
#include <iostream>
using namespace std;

class Base 
{
public:
    virtual void func() = 0; // 纯虚函数
protected:
    int data;
};

class Derived : public Base 
{
public:
    void func() 
    {
        cout << "Derived::func()" << endl;
    }
};

int main() 
{
    Base* ptr_base = new Derived;
    Derived* ptr_derived = dynamic_cast<Derived*>(ptr_base); // 将基类指针转换为派生类指针
    ptr_derived->func(); // 调用派生类函数

    return 0;
}

```

###  常量转换

```c++
const int i = 10;
int& r = const_cast<int&>(i); // 常量转换，将const int转换为int
```

### 重新解释转换

重新解释转换将一个数据类型的值重新解释为另一个数据类型的值，通常用于在不同的数据类型之间进行转换。

重新解释转换不进行任何类型检查，因此可能会导致未定义的行为。

```cpp
int i = 10;
float f = reinterpret_cast<float&>(i); // 重新解释将int类型转换为float类型
```

## 变量作用域

一般来说有三个地方可以定义变量：

- 在函数或一个代码块内部声明的变量，称为**局部变量**。
- 在函数参数的定义中声明的变量，称为**形式参数**。
- 在所有函数外部声明的变量，称为**全局变量**。

作用域是程序的一个区域，变量的作用域可以分为以下几种：

- **局部作用域**：在函数内部声明的变量具有局部作用域，它们只能在函数内部访问。局部变量在函数每次被调用时被创建，在函数执行完后被销毁。
- **全局作用域**：在所有函数和代码块之外声明的变量具有全局作用域，它们可以被程序中的任何函数访问。全局变量在程序开始时被创建，在程序结束时被销毁。
- **块作用域**：在代码块内部声明的变量具有块作用域，它们只能在代码块内部访问。块作用域变量在代码块每次被执行时被创建，在代码块执行完后被销毁。
- **类作用域**：在类内部声明的变量具有类作用域，它们可以被类的所有成员函数访问。类作用域变量的生命周期与类的生命周期相同。

## 字符串

常用的几个如下

| 函数                | 说明                                                         |
| ------------------- | ------------------------------------------------------------ |
| **strcpy(s1,s2)**   | 复制字符串s2到字符串s1                                       |
| **strcat(s1, s2);** | 连接字符串 s2 到字符串 s1 的末尾。连接字符串也可以用 **+** 号，例如:<br/>string str1 = "demo1";<br/>string str2 = "demo2";<br/>string str = str1 + str2; |
| **strlen(s1);**     | 返回字符串 s1 的长度。                                       |
| **strcmp(s1, s2);** | 如果 s1 和 s2 是相同的，则返回 0；如果 s1<s2 则返回值小于 0；如果 s1>s2 则返回值大于 0。 |
| **strchr(s1, ch);** | 返回一个指针，指向字符串 s1 中字符 ch 的第一次出现的位置。   |
| **strstr(s1, s2);** | 返回一个指针，指向字符串 s1 中字符串 s2 的第一次出现的位置。 |

# 指针

```c++
#include <iostream>
using namespace std;
int main ()
{
   int  var1 = 10;
   char var2[12] = "Hello World";
 
   cout << "var1 变量的地址： ";
   cout << &var1 << "value: " << var1 << endl;
 
   cout << "var2 变量的地址： ";
   cout << &var2 << "value: " << var2 << endl;
 
   return 0;
}
```

# 引用

```c++
#include <iostream>
 
using namespace std;
 
int main ()
{
   // 声明简单的变量
   int    i;
   double d;
 
   // 声明引用变量
   int&    r = i;
   double& s = d;
   
   i = 5;
   cout << "Value of i : " << i << endl;
   cout << "Value of i reference : " << r  << endl;
 
   d = 11.7;
   cout << "Value of d : " << d << endl;
   cout << "Value of d reference : " << s  << endl;
   
   return 0;
}
```

# 抽象类

直接放代码

```c++
#include <iostream>
 
using namespace std;
 
// 基类
class Shape 
{
public:
   // 提供接口框架的纯虚函数
   virtual int getArea() = 0;
   void setWidth(int w)
   {
      width = w;
   }
   void setHeight(int h)
   {
      height = h;
   }
protected:
   int width;
   int height;
};
 
// 派生类
class Rectangle: public Shape
{
public:
   int getArea()
   { 
      return (width * height); 
   }
};
class Triangle: public Shape
{
public:
   int getArea()
   { 
      return (width * height)/2; 
   }
};
 
int main(void)
{
   Rectangle Rect;
   Triangle  Tri;
 
   Rect.setWidth(5);
   Rect.setHeight(7);
   // 输出对象的面积
   cout << "Total Rectangle area: " << Rect.getArea() << endl;
 
   Tri.setWidth(5);
   Tri.setHeight(7);
   // 输出对象的面积
   cout << "Total Triangle area: " << Tri.getArea() << endl; 
 
   return 0;
}
```



# Lambda

C++11 提供了对匿名函数的支持,称为 Lambda 函数(也叫 Lambda 表达式)。

Lambda 表达式把函数看作对象。Lambda 表达式可以像对象一样使用，比如可以将它们赋给变量和作为参数传递，还可以像函数一样对其求值。

Lambda 表达式本质上与函数声明非常类似。Lambda 表达式具体形式如下:

格式如下

```
[capture](parameters)->return-type{body}
```

举例如下:

```cpp
[](int x, int y){ return x < y ; }
```

