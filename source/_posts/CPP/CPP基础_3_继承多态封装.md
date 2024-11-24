---
title: 【CPP基础】【三】【继承多态封装】
date: 2024-11-24 13:33:54
tags: cpp
categories: cpp
---
在C++中，继承、多态和封装是面向对象编程的三大基本特性。下面我将分别介绍这三者，并给出相应的示例。

# 1. 封装（Encapsulation）

封装是指将数据（成员变量）和操作数据的方法（成员函数）放在一起，形成一个类，通过访问控制来限制对类内部数据的直接访问，从而保护数据的完整性。

**示例：**

```cpp
#include <iostream>
using namespace std;

class BankAccount {
private:
    double balance; // 私有成员变量

public:
    BankAccount(double initialBalance) {
        balance = initialBalance;
    }

    void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
        }
    }

    void withdraw(double amount) {
        if (amount > 0 && amount <= balance) {
            balance -= amount;
        }
    }

    double getBalance() const {
        return balance;
    }
};

int main() {
    BankAccount account(1000);
    account.deposit(500);
    account.withdraw(200);
    cout << "当前余额: " << account.getBalance() << endl; // 输出: 当前余额: 1300
    return 0;
}
```

# 2. 继承（Inheritance）

继承是指一个类可以从另一个类中继承成员变量和方法，从而实现代码的重用。子类（派生类）可以扩展或重写父类（基类）的行为。

**示例：**

```cpp
#include <iostream>
using namespace std;

class Animal {
public:
    void speak() {
        cout << "动物发声" << endl;
    }
};

class Dog : public Animal { // Dog类继承自Animal类
public:
    void speak() { // 重写父类的方法
        cout << "汪汪" << endl;
    }
};

int main() {
    Animal animal;
    animal.speak(); // 输出: 动物发声

    Dog dog;
    dog.speak(); // 输出: 汪汪
    return 0;
}
```

# 3. 多态（Polymorphism）

多态是指同一个操作可以作用于不同的对象上，不同的对象可以根据其具体类型表现出不同的行为。C++中的多态通常通过虚函数实现。

**示例：**

```cpp
#include <iostream>
using namespace std;

class Shape {
public:
    virtual void draw() { // 虚函数
        cout << "绘制形状" << endl;
    }
};

class Circle : public Shape {
public:
    void draw() override { // 重写父类的虚函数
        cout << "绘制圆形" << endl;
    }
};

class Square : public Shape {
public:
    void draw() override { // 重写父类的虚函数
        cout << "绘制正方形" << endl;
    }
};

void render(Shape* shape) { // 接受基类指针
    shape->draw(); // 调用虚函数
}

int main() {
    Circle circle;
    Square square;

    render(&circle); // 输出: 绘制圆形
    render(&square); // 输出: 绘制正方形

    return 0;
}
```

# 总结

- 封装通过访问控制保护数据，提供了安全性。
- 继承实现了代码的复用，允许子类扩展父类的功能。
- 多态让程序更加灵活，同一操作可以作用于多个类型，增强了代码的可扩展性和可维护性。