---
title: 【CPP基础】【二】【设计模式】
tags: cpp
categories: cpp
abbrlink: 45254
date: 2024-11-17 21:58:49
---
# 前言
我其实是想把指针当做我学习的第二部分，奈何指针很麻烦，智能指针一时半会也说不完.
于是先整理下设计模式
因为我在实际项目中开发用到不少的工厂模式，就先以工厂模式为主要的研究内容开始进行复习和拓展。



设计模式是一种在软件开发中常用的解决方案，旨在帮助开发者解决常见问题，提高软件的可维护性、可重用性和可扩展性。设计模式通常分为三大类：创建型模式、结构型模式和行为型模式。

1. **创建型模式**：这些模式主要关注对象的创建机制，以适应不同的需求和场景。
   - **单例模式**：确保一个类只有一个实例，并提供一个全局访问点。
   - **工厂方法模式**：定义一个创建对象的接口，让子类决定实例化哪一个类。
   - **抽象工厂模式**：提供一个创建一系列相关或相互依赖对象的接口，而无需指定它们的具体类。

2. **结构型模式**：这些模式主要关注对象之间的组合关系。
   - **适配器模式**：将一个类的接口转换成客户端期望的另一个接口，从而使不兼容的接口能够合作。
   - **装饰者模式**：动态地给一个对象添加额外的职责，就增加功能来说，这种模式比生成子类更为灵活。
   - **代理模式**：为其他对象提供一种代理以控制对这个对象的访问。

3. **行为型模式**：这些模式主要关注对象之间的交互和职责分配。
   - **观察者模式**：定义了一种一对多的依赖关系，使得一当一个对象改变状态时，所有依赖于它的对象都得到通知并被自动更新。
   - **策略模式**：定义一系列算法，把它们一个个封装起来，并且使它们可以互相替换，策略模式让算法的变化独立于使用算法的客户。
   - **命令模式**：将一个请求封装成一个对象，从而使你能够使用不同的请求、队列或日志请求，并支持可撤销的操作。

设计模式是软件开发的宝贵经验积累，通过运用这些模式，可以有效地解决许多普遍存在的问题，使代码更加简洁、清晰和易于维护。

# 单例模式

这个在我是实际开发中用到的比较多

单例模式是一种创建型设计模式，它的主要目的是确保一个类只有一个实例，并提供一个全局访问点。这种模式在需要控制对某些资源的访问时尤其有用，比如数据库连接或配置管理等。

### 特点：

1. **唯一性**：单例模式确保任何时候都只有一个类的实例存在。
2. **全局访问**：提供一个静态方法，让用户可以获取到这个唯一实例。
3. **延迟初始化**：可以实现懒汉式单例，只有在第一次使用时才创建实例。

### 实现方式：

单例模式的实现方式有多种，以下是最常见的几种：

1. **懒汉式**：
   - 在需要时才创建实例，在多线程环境下需要加锁以保证线程安全。
   -  完整的代码如下:
```cpp
#include <iostream>
#include <mutex>

class Singleton {
private:
    static Singleton* instance;
    static std::mutex mutex;

    // 私有构造函数
    Singleton() 
    {
        std::cout << "Singleton created" << std::endl;
    }

public:
    // 获取单例实例的方法
    static Singleton* getInstance() 
    {
        if (instance == nullptr) 
        {
            std::lock_guard<std::mutex> lock(mutex); // 加锁以保证线程安全
            if (instance == nullptr) 
            {
                instance = new Singleton();
            }
        }
        return instance;
    }
};

// 初始化静态成员变量
Singleton* Singleton::instance = nullptr;
std::mutex Singleton::mutex;

int main() 
{
    Singleton* singleton = Singleton::getInstance();
    return 0;
}
```
2. **饿汉式**：
   - 在类加载时就创建实例，线程安全，但不支持延迟加载。
   ```cpp
   #include <iostream>
   
   class Singleton
   {
   public:
       // 提供一个静态方法来获取单例对象
       static Singleton &getInstance()
       {
           static Singleton instance; // 在第一次调用时创建实例
           return instance;
       }
   
       // 删除拷贝构造函数和赋值运算符，以防意外复制
       Singleton(const Singleton &) = delete;
       Singleton &operator=(const Singleton &) = delete;
   
       // 示例方法
       void someMethod()
       {
           std::cout << "Hello from Singleton!" << std::endl;
       }
   
   private:
       // 私有构造函数
       Singleton()
       {
           std::cout << "Singleton instance created!" << std::endl;
       }
   
       // 私有析构函数
       ~Singleton()
       {
           std::cout << "Singleton instance destroyed!" << std::endl;
       }
   };
   
   int main()
   {
       // 获取单例对象并调用方法
       Singleton::getInstance().someMethod();
   
       return 0;
   }
   
   ```
   
3. **双重检查锁定**：
   
   - 在懒汉式的基础上，使用双重检查加锁，减少了同步的性能开销。
   ```cpp
   #include <iostream>
   #include <mutex>
   
   class Singleton
   {
   public:
       // 提供一个静态方法来获取单例对象
       static Singleton *getInstance()
       {
           // 首次检查
           if (!instance)
           {
               // 对共享资源加锁
               std::lock_guard<std::mutex> lock(mutex_);
               // 再次检查
               if (!instance)
               {
                   instance = new Singleton(); // 创建实例
               }
           }
           return instance;
       }
   
       // 删除拷贝构造函数和赋值运算符，以防意外复制
       Singleton(const Singleton &) = delete;
       Singleton &operator=(const Singleton &) = delete;
   
       // 示例方法
       void someMethod()
       {
           std::cout << "Hello from Singleton!" << std::endl;
       }
   
   private:
       // 私有构造函数
       Singleton()
       {
           std::cout << "Singleton instance created!" << std::endl;
       }
   
       // 私有析构函数
       ~Singleton()
       {
           std::cout << "Singleton instance destroyed!" << std::endl;
       }
   
       static Singleton *instance; // 单例指针
       static std::mutex mutex_;   // 互斥锁
   };
   
   // 静态成员变量初始化
   Singleton *Singleton::instance = nullptr;
   std::mutex Singleton::mutex_;
   
   int main()
   {
       // 获取单例对象并调用方法
       Singleton::getInstance()->someMethod();
   
       return 0;
   }
   ```


### 使用场景：
- 配置管理：一个应用程序通常只需要一个配置对象来管理配置参数。
- 日志记录：通常使用单例模式来管理日志记录，以确保日志的统一性。
- 数据库连接或线程池：避免创建多个连接，节省资源。

### 总结：
单例模式通过控制实例的创建，提供了对共享资源的管理，避免了资源浪费和状态不一致的问题。在实际应用中，选择合适的实现方式以满足线程安全、性能和资源利用的需求非常重要。

# 工厂模式

工厂模式是一种常见的设计模式，其主要目的是通过创建一个工厂来集中创建对象。工厂模式的主要类型包括：

1. **简单工厂模式**：
   - 定义一个工厂类，根据传入的参数决定创建哪一种产品类的实例。虽然简单工厂模式本身不是GoF设计模式，但它在很多项目中得到了广泛应用。

2. **工厂方法模式**：
   - 定义一个接口用于创建对象，但将实例化的工作推迟到子类中。每个子类都实现了自己的工厂方法，负责创建特定类型的对象。

3. **抽象工厂模式**：
   - 提供一个接口，用于创建一系列相关或依赖的对象，而不需要指定具体类。这种模式通常用于创建一组相关产品，适用于产品族的设计。

4. **静态工厂方法**：
   - 工厂方法被定义为静态方法，可以在没有创建工厂类实例的情况下调用。通常用于简单的对象创建操作。

这些工厂模式各有优缺点，根据具体需求选择合适的模式可以提高代码的可维护性和可扩展性。


### 1. 简单工厂模式

简单工厂模式使用一个工厂类根据给定的信息返回不同类型的对象。这个模式不推荐用于复杂的系统，因为它违反了开闭原则，但对于简单的应用场景非常有用。

#### 示例代码

```cpp
#include <iostream>
#include <string>

// 产品接口
class Product {
public:
    virtual void use() = 0; // 纯虚函数
};

// 具体产品A
class ConcreteProductA : public Product {
public:
    void use() override {
        std::cout << "使用产品A" << std::endl;
    }
};

// 具体产品B
class ConcreteProductB : public Product {
public:
    void use() override {
        std::cout << "使用产品B" << std::endl;
    }
};

// 简单工厂
class SimpleFactory {
public:
    static Product* createProduct(const std::string& type) {
        if (type == "A") {
            return new ConcreteProductA();
        } else if (type == "B") {
            return new ConcreteProductB();
        }
        return nullptr;
    }
};

int main() {
    Product* productA = SimpleFactory::createProduct("A");
    productA->use();
    delete productA;

    Product* productB = SimpleFactory::createProduct("B");
    productB->use();
    delete productB;

    return 0;
}
```

### 2. 工厂方法模式

工厂方法模式定义一个用于创建产品的接口，但由子类来决定实例化哪一个产品。这样可以更好地遵循开闭原则。

#### 示例代码

```cpp
#include <iostream>

// 产品接口
class Product {
public:
    virtual void use() = 0;
};

// 具体产品A
class ConcreteProductA : public Product {
public:
    void use() override {
        std::cout << "使用产品A" << std::endl;
    }
};

// 具体产品B
class ConcreteProductB : public Product {
public:
    void use() override {
        std::cout << "使用产品B" << std::endl;
    }
};

// 工厂接口
class Creator {
public:
    virtual Product* factoryMethod() = 0;

    void someOperation() {
        Product* product = factoryMethod();
        product->use();
        delete product;
    }
};

// 具体工厂A
class ConcreteCreatorA : public Creator {
public:
    Product* factoryMethod() override {
        return new ConcreteProductA();
    }
};

// 具体工厂B
class ConcreteCreatorB : public Creator {
public:
    Product* factoryMethod() override {
        return new ConcreteProductB();
    }
};

int main() {
    Creator* creatorA = new ConcreteCreatorA();
    creatorA->someOperation();
    delete creatorA;

    Creator* creatorB = new ConcreteCreatorB();
    creatorB->someOperation();
    delete creatorB;

    return 0;
}
```

### 3. 抽象工厂模式

抽象工厂模式提供一个接口，用于创建一系列相关或相互依赖的对象，而无需指定具体的类。这种模式非常适合用于需要创建多个不同类型的产品的场景。

#### 示例代码

```cpp
#include <iostream>

// 产品接口
class ProductA {
public:
    virtual void use() = 0;
};

class ProductB {
public:
    virtual void use() = 0;
};

// 具体产品A1
class ConcreteProductA1 : public ProductA {
public:
    void use() override {
        std::cout << "使用产品A1" << std::endl;
    }
};

// 具体产品A2
class ConcreteProductA2 : public ProductA {
public:
    void use() override {
        std::cout << "使用产品A2" << std::endl;
    }
};

// 具体产品B1
class ConcreteProductB1 : public ProductB {
public:
    void use() override {
        std::cout << "使用产品B1" << std::endl;
    }
};

// 具体产品B2
class ConcreteProductB2 : public ProductB {
public:
    void use() override {
        std::cout << "使用产品B2" << std::endl;
    }
};

// 抽象工厂接口
class AbstractFactory {
public:
    virtual ProductA* createProductA() = 0;
    virtual ProductB* createProductB() = 0;
};

// 具体工厂1
class ConcreteFactory1 : public AbstractFactory {
public:
    ProductA* createProductA() override {
        return new ConcreteProductA1();
    }

    ProductB* createProductB() override {
        return new ConcreteProductB1();
    }
};

// 具体工厂2
class ConcreteFactory2 : public AbstractFactory {
public:
    ProductA* createProductA() override {
        return new ConcreteProductA2();
    }

    ProductB* createProductB() override {
        return new ConcreteProductB2();
    }
};

int main() {
    AbstractFactory* factory1 = new ConcreteFactory1();
    ProductA* productA1 = factory1->createProductA();
    ProductB* productB1 = factory1->createProductB();
    
    productA1->use();
    productB1->use();
    
    delete productA1;
    delete productB1;
    delete factory1;

    AbstractFactory* factory2 = new ConcreteFactory2();
    ProductA* productA2 = factory2->createProductA();
    ProductB* productB2 = factory2->createProductB();
    
    productA2->use();
    productB2->use();
    
    delete productA2;
    delete productB2;
    delete factory2;

    return 0;
}
```

### 总结

- **简单工厂模式**适合用于简单的对象创建，便于使用和管理。
- **工厂方法模式**通过子类化来实现扩展功能，更符合面向对象的原则。
- **抽象工厂模式**能够创建一系列相关的产品，适用性更广。

这些工厂模式在软件开发中非常常见，尤其在需要解耦对象创建与使用逻辑时，能够有效提高代码的可维护性和可扩展性。

