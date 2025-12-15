### ✅ Lambda 捕获机制（针对上面讨论内容）

1. **捕获 this**

   - 捕获当前对象指针；
   - lambda 内可访问成员变量；

2. **捕获初始化（C++14 起）**

   ```
   [pLog = this->pLog](CString msg) { pLog->ErrorMsg(msg); };
   ```

   - 捕获 `this->pLog` 的当前值（即指针的副本）；
   - lambda 内使用自己的 `pLog`；
   - 不依赖 `this`，更安全。

3. **为什么不能写 `[this->pLog]`**

   - 捕获列表里只能是变量名或初始化语句；
   - `this->pLog` 是一个**表达式**，不是一个独立变量；
   - 因此语法上不合法，会报错“不是变量”。