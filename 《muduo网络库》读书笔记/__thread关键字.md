# __thread 关键字
`__thread` 是 **GCC 和 Clang 提供的扩展关键字**，用来声明**线程局部存储**（Thread-Local Storage，TLS）变量。简单来说，每个线程都会拥有该变量的一个独立副本，一个线程对它的修改不会影响其他线程。

它在 C11 和 C++11 之前就已经存在，目前仍有大量代码在使用，但新项目更推荐使用标准的 `thread_local`（C++11）或 `_Thread_local`（C11）。

---

### 1. 基本语法

`__thread` 只能修饰**全局变量、文件范围的静态变量和函数内的静态变量**，不能修饰普通的局部变量（自动存储期变量）或类的非静态成员。

```c
__thread int tls_var = 0;                     // 全局 TLS 变量

static __thread int file_static_var = 1;      // 文件作用域 TLS 变量

void func() {
    static __thread int func_static_var = 2;  // 函数内的静态 TLS 变量
    // __thread int local = 3;                // 错误！不能修饰普通局部变量
}
```

每个线程在第一次访问该变量时，都会拥有自己的一份副本，初始值从主线程的初始模板中拷贝。

---

### 2. 典型用法

最常见的场景是替代全局变量来实现线程安全，或为每个线程保存独立的上下文。

```c
#include <stdio.h>
#include <pthread.h>

__thread int thread_id = -1;   // 每个线程独有的 ID

void* worker(void* arg) {
    thread_id = *(int*)arg;
    printf("Thread %d: thread_id = %d\n", thread_id, thread_id);
    return NULL;
}

int main() {
    pthread_t t1, t2;
    int id1 = 1, id2 = 2;
    pthread_create(&t1, NULL, worker, &id1);
    pthread_create(&t2, NULL, worker, &id2);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    return 0;
}
```

输出会显示两个线程各自独立维护 `thread_id`，互不干扰。

---

### 3. 重要限制

- **只支持 “POD” 类型（C 的平凡类型）**  
  `__thread` 变量的初始化必须是**编译期常量**，且不能有用户定义的构造/析构函数。  
  例如在 C++ 中，下面的代码**可能编译失败或产生未定义行为**：
  ```cpp
  __thread std::string str;  // 错误！std::string 不是 POD
  ```
  `__thread` 不会为每个线程运行构造函数，也不会在线程退出时调用析构函数。

- **不能用于需要动态初始化的类型**  
  即使该类型是 POD，若初始值需要运行时计算（例如 `__thread int x = some_func();`），也不符合 `__thread` 的语义，通常会被编译器拒绝。

- **地址使用**  
  可以取 `__thread` 变量的地址，但该地址**仅在线程内有效**。线程退出后，该地址变为悬空指针，绝不能跨线程共享该地址。

- **动态库加载（`dlopen`）的限制**  
  如果在被 `dlopen` 动态加载的共享库中使用 `__thread` 变量，可能因为 TLS 内存模型不支持而失败（通常需要采用“全局动态模型”，并注意可移植性）。主程序和启动时链接的共享库使用 `__thread` 一般是安全的。

---

### 4. 与标准关键字的对比

| 特性 | `__thread` (GCC/Clang) | `thread_local` (C++11) / `_Thread_local` (C11) |
|------|----------------------|----------------------------------------------|
| 标准化 | 否（编译器扩展）    | 是                                           |
| 支持类型 | 仅 POD（静态初始化） | 任意类型，支持动态初始化和析构               |
| 局部变量 | 仅支持静态局部变量   | 支持普通局部变量（自动存储期）               |
| 类成员 | 不能修饰非静态成员   | 可修饰静态成员，不能修饰非静态成员           |
| 延迟初始化 | 否，线程创建时即分配 | 通常在线程首次使用时分配（如同本地静态）     |
| 跨线程地址 | 地址仅本线程有效     | 地址仅本线程有效                             |

由于 `thread_local` 支持非平凡类型和局部变量，**新代码应优先使用 `thread_local`**。  
```cpp
thread_local std::string per_thread_log;  // 安全，每个线程会自动构造和析构
```

---

### 5. 性能与实现原理简述

- `__thread` 变量通常被编译器放入特殊的 `.tdata` 和 `.tbss` 段，每个线程创建时会从“TLS 模板”拷贝一份到自己线程的 TLS 区域。
- 访问 `__thread` 变量通常只有少量指针运算，比 `pthread_getspecific` 快很多，接近普通全局变量的访问速度。
