# share_ptr：析构动作在创建时被捕获
> shared_ptr 不只存储了一个裸指针，它还一并"捕获"并存储了如何销毁这个对象的全部信息。当对象在创建时，它的"死法"就已经被安排好了。
### 核心思想：将"如何死"与"是什么"绑定
在 C++ 里，一个普通的指针（比如 Foo*）只告诉了你"这里有块内存"，但没告诉你要怎么释放它。而 shared_ptr<Foo> 则不同：

* 捕获析构动作：当 shared_ptr 被创建时（比如通过 new 或者 make_shared），它会记住一个删除器（Deleter）。这个删除器知道如何正确地销毁它所指向的具体对象。

* 类型信息被擦除：这个删除器的类型和具体逻辑，是作为 shared_ptr 内部"控制块"的一部分存储的，并不会暴露在其模板类型 T 中。

也就是说，shared_ptr 把"对象类型"和"销毁方式"解耦了，并将后者在对象诞生的那一刻就绑定好了。

### 好处
* 虚析构不再是必需品：即使基类的析构函数不是虚函数，通过基类的 shared_ptr 来删除派生类对象也是安全的，因为正确的删除器（派生类的析构）在创建时就被捕获并存储了。
* shared_ptr<void> 也能安全释放：你可以创建一个 shared_ptr<void> 来指向任何对象，它仍然知道如何正确销毁该对象。
* 安全地跨越模块边界：在动态库（如 DLL）中创建对象，将它的 shared_ptr 返回给主程序使用。即便主程序不知道这个对象的具体大小和析构函数，shared_ptr 内部已捕获的、来自动态库的删除器也能安全地释放它，避免了在不同模块间因内存管理方式不同而导致的错误。

### 弊端与处理方式
* 潜在风险：如果析构非常耗时，又恰好在关键的业务线程中触发了，就会拖慢该线程。
* 优化方法：正因为它"捕获"了析构动作，我们可以将这个动作转移。例如，创建一个 BlockingQueue<shared_ptr<void>> 专门用来回收资源。需要销毁对象时，把它 push 进队列，然后由一个专门的"析构线程"取出并清空这些 shared_ptr，从而将昂贵的析构开销从关键路径上剥离出去。

### 三种智能指针的"析构捕获"对比
| 智能指针 | 析构动作何时捕获？ | 能否自定义删除器？ | 类型擦除？ |
| :---: | :--- | :---: | :--- |
| **`std::shared_ptr`** | ✅ 创建时（构造时绑定） | ✅ 支持 | ✅ 是（删除器类型被擦除） |
| **`std::unique_ptr`** | ⚠️ 编译时（模板参数） | ✅ 支持 | ❌ 否（删除器是类型的一部分） |
| **原始指针 `T*`** | ❌ 不捕获 | ❌ 不支持 | ❌ 不适用 |

### 使用 share_ptr 虚析构不再是必须的
#### 案例：
```c++
#include <iostream>
using namespace std;

class Base {
public:
    Base() { cout << "Base构造" << endl; }
    // 注意：此处析构函数没有 virtual
    ~Base() { cout << "Base析构" << endl; }
};

class Derived : public Base {
public:
    Derived() { cout << "Derived构造" << endl; }
    ~Derived() { cout << "Derived析构" << endl; }
};

int main() {
    Base* ptr = new Derived(); // 基类指针指向派生类对象
    delete ptr;                // 危险操作！
    return 0;
}
```
#### 输出：
```
Base构造
Derived构造
Base析构
```
#### 后果：
Derived 的析构函数没有被调用！这意味着 Derived 类中申请的所有资源（堆内存、文件句柄、网络连接等）全部泄漏。
####  正确的做法：
> 1. 加 **virtual**，只需在基类析构函数前加上 virtual，一切恢复正常：
```cpp
class Base {
public:
    virtual ~Base() { cout << "Base析构" << endl; } // 加 virtual
};
```
> 2. 使用 **share_ptr**
```cpp
// 构造时类型是 Derived
shared_ptr<Base> sp = make_shared<Derived>(); 
```
#### 重新运行：
```
Base构造
Derived构造
Derived析构   // 先清理派生类
Base析构      // 再清理基类
```
#### 注意！用 shared_ptr<Base> 直接构造（危险！）
```cpp
Base* raw = new Derived();
shared_ptr<Base> sp(raw); // 此时 shared_ptr 只知道类型是 Base
// 或者从已有的 shared_ptr<Base> 转型
```
在这种情况下，shared_ptr 的内部删除器只知道销毁一个 Base 对象。如果析构函数非虚，Derived 的析构函数不会被调用，导致内存泄漏
#### 结论：
因为 make_shared<Derived> 记住了对象的原始类型是 Derived，所以即使基类析构非虚，Derived 的析构函数依然会被调用。这是 shared_ptr 比裸指针安全的地方。