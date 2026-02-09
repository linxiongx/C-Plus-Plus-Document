## inline 的作用

### 1. **建议内联（inline expansion）**

- 建议编译器将函数调用处替换为函数体代码

  

- 减少函数调用开销（但编译器可以忽略这个建议）

  

- 现代编译器会自动优化，这个作用已不太重要

  

### 2. **允许多重定义（ODR relaxation）**

这是 `inline` 最重要的现代作用！

**One Definition Rule (ODR)** 规定：

- 普通函数/变量只能在整个程序中定义一次

  

- 但 `inline` 函数/变量可以在多个编译单元中定义

  

- 只要所有定义完全相同，链接器会合并它们

  

### 3. **使用场景**

+ (静态/非静态)成员函数可以使用 inline 修饰 

  

+ 非静态数据成员不能使用 inline 修饰 

  + `inline` 解决的是“同一个实体在多个翻译单元中定义”的问题，而非静态数据成员既不是独立实体，也不存在链接层面的定义冲突，因此语言层面禁止对其使用 `inline`
    
    
  
+ C++17 起静态数据成员可以使用 inline 修饰
  + 在 C++17 之前，静态数据成员需要在类外定义。在 C++17 之后，静态数据成员可以在类内定义，对 header-only 友好



### 4. C++17 **inline 全局变量**

### 1. 基本语法（合法）

```
// header.h
inline int g_value = 42;
// a.cpp
#include "header.h"

// b.cpp
#include "header.h"
```

✔ **整个程序中只有一个 `g_value` 实例**
 ✔ 不会违反 ODR（One Definition Rule）

------

### 2. inline 全局变量解决了什么问题？

这是为了解决 **“头文件中定义全局变量会产生多重定义”** 的老问题。

### 传统做法（C++17 之前）

```
// header.h
extern int g_value;

// source.cpp
int g_value = 42;
```

### C++17 之后（更简洁）

```
// header.h
inline int g_value = 42;   // 一行搞定
```

------

### 3. inline 全局变量的本质语义

`inline` 修饰变量时，**含义与函数类似**：

> **允许在多个翻译单元中出现定义，但链接时合并为同一个实体**

它解决的是 **链接层面的问题**，而不是“内联优化”。

------

### 4. 和 static 全局变量的区别

### static 全局变量

```
// header.h
static int g_value = 42;
```

- 每个 `.cpp` 都有 **自己的一份副本**
- 变量地址不同
- 不共享状态

### inline 全局变量

```
inline int g_value = 42;
```

- 整个程序 **只有一份**
- 地址相同
- 全局共享状态
