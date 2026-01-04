---
sidebar_position: 2
tags: [design-pattern, singleton, cpp, thread-safe]
---

# 单例模式 (Singleton Pattern)

> 保证一个类只有一个实例，并提供全局访问点。

## 📋 概述

**用途：** 控制类的实例化，确保全局只有一个实例存在。

**使用场景：**
- 配置管理器
- 日志系统
- 资源管理器
- 线程池

**核心优势：**
- 控制实例数量
- 延迟初始化
- 全局访问点

---

## 💡 核心实现

### 线程安全的单例（C++11+）

```cpp
class Singleton {
public:
    // 删除拷贝构造和赋值操作
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
    
    // 获取单例实例（线程安全）
    static Singleton& GetInstance() {
        static Singleton instance;  // C++11保证线程安全
        return instance;
    }
    
    void DoSomething() {
        // 业务逻辑
    }

private:
    Singleton() = default;  // 私有构造函数
    ~Singleton() = default;
};
```

### 关键要点

1. **线程安全：** C++11的静态局部变量保证线程安全的初始化
2. **防止拷贝：** 删除拷贝构造函数和赋值运算符
3. **延迟初始化：** 第一次调用GetInstance()时才创建实例

---

## 🎯 实际应用示例

### 配置管理器示例

```cpp
class ConfigManager {
public:
    static ConfigManager& GetInstance() {
        static ConfigManager instance;
        return instance;
    }
    
    void LoadConfig(const std::string& filepath) {
        // 加载配置
        m_ConfigPath = filepath;
    }
    
    std::string GetConfigValue(const std::string& key) {
        // 返回配置值
        return m_ConfigData[key];
    }
    
    ConfigManager(const ConfigManager&) = delete;
    ConfigManager& operator=(const ConfigManager&) = delete;

private:
    ConfigManager() = default;
    std::string m_ConfigPath;
    std::map<std::string, std::string> m_ConfigData;
};

// 使用
int main() {
    auto& config = ConfigManager::GetInstance();
    config.LoadConfig("config.ini");
    std::string value = config.GetConfigValue("ServerIP");
    return 0;
}
```

---

## ⚠️ 注意事项

### 潜在问题

1. **析构顺序：** 多个单例相互依赖时，析构顺序不可控
2. **测试困难：** 全局状态使单元测试复杂化
3. **隐藏依赖：** 使用单例隐藏了类之间的依赖关系

### 替代方案

- 依赖注入（Dependency Injection）
- 服务定位器模式（Service Locator）

---

## 🔗 相关模式

- Factory Pattern（工厂模式）
- Monostate Pattern（单态模式）

---

## 📚 面试要点

**常见问题：**
1. 如何实现线程安全的单例模式？
2. 单例模式有什么缺点？
3. C++11之前如何保证线程安全？

**回答要点：**
- C++11静态局部变量自动线程安全
- 使用双检锁（Double-Checked Locking）和内存屏障
- 注意拷贝构造和赋值的删除
- 了解饿汉式vs懒汉式初始化
- 可能带来的测试和维护问题
