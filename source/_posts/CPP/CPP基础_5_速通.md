---
title: 【CPP基础】【四】【申请释放】
date: 2025-04-04 16:51:04
tags:cpp
categories:cpp
---
# 完整的C++编程教程

## 目录
1. [开发环境配置](#1-开发环境配置)
2. [C++知识体系](#2-c知识体系)
3. [现代C++特性](#3-现代c特性)
4. [设计模式](#4-设计模式)
5. [数据结构](#5-数据结构)
6. [CMake项目构建](#6-cmake项目构建)
7. [调试技巧](#7-调试技巧)
8. [进阶主题](#8-进阶主题)
9. [学习资源](#9-学习资源)

---

## 1. 开发环境配置
### 1.1 安装编译器
```bash
sudo apt-get install g++ build-essential
```

### 1.2 安装构建工具
```bash
sudo apt-get install cmake
```

### 1.3 VS Code配置
1. 安装C++扩展
2. 配置调试环境
3. 安装CMake Tools扩展

## 2. C++知识体系
### 2.1 基础语法
```cpp
#include <iostream>

int main() {
    std::cout << "Hello World!" << std::endl;
    return 0;
}
```

### 2.2 面向对象编程
```cpp
class MyClass {
public:
    MyClass() {}
    ~MyClass() {}
    
    void print() {
        std::cout << "MyClass instance" << std::endl;
    }
};
```

## 3. 现代C++特性
### 3.1 自动类型推导
```cpp
auto x = 5; // 自动推导为int
auto str = "hello"; // 自动推导为const char*
```

### 3.2 Lambda表达式
```cpp
auto sum = [](int a, int b) { return a + b; };
std::cout << sum(3, 4) << std::endl; // 输出7
```

## 4. 设计模式
### 4.1 单例模式
```cpp
class Singleton {
private:
    Singleton() {}
    
public:
    static Singleton& instance() {
        static Singleton instance;
        return instance;
    }
};
```

#### UML图
```
+------------------+
|    Singleton     |
+------------------+
| - instance: static Singleton |
+------------------+
| + getInstance(): static Singleton& |
+------------------+
```

应用场景：
- 全局配置管理
- 日志系统
- 数据库连接池
- 缓存系统
- 线程池

性能分析：
- 线程安全（C++11后static局部变量初始化是线程安全的）
- 延迟初始化
- 内存效率高
- 首次调用可能有轻微性能开销

现代C++改进：
```cpp
// 使用std::call_once确保线程安全
class ThreadSafeSingleton {
private:
    ThreadSafeSingleton() {}
    static std::once_flag initFlag;
    static std::unique_ptr<ThreadSafeSingleton> instance;
    
public:
    static ThreadSafeSingleton& getInstance() {
        std::call_once(initFlag, []() {
            instance.reset(new ThreadSafeSingleton());
        });
        return *instance;
    }
};
```
### 4.1 单例模式
```cpp
class Singleton {
private:
    Singleton() {}
    
public:
    static Singleton& instance() {
        static Singleton instance;
        return instance;
    }
};
```

#### UML图
```
+------------------+
|    Singleton     |
+------------------+
| - instance: static Singleton |
+------------------+
| + getInstance(): static Singleton& |
+------------------+
```

应用场景：
- 全局配置管理
- 日志系统
- 数据库连接池
- 缓存系统
- 线程池

性能分析：
- 线程安全（C++11后static局部变量初始化是线程安全的）
- 延迟初始化
- 内存效率高
- 首次调用可能有轻微性能开销

现代C++改进：
```cpp
// 使用std::call_once确保线程安全
class ThreadSafeSingleton {
private:
    ThreadSafeSingleton() {}
    static std::once_flag initFlag;
    static std::unique_ptr<ThreadSafeSingleton> instance;
    
public:
    static ThreadSafeSingleton& getInstance() {
        std::call_once(initFlag, []() {
            instance.reset(new ThreadSafeSingleton());
        });
        return *instance;
    }
};
```


### 4.2 工厂模式
```cpp
class Product {
public:
    virtual ~Product() {}
    virtual void operation() = 0;
};

class ConcreteProductA : public Product {
public:
    void operation() override {
        std::cout << "ConcreteProductA operation" << std::endl;
    }
};

class ConcreteProductB : public Product {
public:
    void operation() override {
        std::cout << "ConcreteProductB operation" << std::endl;
    }
};

class Factory {
public:
    virtual Product* createProduct() = 0;
};

class ConcreteFactoryA : public Factory {
public:
    Product* createProduct() override {
        return new ConcreteProductA();
    }
};

class ConcreteFactoryB : public Factory {
public:
    Product* createProduct() override {
        return new ConcreteProductB();
    }
};
```

#### UML图
```
+------------------+       +------------------+
|     Product      |<------| ConcreteProductA |
+------------------+       +------------------+
| + operation(): virtual |  +------------------+
+------------------+       | ConcreteProductB |
                           +------------------+

+------------------+       +------------------+
|     Factory      |<------| ConcreteFactoryA |
+------------------+       +------------------+
| + createProduct(): virtual |  +------------------+
+------------------+       | ConcreteFactoryB |
                           +------------------+
```

应用场景：
- 需要创建多种相似对象
- 对象创建逻辑复杂
- 需要解耦客户端和具体产品类
- 需要动态扩展产品类型
- 需要集中管理对象创建

性能分析：
- 虚函数调用开销
- 对象创建成本
- 内存分配开销

现代C++改进：
```cpp
// 使用智能指针避免内存泄漏
template<typename T>
std::unique_ptr<Product> createProduct() {
    return std::make_unique<T>();
}

// 使用模板工厂
template<typename ProductType>
class TemplateFactory {
public:
    std::unique_ptr<Product> create() {
        return std::make_unique<ProductType>();
    }
};

// 使用lambda工厂
auto productFactory = [](auto&&... args) {
    return std::make_unique<Product>(std::forward<decltype(args)>(args)...);
};
```
```cpp
class Product {
public:
    virtual ~Product() {}
    virtual void operation() = 0;
};

class ConcreteProductA : public Product {
public:
    void operation() override {
        std::cout << "ConcreteProductA operation" << std::endl;
    }
};

class ConcreteProductB : public Product {
public:
    void operation() override {
        std::cout << "ConcreteProductB operation" << std::endl;
    }
};

class Factory {
public:
    virtual Product* createProduct() = 0;
};

class ConcreteFactoryA : public Factory {
public:
    Product* createProduct() override {
        return new ConcreteProductA();
    }
};

class ConcreteFactoryB : public Factory {
public:
    Product* createProduct() override {
        return new ConcreteProductB();
    }
};
```

#### UML图
```
+------------------+       +------------------+
|     Product      |<------| ConcreteProductA |
+------------------+       +------------------+
| + operation(): virtual |  +------------------+
+------------------+       | ConcreteProductB |
                           +------------------+

+------------------+       +------------------+
|     Factory      |<------| ConcreteFactoryA |
+------------------+       +------------------+
| + createProduct(): virtual |  +------------------+
+------------------+       | ConcreteFactoryB |
                           +------------------+
```

应用场景：
- 需要创建多种相似对象
- 对象创建逻辑复杂
- 需要解耦客户端和具体产品类
- 需要动态扩展产品类型
- 需要集中管理对象创建

性能分析：
- 虚函数调用开销
- 对象创建成本
- 内存分配开销

现代C++改进：
```cpp
// 使用智能指针避免内存泄漏
template<typename T>
std::unique_ptr<Product> createProduct() {
    return std::make_unique<T>();
}

// 使用模板工厂
template<typename ProductType>
class TemplateFactory {
public:
    std::unique_ptr<Product> create() {
        return std::make_unique<ProductType>();
    }
};

// 使用lambda工厂
auto productFactory = [](auto&&... args) {
    return std::make_unique<Product>(std::forward<decltype(args)>(args)...);
};
```


### 4.3 观察者模式
```cpp
#include <vector>
#include <algorithm>
#include <memory>
#include <mutex>
#include <functional>
#include <unordered_map>
#include <any>

// 基础观察者接口
class Observer {
public:
    virtual ~Observer() = default;
    virtual void update(const std::string& message) = 0;
};

// 主题/被观察者
class Subject {
private:
    std::vector<std::weak_ptr<Observer>> observers;
    std::mutex mtx;
    
public:
    // 线程安全的观察者注册
    void attach(std::shared_ptr<Observer> obs) {
        std::lock_guard<std::mutex> lock(mtx);
        observers.emplace_back(obs);
    }
    
    // 线程安全的观察者注销
    void detach(std::shared_ptr<Observer> obs) {
        std::lock_guard<std::mutex> lock(mtx);
        observers.erase(
            std::remove_if(observers.begin(), observers.end(),
                [&obs](const std::weak_ptr<Observer>& weakObs) {
                    return weakObs.expired() || weakObs.lock() == obs;
                }),
            observers.end());
    }
    
    // 线程安全的通知所有观察者
    void notify(const std::string& message) {
        std::lock_guard<std::mutex> lock(mtx);
        for (auto it = observers.begin(); it != observers.end(); ) {
            if (auto obs = it->lock()) {
                obs->update(message);
                ++it;
            } else {
                it = observers.erase(it);
            }
        }
    }
};

// 具体观察者实现
class ConcreteObserver : public Observer {
    std::string name;
public:
    explicit ConcreteObserver(std::string name) : name(std::move(name)) {}
    
    void update(const std::string& message) override {
        std::cout << "Observer " << name << " received: " << message << std::endl;
    }
};

// 现代C++改进：使用std::function和lambda
class Observable {
    std::vector<std::function<void(const std::string&)>> observers;
    std::mutex mtx;
    
public:
    void subscribe(std::function<void(const std::string&)> observer) {
        std::lock_guard<std::mutex> lock(mtx);
        observers.push_back(observer);
    }
    
    void unsubscribe(std::function<void(const std::string&)> observer) {
        std::lock_guard<std::mutex> lock(mtx);
        observers.erase(
            std::remove(observers.begin(), observers.end(), observer),
            observers.end());
    }
    
    void notify(const std::string& message) {
        std::lock_guard<std::mutex> lock(mtx);
        for (auto& observer : observers) {
            observer(message);
        }
    }
};

// C++20 协程改进
struct EventAwaiter {
    Observable& observable;
    std::string message;
    bool ready = false;
    
    EventAwaiter(Observable& obs) : observable(obs) {
        observable.subscribe([this](const std::string& msg) {
            message = msg;
            ready = true;
        });
    }
    
    bool await_ready() const { return ready; }
    void await_suspend(std::coroutine_handle<>) {}
    std::string await_resume() { return message; }
};

// 使用示例
Task asyncEventListener(Observable& observable) {
    auto message = co_await EventAwaiter(observable);
    std::cout << "Received async message: " << message << std::endl;
}

// 使用示例
int main() {
    // 传统观察者模式使用
    auto subject = std::make_shared<Subject>();
    auto observer1 = std::make_shared<ConcreteObserver>("Observer1");
    auto observer2 = std::make_shared<ConcreteObserver>("Observer2");
    
    subject->attach(observer1);
    subject->attach(observer2);
    subject->notify("Hello World!");
    
    // 现代C++风格使用
    Observable modernSubject;
    auto callback = [](const std::string& msg) {
        std::cout << "Lambda observer received: " << msg << std::endl;
    };
    
    modernSubject.subscribe(callback);
    modernSubject.notify("Modern C++ message");
    
    return 0;
}
```

#### UML图
```
+------------------+       +------------------+
|     Subject      |<>----->|     Observer     |
+------------------+       +------------------+
| + attach(Observer) |      | + update(): virtual |
| + detach(Observer) |      +------------------+
| + notify()         |               ^
+------------------+                |
                                    |
                         +------------------+
                         | ConcreteObserver |
                         +------------------+
```

应用场景：
- 事件驱动系统
- GUI组件交互
- 发布-订阅系统
- 状态监控
- 数据同步
- 微服务架构中的事件通知
- 物联网设备状态更新
- 金融交易实时监控
- 游戏引擎事件处理
- 分布式系统状态同步

性能优化：
- 使用weak_ptr避免内存泄漏
- 异步通知减少阻塞
- 批量通知优化
- 线程安全实现
- 观察者去重
- 使用对象池管理观察者
- 事件过滤减少不必要通知
- 按优先级分组通知
- 使用无锁数据结构优化高频事件
- 事件合并减少通知频率

现代C++改进：
```cpp
// 使用信号槽库
#include <boost/signals2.hpp>

struct Event {
    void operator()() {
        std::cout << "Event triggered" << std::endl;
    }
};

boost::signals2::signal<void()> signal;
signal.connect(Event());
signal();

// 使用std::function和lambda
class Observable {
    std::vector<std::function<void(const std::string&)>> observers;
public:
    void subscribe(std::function<void(const std::string&)> observer) {
        observers.push_back(observer);
    }
    
    void notify(const std::string& message) {
        for (auto& observer : observers) {
            observer(message);
        }
    }
};
```


## 5. 数据结构
### 5.1 链表实现
```cpp
struct Node {
    int data;
    Node* next;
};

class LinkedList {
    Node* head;
public:
    void insert(int data) {
        Node* newNode = new Node{data, head};
        head = newNode;
    }
};
```

## 6. CMake项目构建
```cmake
cmake_minimum_required(VERSION 3.10)
project(MyProject)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(myapp main.cpp)
```

## 7. 调试技巧
1. 使用gdb调试
2. 打印变量值
3. 设置断点
4. 查看调用栈

## 8. 进阶主题
### 8.1 模板元编程
```cpp
// C++20 概念(Concepts)示例
template<typename T>
concept Numeric = std::is_arithmetic_v<T>;

template<Numeric T>
T square(T x) {
    return x * x;
}

// 编译期字符串处理
constexpr size_t string_length(const char* str) {
    return *str ? 1 + string_length(str + 1) : 0;
}

// C++23 编译期反射提案示例
/*
struct Person {
    std::string name;
    int age;
};

constexpr auto members = reflect(Person);
static_assert(members.size() == 2);
static_assert(members[0].name == "name");
*/
```

应用场景：
- 编译期类型检查
- 领域特定嵌入式语言(EDSL)
- 序列化/反序列化框架
- 高性能数学库
- 编译期数据结构

性能优化技巧：
1. 使用constexpr if减少实例化
2. 模板特化优化热点路径
3. 使用变量模板缓存中间结果
4. 编译期字符串哈希优化查找
5. 使用折叠表达式简化可变参数模板
```cpp
template<int N>
struct Factorial {
    static const int value = N * Factorial<N-1>::value;
};

template<>
struct Factorial<0> {
    static const int value = 1;
};

// 编译期断言
static_assert(Factorial<5>::value == 120, "Factorial calculation error");
```

现代C++改进（C++11/14/17）：
```cpp
// C++11 constexpr函数
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

// C++14 constexpr函数改进
constexpr auto factorial14(auto n) {
    decltype(n) result = 1;
    for (decltype(n) i = 1; i <= n; ++i) {
        result *= i;
    }
    return result;
}

// C++17 变量模板
template<auto N>
constexpr auto factorial17 = N * factorial17<N-1>;

template<>
constexpr auto factorial17<0> = 1;

// C++20 consteval函数
consteval int compile_time_factorial(int n) {
    return n <= 1 ? 1 : n * compile_time_factorial(n - 1);
}
```

应用场景：
- 编译期计算
- 类型特征检查
- 代码生成
- 算法优化
- 领域特定语言(DSL)

性能分析：
- 零运行时开销
- 增加编译时间
- 可能增加二进制大小

高级技巧：
```cpp
// SFINAE (Substitution Failure Is Not An Error)
template<typename T>
auto print_type_info(const T& t) -> decltype(t.toString(), void()) {
    std::cout << t.toString() << std::endl;
}

template<typename T>
auto print_type_info(const T& t) -> decltype(t.to_string(), void()) {
    std::cout << t.to_string() << std::endl;
}

template<typename T>
auto print_type_info(const T& t) -> decltype(std::cout << t, void()) {
    std::cout << t << std::endl;
}

// 类型特征
template<typename T>
struct is_pointer {
    static constexpr bool value = false;
};

template<typename T>
struct is_pointer<T*> {
    static constexpr bool value = true;
};

// 使用if constexpr (C++17)
template<typename T>
auto process(const T& t) {
    if constexpr (is_pointer<T>::value) {
        std::cout << "Pointer to " << *t << std::endl;
    } else {
        std::cout << "Value " << t << std::endl;
    }
}
```


### 8.2 并发编程
```cpp
#include <thread>
#include <iostream>
#include <mutex>
#include <latch>
#include <barrier>
#include <semaphore>

// C++20 新特性示例
void concurrent_operations() {
    // std::latch 一次性屏障
    std::latch work_done(3);
    
    // std::barrier 可重用屏障
    std::barrier sync_point(3);
    
    // std::counting_semaphore 信号量
    std::counting_semaphore<10> sem(3);
    
    auto worker = [&](int id) {
        std::cout << "Worker " << id << " started" << std::endl;
        
        // 模拟工作
        std::this_thread::sleep_for(std::chrono::milliseconds(100 * id));
        
        // 使用信号量
        sem.acquire();
        std::cout << "Worker " << id << " acquired semaphore" << std::endl;
        sem.release();
        
        // 到达同步点
        work_done.count_down();
        sync_point.arrive_and_wait();
        
        std::cout << "Worker " << id << " completed" << std::endl;
    };
    
    std::jthread t1(worker, 1);
    std::jthread t2(worker, 2);
    std::jthread t3(worker, 3);
    
    work_done.wait();
    std::cout << "All workers finished initial phase" << std::endl;
}

#include <thread>
#include <iostream>
#include <mutex>
#include <vector>
#include <future>
#include <condition_variable>
#include <atomic>
#include <queue>

// 基本线程同步
std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void worker_thread() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, []{ return ready; });
    std::cout << "Worker thread is processing data" << std::endl;
}

// 线程安全队列
template<typename T>
class ThreadSafeQueue {
    std::queue<T> queue;
    mutable std::mutex mtx;
    std::condition_variable cv;
    
public:
    void push(T value) {
        std::lock_guard<std::mutex> lock(mtx);
        queue.push(std::move(value));
        cv.notify_one();
    }
    
    bool try_pop(T& value) {
        std::lock_guard<std::mutex> lock(mtx);
        if (queue.empty()) return false;
        value = std::move(queue.front());
        queue.pop();
        return true;
    }
    
    void wait_and_pop(T& value) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [this]{ return !queue.empty(); });
        value = std::move(queue.front());
        queue.pop();
    }
};

// 原子操作
std::atomic<int> counter(0);

void increment_atomic() {
    for (int i = 0; i < 1000; ++i) {
        ++counter;
    }
}

int main() {
    // 基本线程示例
    std::thread worker(worker_thread);
    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }
    cv.notify_one();
    worker.join();

    // 线程池示例
    const unsigned num_threads = std::thread::hardware_concurrency();
    std::vector<std::thread> threads;
    
    // 创建线程池工作函数
    auto worker = [](int id) {
        std::cout << "Thread " << id << " 开始工作" << std::endl;
        
        // 模拟工作负载
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        
        std::cout << "Thread " << id << " 完成工作" << std::endl;
    };
    
    // 启动所有工作线程
    for (unsigned i = 0; i < num_threads; ++i) {
        threads.emplace_back(worker, i);
    }
    
    // 等待所有线程完成
    for (auto& t : threads) {
        if (t.joinable()) {
            t.join();
        }
    }
    
    std::cout << "所有线程任务已完成" << std::endl;
```

### 8.3 内存管理
```cpp
#include <memory>
#include <memory_resource>
#include <vector>

// 内存对齐分配示例 (C++17)
template<typename T>
struct AlignedAllocator {
    using value_type = T;
    
    explicit AlignedAllocator(size_t alignment = alignof(T)) 
        : alignment(alignment) {
        if (alignment & (alignment - 1)) {
            throw std::invalid_argument("Alignment must be power of two");
        }
    }
    
    template<typename U>
    AlignedAllocator(const AlignedAllocator<U>& other) noexcept 
        : alignment(other.alignment) {}
    
    [[nodiscard]] T* allocate(size_t n) {
        if (n > std::numeric_limits<size_t>::max() / sizeof(T)) {
            throw std::bad_alloc();
        }
        
        size_t size = n * sizeof(T);
        void* ptr = std::aligned_alloc(alignment, size);
        if (!ptr) {
            throw std::bad_alloc();
        }
        return static_cast<T*>(ptr);
    }
    
    void deallocate(T* p, size_t n) noexcept {
        std::free(p);
    }
    
    bool operator==(const AlignedAllocator&) const noexcept { return true; }
    bool operator!=(const AlignedAllocator&) const noexcept { return false; }
    
    size_t alignment;
};

// 使用示例
std::vector<int, AlignedAllocator<int>> aligned_vec(AlignedAllocator<int>(64));

// 自定义内存池分配器 (C++11)
template<typename T>
class PoolAllocator {
    struct Block {
        Block* next;
    };
    
    Block* freeList = nullptr;
    
public:
    using value_type = T;
    using propagate_on_container_copy_assignment = std::true_type;
    using propagate_on_container_move_assignment = std::true_type;
    using propagate_on_container_swap = std::true_type;
    
    explicit PoolAllocator(size_t poolSize = 1024) {
        if (poolSize == 0) {
            throw std::invalid_argument("Pool size must be positive");
        }
        
        // 预分配内存池
        freeList = static_cast<Block*>(::operator new(poolSize * sizeof(T)));
        
        // 初始化空闲链表
        Block* current = freeList;
        for (size_t i = 0; i < poolSize - 1; ++i) {
            current->next = reinterpret_cast<Block*>(reinterpret_cast<char*>(current) + sizeof(T));
            current = current->next;
        }
        current->next = nullptr;
    }
    
    PoolAllocator(const PoolAllocator&) = delete;
    PoolAllocator& operator=(const PoolAllocator&) = delete;
    
    [[nodiscard]] T* allocate(size_t n) {
        if (n != 1 || !freeList) {
            throw std::bad_alloc();
        }
        
        Block* block = freeList;
        freeList = freeList->next;
        return reinterpret_cast<T*>(block);
    }
    
    void deallocate(T* p, size_t n) noexcept {
        if (n != 1 || !p) return;
        
        Block* block = reinterpret_cast<Block*>(p);
        block->next = freeList;
        freeList = block;
    }
    
    template<typename U>
    struct rebind {
        using other = PoolAllocator<U>;
    };
};

// 使用示例
std::vector<int, PoolAllocator<int>> vec(PoolAllocator<int>(100));

// 现代C++内存管理特性
/*
1. 智能指针 (C++11):
   - std::unique_ptr: 独占所有权
   - std::shared_ptr: 共享所有权
   - std::weak_ptr: 打破循环引用

2. 内存资源 (C++17):
   - std::pmr::memory_resource
   - std::pmr::polymorphic_allocator
   - 内置内存资源 (monotonic, pool, synchronized)

3. 垃圾回收支持 (C++11):
   - std::declare_reachable
   - std::undeclare_reachable
   - std::declare_no_pointers
   - std::undeclare_no_pointers

4. 内存模型 (C++11):
   - std::atomic
   - 内存顺序约束
   - 线程安全保证
*/
```
```

现代C++内存管理特性：
1. 内存资源(Memory Resources)
```cpp
class MonotonicResource : public std::pmr::memory_resource {
    void* current = nullptr;
    size_t remaining = 0;
    
protected:
    void* do_allocate(size_t bytes, size_t alignment) override {
        // 简单线性分配实现
        void* p = std::align(alignment, bytes, current, remaining);
        if (!p) throw std::bad_alloc();
        current = static_cast<char*>(p) + bytes;
        remaining -= bytes;
        return p;
    }
    
    void do_deallocate(void*, size_t, size_t) override {}
    
    bool do_is_equal(const memory_resource& other) const noexcept override {
        return this == &other;
    }
};
```
2. 多态内存资源
3. 栈分配器(stack allocator)
4. 内存池优化

性能分析工具：
- Valgrind
- AddressSanitizer
- MemorySanitizer
- ThreadSanitizer
- 自定义分配器性能测试方法
```cpp
#include <memory>

class Resource {
public:
    Resource() { std::cout << "Resource acquired" << std::endl; }
    ~Resource() { std::cout << "Resource released" << std::endl; }
};

int main() {
    // 使用智能指针自动管理内存
    auto ptr = std::make_unique<Resource>();
    
    // 移动语义示例
    auto ptr2 = std::move(ptr); // 所有权转移
    
    // 共享所有权
    auto shared = std::make_shared<Resource>();
    
    return 0;
}
```

现代C++特性：
- 移动语义（std::move）
- 完美转发
- RAII原则


## 9. 学习资源
1. 《C++ Primer》
2. 《Effective C++》
3. cppreference.com
4. Stack Overflow C++社区