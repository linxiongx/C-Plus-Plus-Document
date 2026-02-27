<p align="center" style="font-size: 32px;font-weight: bold;">shared_ptr 的构造方式</p>

# 1. `std::shared_ptr` 的构造方式（构造函数）

`shared_ptr` 本质上是一个 **智能指针对象**，它内部维护两块东西：

- **指向资源的指针（T\*)**
- **控制块（control block）**：引用计数、deleter、weak count 等

------

## 1.1 用裸指针构造（最常见）

```
std::shared_ptr<int> p(new int(10));
```

这会发生：

- `new int(10)` 分配一块堆内存（对象）
- `shared_ptr` 内部再分配一块堆内存（控制块）

即：**两次堆分配**

------

## 1.2 用 deleter 构造（指定释放方式）

```
std::shared_ptr<FILE> fp(fopen("a.txt", "r"), fclose);
```

此时 `shared_ptr` 的控制块里保存了 `fclose` 作为 deleter。

------

## 1.3 用数组构造（C++17 才支持）

```
std::shared_ptr<int[]> arr(new int[10]);
```

注意：这不是 `shared_ptr<int>`，而是 `shared_ptr<int[]>`。

------

## 1.4 用 `nullptr` 构造

```
std::shared_ptr<int> p(nullptr);
```

表示空指针。

------

## 1.5 拷贝构造（引用计数 +1）

```
std::shared_ptr<int> p1 = std::make_shared<int>(5);
std::shared_ptr<int> p2(p1);
```

p2 和 p1 共享同一对象，同一控制块。

------

## 1.6 移动构造（转移所有权）

```
std::shared_ptr<int> p1 = std::make_shared<int>(5);
std::shared_ptr<int> p2(std::move(p1));
```

p1 变为空，p2 接管资源。

------

## 1.7 aliasing 构造（高级用法）

这个构造函数非常重要：

```
template<class Y>
shared_ptr(const shared_ptr<Y>& r, element_type* ptr) noexcept;
```

用法：

```
auto p = std::make_shared<std::vector<int>>(100);
std::shared_ptr<int> p2(p, &(*p)[0]);
```

含义：

- `p2` 指向 vector 内部元素
- 但生命周期由 `p` 的控制块管理

即：**共享控制块，但指向不同地址**

------

# 2. `std::make_shared` 是什么？

`make_shared<T>(args...)` 是一个函数模板：

```
auto p = std::make_shared<MyClass>(arg1, arg2);
```

它会：

- **一次性分配**：控制块 + 对象内存（通常在一块连续内存中）
- 在这块内存上构造 `T`

即：**一次堆分配**

------

# 3. `shared_ptr(new T)` vs `make_shared<T>()` 核心区别

| 对比点             | `shared_ptr<T>(new T(...))` | `make_shared<T>(...)` |
| ------------------ | --------------------------- | --------------------- |
| 堆分配次数         | 2 次（对象 + 控制块）       | 1 次（合并分配）      |
| 性能               | 较慢                        | 更快                  |
| 内存碎片           | 更严重                      | 更少                  |
| 异常安全           | 有潜在风险                  | 更安全                |
| 控制块和对象布局   | 分开                        | 通常连续              |
| 支持自定义 deleter | 支持                        | 不直接支持            |

------

# 4. 异常安全问题（重要）

## 4.1 问题的核心：参数求值顺序的不确定性

看下面这种写法：

```cpp
foo(std::shared_ptr<int>(new int(5)), bar());
```

这行代码看起来很简单，但在 **C++17 之前**可能导致内存泄漏。

------

## 4.2 为什么会泄漏？拆解执行步骤

### C++17 之前的危险顺序

编译器在求值 `foo()` 的两个参数时，可能按以下顺序执行：

**危险的执行顺序**：

1. 执行 `new int(5)` → 分配内存，返回裸指针（比如地址 `0x1234`）
2. 执行 `bar()` → 调用函数 `bar()`
3. 用步骤1的裸指针构造 `std::shared_ptr<int>`

**如果 `bar()` 抛出异常会发生什么？**

```
步骤1: new int(5) ✅ 成功，内存已分配（地址 0x1234）
步骤2: bar()      ❌ 抛出异常！
步骤3: shared_ptr 构造 ⚠️ 永远不会执行
```

**结果**：

- `new int(5)` 分配的内存**永远不会被释放**（没有指针指向它）
- `shared_ptr` 还没构造完成，无法自动管理
- **内存泄漏！**

### 为什么会这样？

在 C++17 之前，标准对**函数参数的求值顺序未作规定**，编译器可以自由选择：

```cpp
// 可能的顺序1（安全）
1. new int(5)
2. 构造 shared_ptr<int>
3. bar()

// 可能的顺序2（危险）
1. new int(5)
2. bar()  // 如果这里抛异常，shared_ptr 还没构造
3. 构造 shared_ptr<int>
```

------

## 4.3 C++17 的改进

### 新的求值顺序保证

C++17 引入了更严格的规则：

> **函数参数的求值顺序是不确定的，但每个参数的求值必须完整完成后，才能开始求值下一个参数。**

具体到 `foo(A, B)` 形式：

- 要么先完整求值 A（包括所有子表达式），再求值 B
- 要么先完整求值 B，再求值 A
- 但**不允许交错求值**（A的一半 → B → A的另一半）

### 应用到我们的例子

```cpp
foo(std::shared_ptr<int>(new int(5)), bar());
```

C++17 之后，只能是：

**顺序1**：

```
1. new int(5)
2. 构造 shared_ptr<int>(步骤1的指针)  ← 完整求值完第一个参数
3. bar()
```

**顺序2**：

```
1. bar()                               ← 完整求值完第二个参数
2. new int(5)
3. 构造 shared_ptr<int>
```

**无论哪种顺序**，`new int(5)` 和 `shared_ptr` 构造都是原子性的整体，不会被 `bar()` 打断。

所以：

- 如果 `bar()` 在顺序1中抛异常 → `shared_ptr` 已构造完成，会自动释放
- 如果 `bar()` 在顺序2中抛异常 → `new int(5)` 还没执行，没有泄漏

------

## 4.4 为什么仍然推荐 `make_shared`？

虽然 C++17 修复了这个问题，但 `make_shared` 仍然更优：

### 1. 兼容性 - 支持 C++11/14 代码库

如果你的项目需要支持旧标准，用 `make_shared` 是最保险的。

### 2. 没有裸指针暴露 - 更符合现代 C++ 理念

```cpp
// 不好：裸指针在栈上短暂存在
foo(std::shared_ptr<int>(new int(5)), bar());

// 好：没有裸指针
foo(std::make_shared<int>(5), bar());
```

### 3. 性能更好 - 一次分配 vs 两次分配

```cpp
// 两次堆分配
std::shared_ptr<int>(new int(5))
// 分配1: new int(5) → 对象内存
// 分配2: shared_ptr 构造 → 控制块内存

// 一次堆分配
std::make_shared<int>(5)
// 分配1: 对象 + 控制块 在同一块连续内存中
```

### 4. 代码更简洁

```cpp
auto p = std::make_shared<MyClass>(arg1, arg2, arg3);
// vs
auto p = std::shared_ptr<MyClass>(new MyClass(arg1, arg2, arg3));
```

------

## 4.5 总结

- **C++17 之前**：函数参数求值顺序未定义，可能导致异常安全问题
- **C++17 之后**：保证参数求值不交错，异常安全性改善
- **最佳实践**：无论哪个标准，都优先用 `make_shared`

**推荐写法**：

```cpp
foo(std::make_shared<int>(5), bar());  // ✅ 任何标准都安全
```

------

## 5. `make_shared` 的一个缺点（weak_ptr + 内存释放延迟）

因为 `make_shared` 把对象和控制块放在同一块内存中：

- `shared_ptr` 引用计数归零时，对象析构
- 但如果还有 `weak_ptr` 存在，控制块不能释放
- 由于控制块和对象在同一块内存中，这块内存也无法释放

因此可能出现现象：

> 对象析构了，但内存还占着（直到 weak_ptr 也销毁）

如果你需要对象析构后立刻释放内存，`shared_ptr(new T)` 分配分离可能更适合。

------

## 6. 什么时候用 `make_shared`？

✅ 推荐默认使用：

```
auto p = std::make_shared<MyClass>(...);
```

原因：更快、更省内存、更安全。

------

## 7. 什么时候必须用 `shared_ptr(new T)`？

典型场景：

## 7.1 需要自定义 deleter

```
std::shared_ptr<FILE> fp(fopen("a.txt", "r"), fclose);
```

## 7.2 对象不能在连续块中构造（很少见）

例如某些特殊内存池/对齐控制。

## 7.3 想控制对象与控制块分离释放

如前面 weak_ptr 的内存释放延迟问题。

------

## 8. 总结一句话

- `std::shared_ptr` 构造函数：**你给它一个指针，它给你创建控制块**
- `std::make_shared`：**它帮你创建对象 + 控制块（一次分配），并返回 shared_ptr**

默认推荐：

> **优先用 `make_shared`，除非你需要自定义 deleter 或特殊内存控制。**