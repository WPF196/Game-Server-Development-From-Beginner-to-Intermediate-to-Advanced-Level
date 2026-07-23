# 为什么要有 std::function
**`std::function` 的本质是"可调用对象的类型擦除容器"**——它能存储**任何**可以调用的东西（函数指针、函数对象、Lambda、成员函数等，只要函数签名（参数类型、返回类型）一致即可），并提供统一的调用接口。

为什么要这么麻烦，而不是直接调用函数？因为**直接调用是编译期绑定，`std::function` 是运行期绑定**，两者适用的场景完全不同。

---

## 1. 直接调用 vs std::function

### 直接调用：编译期绑定
```cpp
void foo() { std::cout << "foo" << std::endl; }
void bar() { std::cout << "bar" << std::endl; }

int main() {
    foo();  // 编译期就确定了：调用 foo()
    bar();  // 编译期就确定了：调用 bar()
}
```
**特点**：
- ✅ 零开销（直接跳转）
- ❌ 固定死的，运行时无法改变
- ❌ 不能存储、不能传递、不能延迟执行

### std::function：运行期绑定
```cpp
void foo() { std::cout << "foo" << std::endl; }
void bar() { std::cout << "bar" << std::endl; }

int main() {
    std::function<void()> func;  // 空容器
    
    func = foo;   // 运行时决定装 foo
    func();       // 输出 foo
    
    func = bar;   // 运行时改成 bar
    func();       // 输出 bar
    
    // 甚至可以装 Lambda
    func = [] { std::cout << "lambda" << std::endl; };
    func();       // 输出 lambda
}
```
**特点**：
- ✅ 灵活：运行时可以换不同的东西
- ✅ 可存储、可传递、可延迟执行
- ❌ 有一点运行时开销（虚函数调用）

---

## 2. 核心价值：类型擦除（Type Erasure）

**`std::function` 能存不同类型的东西，但用统一的方式调用**。

### 没有 std::function 的痛苦

```cpp
// 三种不同的可调用类型
void freeFunc(int x) { /* ... */ }

struct Functor {
    void operator()(int x) { /* ... */ }
};

auto lambda = [](int x) { /* ... */ };

// 想统一存储？做不到！
// 只能用不同的容器分别存储
vector<void(*)(int)> funcPtrs;   // 只能存函数指针
vector<Functor> functors;        // 只能存函数对象
// lambda 有自己的类型，没办法存到 vector 里
```

### 有了 std::function 的优雅

```cpp
// 统一容器，装什么都行！
std::vector<std::function<void(int)>> callbacks;

callbacks.push_back(freeFunc);     // 函数指针
callbacks.push_back(Functor());    // 函数对象
callbacks.push_back([](int x) {    // Lambda
    std::cout << x << std::endl;
});

// 统一调用
for (auto& cb : callbacks) {
    cb(42);
}
```

---

## 3. 典型应用场景

### ① 回调函数（最常用）

```cpp
// muduo 中的回调机制
class TcpServer {
public:
    // 用户注册回调
    void setConnectionCallback(std::function<void(const TcpConnectionPtr&)> cb) {
        connectionCallback_ = cb;
    }
    
private:
    std::function<void(const TcpConnectionPtr&)> connectionCallback_;
};

// 用户使用
server.setConnectionCallback([this](const TcpConnectionPtr& conn) {
    // 你也不知道 muduo 内部什么时候调用，但一定会调用
    onConnection(conn);
});
```

**关键价值**：muduo 不需要知道用户的具体类型，只需保存一个 `std::function` 对象。

### ② 延迟执行 / 命令模式

```cpp
class TaskQueue {
    std::queue<std::function<void()>> tasks_;
    
public:
    void addTask(std::function<void()> task) {
        tasks_.push(task);
    }
    
    void processAll() {
        while (!tasks_.empty()) {
            auto task = tasks_.front();
            tasks_.pop();
            task();  // 延迟执行
        }
    }
};

// 使用
TaskQueue queue;
queue.addTask([] { std::cout << "Task 1" << std::endl; });
queue.addTask([] { std::cout << "Task 2" << std::endl; });
// 稍后统一执行
queue.processAll();
```

### ③ 策略模式（运行时替换算法）

```cpp
class DataProcessor {
    std::function<int(int, int)> strategy_;
    
public:
    void setStrategy(std::function<int(int, int)> strategy) {
        strategy_ = strategy;
    }
    
    int process(int a, int b) {
        return strategy_(a, b);
    }
};

// 使用
DataProcessor processor;
processor.setStrategy([](int a, int b) { return a + b; });  // 加法策略
processor.process(10, 20);  // 30

processor.setStrategy([](int a, int b) { return a * b; });  // 乘法策略
processor.process(10, 20);  // 200
```

### ④ 与 `std::bind` 配合

```cpp
std::function<void(int)> func = std::bind(&MyClass::memberFunc, &obj, _1);
func(42);  // 调用 obj.memberFunc(42)
```

---

## 4. std::function 的底层实现

`std::function` 内部使用了**类型擦除 + 虚函数**技术：

```cpp
// 简化版实现
template<typename Signature>
class function;

template<typename Ret, typename... Args>
class function<Ret(Args...)> {
private:
    struct CallableBase {
        virtual Ret invoke(Args...) = 0;
        virtual ~CallableBase() = default;
    };
    
    template<typename F>
    struct CallableImpl : CallableBase {
        F f;
        CallableImpl(F f) : f(std::move(f)) {}
        Ret invoke(Args... args) override {
            return f(std::forward<Args>(args)...);
        }
    };
    
    std::unique_ptr<CallableBase> base_;
    
public:
    template<typename F>
    function(F f) : base_(new CallableImpl<F>(std::move(f))) {}
    
    Ret operator()(Args... args) {
        return base_->invoke(std::forward<Args>(args)...);
    }
};
```

**开销来源**：
- 虚函数调用（间接跳转）
- 动态内存分配（可能）
- 类型擦除的代价

---

## 5. 什么时候用 std::function？什么时候直接调用？

| 场景 | 用什么 | 原因 |
|------|--------|------|
| **API 设计（回调）** | `std::function` | 用户需要灵活注入自己的逻辑 |
| **存储多种可调用对象** | `std::function` | 类型擦除是唯一选择 |
| **延迟执行** | `std::function` | 需要把"动作"作为对象存储 |
| **高性能热路径** | 直接调用 / 模板 | 避免虚函数开销 |
| **已知具体类型** | 直接调用 | 不需要灵活性 |
| **模板函数** | 模板参数 `typename F` | 零开销抽象，编译期确定 |

### 性能对比示例

```cpp
#include <chrono>
#include <functional>

void directCall() {
    // 编译期确定，零开销
    for (int i = 0; i < 10000000; ++i) {
        foo(i);
    }
}

void functionCall() {
    std::function<void(int)> func = foo;
    // 每次调用都有虚函数开销
    for (int i = 0; i < 10000000; ++i) {
        func(i);
    }
}

void templateCall() {
    // 模板：零开销，但编译期确定
    callWithFoo<foo>();
}
```

---

## 6. std::function 的替代方案

| 方案 | 灵活性 | 性能 | 使用场景 |
|------|--------|------|---------|
| **直接调用** | ❌ 固定 | 最快 | 已知具体函数 |
| **函数指针** | ⚠️ 只能存函数 | 很快 | C 风格回调 |
| **模板参数** | ⚠️ 编译期确定 | 最快 | 泛型代码 |
| **std::function** | ✅ 任意可调用对象 | 有开销 | API 设计、回调 |
| **Lambda + auto** | ✅ 但类型被擦除 | 快 | 现代 C++ |

---

## 7. 总结

>* **`std::function` 的价值不是"替代直接调用"，而是"让调用变得更灵活"——它允许你将"行为"作为一等公民，像普通对象一样传递、存储、延迟执行。**
>* 直接调用是"我知道现在要做什么"，`std::function` 是"我不知道现在谁来做，但我知道它一定能做"——前者是实现，后者是抽象。在框架设计、回调机制、事件驱动等场景中，你需要的是抽象，而不是具体实现。
