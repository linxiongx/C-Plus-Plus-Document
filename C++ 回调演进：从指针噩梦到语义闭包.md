# C++ 回调演进：从指针噩梦到语义闭包

## 一、旧时代的痛点（C++11 之前）

### 1. C 风格函数指针：上下文缺失

C++

```c++
// 定义
typedef void (*FuncPtr)(void* context);

// 痛点：必须手动透传 void* context 才能找回对象状态
void my_callback(void* context) {
    MyClass* obj = static_cast<MyClass*>(context);
    obj->doSomething();
}
```

- **局限**：类型不安全（`void*` 强转），无法直接调用成员函数。

### 2. 成员函数指针：语法极其晦涩

成员函数指针 (`void (A::*fp)()`) 最大的问题在于它**不能独立存在**，必须绑定一个实例对象才能调用。

C++

```c++
class A { public: void fun() {} };

// 极其反直觉的定义与调用
void (A::*fp)() = &A::fun;
A* obj = new A();
(obj->*fp)(); // 必须有 obj 参与
```

**工程中的致命缺陷：**

- **接口强耦合**：回调接口必须写死类名 `A`（如 `void setCallback(A* obj, ...)`），导致该接口无法被类 `B` 复用。
- **无法适配**：无法让一个接口同时接受“普通函数”和“成员函数”。

------

## 二、现代标准方案（C++11 及以后）

### 1. 通用多态封装：`std::function`

`std::function` 是通过**类型擦除（Type Erasure）**技术实现的通用容器。

C++

```c++
// 接口定义：我不关心你是谁，我只关心你能被 invoke，且签名是 void()
using Callback = std::function<void()>;

void setCallback(Callback cb);
```

### 2. 核心胶水：Lambda 表达式

Lambda 的本质是编译器自动生成的**仿函数对象（Functor）**，也就是**闭包（Closure）**。

C++

```c++
// 现代写法：捕获 this 指针
setCallback([this]() {
    this->fun(); 
});
```

#### 为什么 Lambda 能彻底替代 `A::*fp`？

请看 Lambda 的**底层等价实现**：

C++

```c++
// [this](){ this->fun(); } 
// 编译器其实生成了这样一个匿名类：
struct __Lambda_Generated_Name {
    A* __this; // 1. 捕获列表：自动保存了对象指针（状态）

    void operator()() const {
        __this->fun(); // 2. 函数体：内部执行成员调用
    }
};
```

**结论**：成员函数指针需要的“对象 + 函数”，已经被 Lambda 封装在了一个独立的结构体内部。对外表现看来，它就是一个普通的无参函数。

------

## 三、已被淘汰的过渡方案：`std::bind`

在 C++11 早期，常使用 `std::bind` 将对象和成员函数绑定：

C++

```c++
// 不推荐：代码冗长，且难以被编译器内联优化
setCallback(std::bind(&A::fun, this));
```

**现状**：C++14 之后，**Lambda 全面优于 bind**（可读性更高、参数处理更灵活、性能更好），`std::bind` 已基本不再建议使用。

------

## 四、工程实践指南

### 1. 推荐的接口定义

使用 `using` 增加可读性。

C++

```c++
using EventCallback = std::function<void(int status)>;

class Button {
public:
    void setOnClick(EventCallback cb) { m_cb = std::move(cb); }
private:
    EventCallback m_cb;
};
```

### 2. 必须注意的“生命周期”陷阱

当 Lambda 捕获 this 时，如果回调执行时对象已销毁，会导致 Crash。

解决方案：使用 weak_ptr 守卫

C++

```c++
// 假设当前类继承自 std::enable_shared_from_this
std::weak_ptr<MyClass> weak_self = shared_from_this();

button.setOnClick([weak_self](int status) {
    // 尝试提升为 shared_ptr
    if (auto shared_self = weak_self.lock()) {
        shared_self->handleClick(status);
    } else {
        // 对象已销毁，安全忽略或打日志
        printf("Target object is dead.\n");
    }
});
```

### 3. 性能敏感场景的取舍

`std::function` 虽好，但有代价：

1. **虚函数开销**：内部通过虚表调用，无法内联。
2. **内存分配**：如果 Lambda 捕获变量较多（超过 16-32 字节，视编译器实现而定），会触发堆内存分配（Heap Allocation）。

高性能替代方案（模板）：

如果是编写极其底层的库（如 STL 算法），为了极致性能，通常使用模板泛型，而非 std::function。

C++

```c++
// 编译器期静态绑定，零开销，但代码膨胀
template <typename Func>
void execute(Func&& func) {
    func();
}
```

------

## 五、总结

| **维度**     | **旧方式 (void\* / A::\*)** | **现代方式 (std::function + Lambda)** |
| ------------ | --------------------------- | ------------------------------------- |
| **耦合度**   | 强耦合特定类或需手动转换    | **完全解耦**，只看函数签名            |
| **状态管理** | 需人工传递 `context`        | **自动捕获**（闭包机制）              |
| **安全性**   | 容易空悬指针或类型转换错误  | 配合 `weak_ptr` 可完美管理生命周期    |
| **适用场景** | C 语言接口兼容、极老旧代码  | **99% 的现代 C++ 业务开发**           |

