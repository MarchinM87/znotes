---
sidebar_position: 1
tags: [cpp, memory, smart-pointer, raii, interview]
---

# 智能指针 (Smart Pointers)

> C++11引入的自动内存管理工具，遵循RAII原则。

## 📋 概述

**用途：** 自动管理动态分配的内存，避免内存泄漏。

**三种智能指针：**
- `std::unique_ptr` - 独占所有权
- `std::shared_ptr` - 共享所有权（引用计数）
- `std::weak_ptr` - 弱引用（不增加引用计数）

---

## 💡 核心实现

### unique_ptr - 独占所有权

```cpp
#include <memory>
#include <iostream>

class Resource {
public:
    Resource() { std::cout << "Resource acquired\n"; }
    ~Resource() { std::cout << "Resource destroyed\n"; }
    void Use() { std::cout << "Resource in use\n"; }
};

void UniquePointerExample() {
    // 创建unique_ptr
    std::unique_ptr<Resource> ptr1 = std::make_unique<Resource>();
    ptr1->Use();
    
    // 转移所有权（移动语义）
    std::unique_ptr<Resource> ptr2 = std::move(ptr1);
    // ptr1现在为nullptr
    
    if (!ptr1) {
        std::cout << "ptr1 is null\n";
    }
    
    ptr2->Use();
    // 离开作用域，自动释放资源
}
```

### shared_ptr - 共享所有权

```cpp
void SharedPointerExample() {
    std::shared_ptr<Resource> ptr1 = std::make_shared<Resource>();
    std::cout << "Count: " << ptr1.use_count() << "\n";  // 1
    
    {
        std::shared_ptr<Resource> ptr2 = ptr1;  // 复制，引用计数+1
        std::cout << "Count: " << ptr1.use_count() << "\n";  // 2
        ptr2->Use();
    }  // ptr2离开作用域，引用计数-1
    
    std::cout << "Count: " << ptr1.use_count() << "\n";  // 1
    // 最后一个shared_ptr销毁时，资源被释放
}
```

### weak_ptr - 解决循环引用

```cpp
class Node {
public:
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;  // 使用weak_ptr打破循环
    int value;
    
    Node(int v) : value(v) {
        std::cout << "Node " << value << " created\n";
    }
    ~Node() {
        std::cout << "Node " << value << " destroyed\n";
    }
};

void WeakPointerExample() {
    auto node1 = std::make_shared<Node>(1);
    auto node2 = std::make_shared<Node>(2);
    
    node1->next = node2;
    node2->prev = node1;  // weak_ptr不增加引用计数
    
    // 使用weak_ptr
    if (auto prev = node2->prev.lock()) {  // 转换为shared_ptr
        std::cout << "Previous node value: " << prev->value << "\n";
    }
    // 正常释放，无内存泄漏
}
```

---

## 🎯 实际应用场景

### 工厂函数返回unique_ptr

```cpp
class GameObject {
public:
    virtual ~GameObject() = default;
    virtual void Update() = 0;
};

class Player : public GameObject {
public:
    void Update() override {
        // 玩家更新逻辑
    }
};

std::unique_ptr<GameObject> CreatePlayer() {
    return std::make_unique<Player>();
}
```

### 观察者模式中使用weak_ptr

```cpp
class Observer {
public:
    virtual void OnNotify() = 0;
    virtual ~Observer() = default;
};

class Subject {
private:
    std::vector<std::weak_ptr<Observer>> observers;
    
public:
    void RegisterObserver(std::shared_ptr<Observer> obs) {
        observers.push_back(obs);
    }
    
    void NotifyAll() {
        // 清理已经失效的观察者
        observers.erase(
            std::remove_if(observers.begin(), observers.end(),
                [](const std::weak_ptr<Observer>& wp) { 
                    return wp.expired(); 
                }),
            observers.end()
        );
        
        // 通知所有有效观察者
        for (auto& wp : observers) {
            if (auto sp = wp.lock()) {
                sp->OnNotify();
            }
        }
    }
};
```

---

## ⚠️ 注意事项

### 性能考虑

1. **shared_ptr开销：** 引用计数需要原子操作，有轻微性能开销
2. **make_shared优势：** 一次分配，对象和控制块连续存储
3. **避免裸指针混用：** 不要用裸指针和智能指针同时管理同一对象

### 常见错误

```cpp
// ❌ 错误：用同一个裸指针创建多个shared_ptr
Resource* raw = new Resource();
std::shared_ptr<Resource> ptr1(raw);
std::shared_ptr<Resource> ptr2(raw);  // 错误！会重复删除

// ✅ 正确：使用make_shared
auto ptr = std::make_shared<Resource>();

// ❌ 错误：循环引用导致内存泄漏
class Node {
    std::shared_ptr<Node> next;
    std::shared_ptr<Node> prev;  // 应该用weak_ptr
};

// ✅ 正确：使用weak_ptr打破循环
class Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;
};
```

---

## 📊 对比表格

| 特性 | unique_ptr | shared_ptr | weak_ptr |
|------|-----------|-----------|----------|
| 所有权 | 独占 | 共享 | 不拥有 |
| 可复制 | ❌ | ✅ | ✅ |
| 可移动 | ✅ | ✅ | ✅ |
| 引用计数 | 无 | 有 | 不增加计数 |
| 开销 | 最小 | 中等 | 小 |
| 线程安全 | 否* | 引用计数是 | 引用计数是 |

*对象本身不是线程安全的，但所有权转移是安全的

---

## 📚 面试要点

**常见问题：**

1. **unique_ptr和shared_ptr的区别？**
   - unique_ptr独占所有权，不可复制只能移动
   - shared_ptr共享所有权，使用引用计数

2. **如何解决shared_ptr的循环引用问题？**
   - 使用weak_ptr打破循环
   - weak_ptr不增加引用计数，需要lock()转换

3. **为什么优先使用make_shared而不是直接new？**
   - 一次内存分配（对象+控制块）
   - 异常安全
   - 性能更好

4. **智能指针是线程安全的吗？**
   - 引用计数的修改是线程安全的（原子操作）
   - 对象本身的访问不是线程安全的
   - 多线程需要额外同步

5. **何时使用裸指针？**
   - 不涉及所有权时（观察者模式）
   - 性能关键路径
   - 与C API交互

**代码题示例：**
```cpp
// 实现一个简化的unique_ptr
template<typename T>
class MyUniquePtr {
private:
    T* ptr;
public:
    explicit MyUniquePtr(T* p = nullptr) : ptr(p) {}
    ~MyUniquePtr() { delete ptr; }
    
    MyUniquePtr(const MyUniquePtr&) = delete;
    MyUniquePtr& operator=(const MyUniquePtr&) = delete;
    
    MyUniquePtr(MyUniquePtr&& other) noexcept : ptr(other.ptr) {
        other.ptr = nullptr;
    }
    
    MyUniquePtr& operator=(MyUniquePtr&& other) noexcept {
        if (this != &other) {
            delete ptr;
            ptr = other.ptr;
            other.ptr = nullptr;
        }
        return *this;
    }
    
    T* operator->() { return ptr; }
    T& operator*() { return *ptr; }
};
```
