# C++关键字专项八股

本文档从《C++新八股文》中抽取所有专门讲解 C++ 关键字的面试题，方便集中复习。

---

## 目录

- [new / delete](#new--delete)
- [override / final](#override--final)
- [static / const](#static--const)
- [volatile](#volatile)
- [inline](#inline)
- [explicit](#explicit)
- [class / struct](#class--struct)
- [auto / decltype](#auto--decltype)
- [extern](#extern)

---

## new / delete

### new和malloc的区别？

- **性质**：new是C++操作符，malloc是C语言函数
- **初始化**：new调用构造函数初始化对象，malloc返回未初始化内存
- **语法**：new无需指定大小（如`new int`），malloc需要（如`malloc(sizeof(int))`）
- **返回类型**：new返回具体类型指针，malloc返回void*需强转
- **错误处理**：new抛出std::bad_alloc异常，malloc返回null
- **配对操作**：new配delete，malloc配free
- **实现**：**malloc**的实现核心是通过操作系统提供的系统调用管理堆内存。
  使用分配的内存块头部存储元数据，通过链表链接所有空闲块。
  **new**是C++运算符，其行为包含内存分配和对象构造两阶段，
  内存分配阶段调用全局operator new函数，默认实现内部调用malloc。
  对象构造阶段用placement new在已分配内存上调用构造函数。
  new直接返回响应的数据类型的指针。

---

## override / final

### override和final关键字的作用？

> 原文档此处仅有标题，暂无内容。

---

## static / const

### 介绍一下static和const？

**Const**：指定语义约束，告诉编译器对象不能被改变，编译器强制检查。

- 可修饰：普通对象（局部/全局）、函数返回值/参数、指针本身/指针指向对象、类成员变量/函数

**Static**：声明静态成员变量、静态成员函数、静态局部变量、静态全局变量。

- **静态成员**：属于类而非对象，所有对象共享，无需对象即可调用
- **静态局部变量**：函数内声明，程序运行期间只初始化一次
- **静态全局变量**：仅在定义文件内可见，避免命名冲突

---

## volatile

### volatile关键字的作用和适用场景？

**作用**：防止编译器优化，强制每次从内存直接读写变量。不保证线程安全，原子操作推荐用atomic。

**适用场景**：

- 硬件寄存器访问：硬件寄存器的值可能被外部设备随时修改（如传感器、GPIO 状态）
- 中断服务程序（ISR）与主程序共享变量：中断可能异步修改变量（如标志位），主程序需感知最新值
- 多线程环境中的简单标志位：用于线程间通知（如退出标志），但不保证线程安全
- 防止空循环被优化

---

## inline

### C++ inline内联的作用？

**作用**：将代码复制到调用处，消除函数调用开销，提高性能但会导致代码膨胀。

**C++17新特性**：允许多次定义，不同文件中同名inline函数可实现不同功能（类似static）。

---

## explicit

### explicit关键字是什么作用？

explicit 关键字在 C++ 中用于禁止**编译器进行隐式类型转换**，强制要求开发者显式调用构造函数或转换运算符，提高代码的安全性和可读性。

- 禁止隐式类型转换
- 禁止拷贝初始化
- 防止隐式转换链

---

## class / struct

### class和struct的区别？

- class的默认成员和继承都是**private**的，如果要存储一些内部使用的成员变量推荐使用class,因为内部的一些数据不希望被外部随意获取。

- struct默认是**public**的，如果是要给外部提供一些所需的数据可以使用struct。

---

## auto / decltype

### C++类型推导的作用和用法？

**auto**:变量类型推导。会丢失引用和cv语义，用`auto&`保留。万能引用`auto&&`根据初始值推导左/右值引用。

**decltype**：推导表达式类型，保留所有信息（引用、cv限定符）。

```c++
int a = 10;
decltype(a) b = 20;     // b 为 int
decltype(a + 3.14) c;   // c 为 double[5]
```

---

## extern

### C++中Static全局变量，全局变量，和extern变量的区别

核心区别在于**链接属性**：

| 类型           | 链接属性 | 作用域   | 存储位置   | 说明                     |
| -------------- | -------- | -------- | ---------- | ------------------------ |
| 全局变量       | 外部链接 | 整个程序 | 静态存储区 | 其他文件可通过extern访问 |
| static全局变量 | 内部链接 | 仅本文件 | 静态存储区 | 不同文件可同名，互不干扰 |
| extern声明     | -        | -        | 不分配空间 | 声明外部变量，不是定义   |

```c++
// a.cpp
int g_var = 1;           // 全局变量，外部链接
static int s_var = 2;    // static全局变量，仅a.cpp可见

// b.cpp
extern int g_var;        // 声明a.cpp中的g_var，可访问
extern int s_var;        // 链接错误，s_var是内部链接
```
