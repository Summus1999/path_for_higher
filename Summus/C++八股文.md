<h1 style="text-align:center">C++八股文</h1>

## C++面向对象的三大特性

**封装**：隐藏实现细节，实现模块化。控制访问权限，private仅对自己和友元开放，protected开放给子类，public开放给所有对象。
**继承**：无需修改原有类的情况下实现对功能的扩展。存在三种继承，即private继承，protected继承和public继承，解决基类在子类中最高权限的问题(即基类中是public，子类中为private，则权限在子类中修改为private级别，也可以使用using去修改权限)，还可以做多继承和接口继承。
**多态**：一个接口多种形态，通过实现接口重用增加可扩展性。分为静态多态和动态多态。

- 静态多态：函数重载
- 动态多态：通过虚函数重写

## C++多态的实现

C++ 多态分为静态多态和动态多态。

**静态多态**发生在编译期,主要包括函数重载和模板。  
- 函数重载: 同一作用域下,函数名相同,参数列表不同,编译器在编译期根据实参类型/数量选择合适的重载版本。  
- 函数模板: 使用类型参数编写一份通用代码,编译器在编译期根据实参类型对模板进行实例化,生成不同的函数实现。

**动态多态**发生在运行期,通过虚函数实现。  
典型用法是:基类中将成员函数声明为 `virtual`, 派生类重写该函数, 并通过基类指针或引用调用, 这样在运行时会根据对象的实际类型选择调用哪个版本。

其本质是**晚绑定**:  
- 非虚函数在编译期就确定调用地址(早绑定);  
- 虚函数通过虚函数表(vtable)实现, 对象中有一个虚表指针(vptr), 在构造时初始化, 运行时通过 vptr 在虚表中查找实际要调用的函数地址。

## C++ 指针和引用的区别？

**引用的本质**：引用是对象的别名,从语义上看“不是一个独立对象”。  
在大多数实现中,编译器会用一个隐藏指针来实现引用,因此它通常占用与指针相同的空间,但这属于实现细节,标准并不强制,一般不在代码中依赖 `sizeof(引用)` 的具体值。

指针与引用的区别:

- 指针本身是一个对象,有自己的地址,可以赋值、拷贝,也可以指向不同的对象,还能为 `nullptr`。
- 引用在定义时**必须初始化**,并且一旦绑定某个对象后就不能再改为引用其他对象,通常也不允许“空引用”。
- 使用指针需要显式解引用 `*p` 访问目标对象;引用则可以像普通变量一样直接使用。
- 语义上,指针适合表示“可选/可变的指向关系”(可以为空、可以重指向),引用适合表示“必须存在的别名”,常用于函数参数和返回值以避免拷贝。

## C++ 的构造函数能否定义为虚函数？

不能,语法上就不允许给构造函数加 `virtual`。

原因简述:

- 虚函数依赖虚函数表 (vtable) 和虚表指针 (vptr) 实现运行时多态。vptr 是在对象构造过程中由构造函数负责初始化的,也就是说,**对象还没完全构造好之前,就谈“根据动态类型做虚派发”在语义上说不通**。
- 构造顺序是自上而下: 先调用基类构造函数,再依次调用派生类构造函数。在执行基类构造函数时,对象只能被当作“基类子对象”来看待,此时即使有虚派发,也只会调用基类版本,达不到“根据派生类型选择构造函数”的效果。

因此 C++ 标准直接禁止“虚构造函数”。如果需要“多态创建对象”,通常通过**工厂函数/虚 clone 接口**等模式来实现,而不是让构造函数变成虚函数。

## 智能指针有没有了解? shared_ptr 和 unique_ptr 讲一下

智能指针是基于 RAII 封装裸指针的类对象,在构造时获取资源,在析构时自动释放资源,避免忘记 delete、异常路径泄露等问题。

- `shared_ptr` : 共享所有权,内部维护引用计数。每次拷贝计数 +1,销毁或 reset 计数 -1,为 0 时自动释放对象。计数本身是线程安全的,但多线程同时读写同一对象仍需加锁。
- `unique_ptr` : 独占所有权,禁止拷贝,只支持移动。同一时间只能有一个 `unique_ptr` 指向该对象,适合唯一拥有者场景,开销最小。
- `weak_ptr` : 不参与所有权,从 `shared_ptr` 构造,只做“旁观者”。不会增加引用计数,常用于解决 `shared_ptr` 之间的循环引用,需要访问时通过 `lock()` 临时升级为 `shared_ptr`。

Tips: `auto_ptr` 已在 C++11 中废弃, C++17 中移除。

## 对锁有没有了解,介绍一下？

常见 C++ 锁类型:

- `std::mutex`               : 最基本的互斥锁,不可重入。
- `std::recursive_mutex`     : 可重入互斥锁,同一线程可多次加锁。
- `std::timed_mutex`         : 带超时的互斥锁,支持 `try_lock_for/try_lock_until`。
- `std::recursive_timed_mutex`: 可重入 + 支持超时。
- `std::shared_mutex`        : 读写锁,支持多读单写,适合读多写少场景。
- `std::shared_timed_mutex`  : 在 `shared_mutex` 基础上增加超时功能。

```c++
std::string s1 = "Hello";
std::string s2 = std::move(s1);  // 移动构造，s1 变为空
```

- forward:在模板中完美转发参数，保持原始值类别（左值/右值）。仅在模板中使用（通常是通用引用 T&&）。根据模板参数T的类型决定转发为左值还是右值.

```c++
template <typename T, typename Arg>
T create(Arg&& arg) {
    return T(std::forward<Arg>(arg));  // 保持 arg 的原始值类别
}

std::string s = "Test";
auto a = create<std::string>(s);       // 传递左值 → 调用拷贝构造
auto b = create<std::string>("Temp");  // 传递右值 → 调用移动构造
```

## C++的栈容器的内部是什么样的，内存是否连续？

C++的栈stack不是独立容器，是基于其他的序列容器的。默认是使用deque双端队列实现。

- deque内存连续，由多个固定大小内存块组成
- vector：完全连续，是开辟了一大块大的内存块用于使用
- list：非连续，双向链表节点分散存储

## shared_ptr引用计数的原理是什么？什么时候增加引用计数，什么时候减少引用计数？

**核心原理**：内部维护计数器跟踪指向资源的shared_ptr数量，计数为0时自动释放资源。

**引用计数增加**：创建新shared_ptr、拷贝构造、拷贝赋值时。

**引用计数减少**：对象析构（离开作用域）、reset()、重新赋值时。

## 讲一下循环引用如何发生的，以及如何解决？

对象相互引用时（如双向链表、图结构），会导致引用计数无法归零，资源无法释放。使用weak_ptr打破循环引用，因为它不增加引用计数。

```c++
#include <iostream>
#include <memory>

// 前向声明
class NodeB;

class NodeA {
public:
    // 使用 shared_ptr 会导致循环引用
    std::shared_ptr<NodeB> b_ptr;
    // 解决方案：使用 weak_ptr 替代
    // std::weak_ptr<NodeB> b_ptr;
    
    ~NodeA() { std::cout << "NodeA 销毁\n"; }
};

class NodeB {
public:
    std::shared_ptr<NodeA> a_ptr;
    ~NodeB() { std::cout << "NodeB 销毁\n"; }
};

int main() {
    // 创建两个节点
    auto a = std::make_shared<NodeA>();
    auto b = std::make_shared<NodeB>();
    
    // 建立相互引用
    a->b_ptr = b;  // shared_ptr 版本会导致循环引用
    b->a_ptr = a;
    std::cout << "a 引用计数: " << a.use_count() << "\n";
    std::cout << "b 引用计数: " << b.use_count() << "\n";
    
    // main 结束时，a 和 b 应该被销毁...
    // 但如果使用 shared_ptr，由于循环引用，引用计数不会归零
    // 只有使用 weak_ptr 才能正确销毁
    return 0;
}
```

## shared_ptr是线程安全的吗？多线程中使用智能指针要注意什么？

- 多线程代码操作的是同一个shared_ptr的对象是线程不安全的。
- 多线程代码操作的不是同一个shared_ptr的对象，但不同的shared_ptr指向了相同的内存，此时是线程安全的。

## 何时用shared_ptr，何时用weak_ptr?

**shared_ptr**：需要共享资源所有权时使用。

**weak_ptr**：作为shared_ptr的辅助指针，用于两种场景：
- 打破循环引用
- 需要访问共享资源但不影响其生命周期

## 介绍⼀下static和const

**Const**：指定语义约束，告诉编译器对象不能被改变，编译器强制检查。
- 可修饰：普通对象（局部/全局）、函数返回值/参数、指针本身/指针指向对象、类成员变量/函数

**Static**：声明静态成员变量、静态成员函数、静态局部变量、静态全局变量。
- **静态成员**：属于类而非对象，所有对象共享，无需对象即可调用
- **静态局部变量**：函数内声明，程序运行期间只初始化一次
- **静态全局变量**：仅在定义文件内可见，避免命名冲突

## new和malloc的区别

- **性质**：new是C++操作符，malloc是C语言函数
- **初始化**：new调用构造函数初始化对象，malloc返回未初始化内存
- **语法**：new无需指定大小（如`new int`），malloc需要（如`malloc(sizeof(int))`）
- **返回类型**：new返回具体类型指针，malloc返回void*需强转
- **错误处理**：new抛出std::bad_alloc异常，malloc返回null
- **配对操作**：new配delete，malloc配free

## 什么是左值？什么是右值？

- 左值⼀般是指向⼀个指定内存的，具有名称的值，它通常拥有⼀个稳定的内存地址，并且有⼀段较长时间的声明周期。左值能取到地址。
- 右值通常是不指向稳定内存地址的匿名值，声明周期很短，通常是暂时的。基于此特性，可以用取地址符来判断，右值不能取到地址。

## C和C++的区别？

 C++是C加上⼀些⾯向对象的特性。最初C++只是C加上⼀些⾯向对象的特性，但随着语⾔的发展，C++⽀持了更多观念和特性，变得⽐C语⾔更具有弹性和灵活性。现在的C++相⽐C，是⼀个语⾔联邦，它包含了C语⾔，但具有更多特性。

- .C++以C为基础，包含了C语⾔部分。区块，语句，预处理器，内置数据类型，数组，指针等特性都是来⾃于C。
- C++包含了⾯向对象的特性，⽐如封装，继承，多态，virtual函数的特性。
- C++包含了泛型编程的部分。
- C++包含了STL部分。
总之，C++是在C语⾔基础上，包含了其它特性⽽发展⽽来的，相⽐C语⾔来说更加灵活和复杂。

## 前置++返回的是左值还是右值，后置++呢？字符串字面量呢?

- **前置++**：返回**左值**（直接对对象自增并返回该对象）
- **后置++**：返回**右值**（创建临时对象保存原值，对原对象自增后返回临时对象）
- **字符串字面量**：返回**左值**（存储在数据段，有固定内存地址可取址）

## 右值引用是如何提⾼性能的？

右值引⽤主要是通过避免不必要的拷贝操作来提高代码的性能的。

## 介绍⼀下RVO？

RVO（ReturnValueOptimization）是⼀种编译器优化技术，用于消除不必要的临时对象拷贝，提高代码的性能。RVO主要针对函数返回局部对象的情况，通过优化，可以避免创建临时对象并执行拷贝构造函数。
RVO的**基本思想**是：在函数调用栈上直接构造返回值，而不是先构造⼀个局部对象，然后再拷贝到调用者的栈空间。这样可以减少临时对象的创建和销毁，提高代码的运行效率。

## 如果一个class的this指针被删除后强行访问会有什么影响？

会出现崩溃或者输出乱码值

```c++
#include <iostream>
using namespace std;
class C
{
public:
    int data = 42; // 假设有一个成员变量
    void destroy()
    {
        delete this; // 销毁对象
    }
};

int main()
{
    C *obj = new C();
    obj->destroy();
    cout << "强行访问 obj->data: " << obj->data << endl;
    return 0;
}
```

输出结果：

![delete this后访问成员变量的乱码输出](image.png)

## volatile关键字的作用？

**作用**：防止编译器优化，强制每次从内存直接读写变量。不保证线程安全，原子操作推荐用atomic。

- 硬件寄存器访问：硬件寄存器的值可能被外部设备随时修改（如传感器、GPIO 状态）
- 中断服务程序（ISR）与主程序共享变量：中断可能异步修改变量（如标志位），主程序需感知最新值
- 多线程环境中的简单标志位：用于线程间通知（如退出标志），但不保证线程安全
- 防止空循环被优化

## C++函数封装器为什么优于函数指针？

函数封装器的**优点**：

- 函数封装器兼容函数指针，lambda表达式和仿函数，代码更加简洁、清晰且利于拓展。
- 函数封装器类型安全，有严格的类型检查。
- 与现代C++特性的深度集成
- 面向对象支持

## strcpy的缺点是什么？

- strcpy会造成缓冲区溢出并导致不确定的问题。
- strcpy会导致软件漏洞容易被利用。轻则导致程序崩溃，重则导致黑客找到存储器上返回地址的值，替换为恶意程序。

## class和struct的区别？

- class的默认成员和继承都是private的，如果要存储一些内部使用的成员变量推荐使用class,因为内部的一些数据不希望被外部随意获取。
- struct默认是public的，如果是要给外部提供一些所需的数据可以使用struct。

## C++中switch和if else的区别在哪里？

- switch只支持整数和枚举类型，如果是仅仅使用整数和枚举类型的逻辑判断，使用switch的性能更佳。编译器会生成一个跳转表给switch语句。考虑代码可读性推荐使用。
- if else可以判断所有的逻辑类型。if else使用遍历的方法。理论上性能会差一点，但是编译器优化后，只要分支不超过100个，switch的性能和if else性能接近。

## C++ inline内联的作用？

**作用**：将代码复制到调用处，消除函数调用开销，提高性能但会导致代码膨胀。

**C++17特性**：允许多次定义，不同文件中同名inline函数可实现不同功能（类似static）。

## 虚函数和纯虚函数的区别？

- **虚函数**: 用 `virtual` 声明，可有默认实现，派生类可选重写，类可实例化。运行时通过基类指针/引用动态绑定到实际类型的函数。
- **纯虚函数**: 声明后加 `=0`，无默认实现，强制派生类重写。包含纯虚函数的类是抽象类，不可实例化，用于定义接口规范。
代码：

```c++
#include <iostream>
using namespace std;

class Animal {
public:
    virtual void speak() {  // 虚函数，有默认实现
        cout << "Animal sound" << endl;
    }
    virtual ~Animal() {}  // 虚析构函数
};

class Dog :public Animal {
public:
    void speak() override {  // 重写虚函数
        cout << "Woof!" << endl;
    }
};

int main() {
    Animal* animal = new Dog();
    animal->speak();  // 输出: Woof!
    delete animal;
    return 0;
}
```

```c++
#include <iostream>
using namespace std;

class Shape {// 抽象基类
public:
    virtual void draw() = 0;  // 纯虚函数，无实现
    virtual ~Shape() {}
};

class Circle :public Shape {
public:
    void draw() override {  // 必须实现纯虚函数
        cout << "Drawing a circle" << endl;
    }
};

int main() {
    // Shape shape;  // 错误: 不能实例化抽象类
    Shape* shape = new Circle();
    shape->draw();  // 输出: Drawing a circle
    delete shape;
    return 0;
}
```

## 怎么解决菱形继承？

C++具备多重继承，导致会出现菱形继承的问题。
一个子类继承自多个父类，多个父类本身也可以继承自同一个基类。
菱形继承会导致二义性，存储空间浪费的问题。
解决方法：

- 虚继承：子类只继承一次父类的父类。在中间基类继承父共同基类时加上virtual关键词。其实现原理就是**依靠虚基类表**，编译器为每个虚继承的类生成一个虚基类指针，指向这个虚基类表。虚基类表存储偏移量，用于在运行时定位共享基类成员的位置对象。

```c++
class Derived1 : virtual public Base {}; // 虚继承
class Derived2 : virtual public Base {}; // 虚继承
class MostDerived : public Derived1, public Derived2 {};
```

## override和final关键字的作用？

解决不能阻止某个虚函数进一步重写的问题。
override:显式标记派生类中的函数是对基类虚函数的覆盖（重写），并强制编译器检查函数签名是否完全匹配。核心机制是签名检查(编译器验证派生类函数的签名是否与基类虚函数一致）和避免假重写。
final:禁止类被继承或虚函数被进一步重写，锁定设计意图。修饰类则类不可被继承。修饰虚函数则虚函数在派生类中不可再被重写。

```c++
class Base {
public:
    virtual void print(int x) const; // 基类虚函数
};
class Derived : public Base {
public:
    void print(int x) const override; // ✅ 正确覆盖
    void print(double x) override;    // ❌ 编译错误：参数类型不匹配
};
```

```c++
class Base {  
public:  
    virtual void foo() final;  
};  
class Derived : public Base {  
    void foo() override; // ❌ 编译错误：foo是final的[3,9]
};  
```

## C++类型推导的作用和用法？

C++作为一种强类型语言，类型匹配比较麻烦，所以借助编译器来处理类型推导比较好，提升编码效率。
**auto**：用于变量的类型的推导，初始化一个值然后去推导变量的类型。如果是使用auto定义多个变量，多个变量必须是同一类型。类型推导会丢失引用和cv语义，可以使用auto&保留，万能引用auto&&,会根据初始值属性判断是左值还是右值引用。常见于lambda表达式。
**decltype**:推导表达式的类型（保留所有信息)

```c++
int a = 10;
decltype(a) b = 20;     // b 为 int
decltype(a + 3.14) c;   // c 为 double[5]
```

## function,lambda,bind之间的关系？

​Lambda 和 std::bind 是生产者，生成可调用的对象。function是消费者，管理各类对象并提供一致的调用接口。

- function：通用可调用对象的包装器，支持类型擦除，统一存储 Lambda、std::bind 结果等。依赖其他可调用对象作为其内容。
- lambda：生成匿名函数对象（闭包），可捕获外部变量，提供简洁的语法定义临时函数。可独立使用，或作为 std::function/std::bind 的输入。优点是直接内联，系统开销小。
- bind：绑定函数的部分参数，生成新的可调用对象，支持参数重排和占位符机制。常常绑定普通函数、成员函数或 Lambda。

## 继承下的构造函数和析构函数执行顺序？

**构造顺序**（从上到下）:
1. 基类构造函数
2. 派生类成员对象构造函数（按声明顺序）
3. 派生类自身构造函数

**析构顺序**（与构造相反）:
1. 派生类自身析构函数
2. 派生类成员对象析构函数（按声明逆序）
3. 基类析构函数

**原理**: 确保派生类使用基类资源前，基类已完成初始化；销毁时先释放派生类资源，再释放基类资源。

**多继承情况**: 按继承列表顺序依次构造基类，析构时逆序。

## 虚函数表和虚指针的创建时机？

- **虚函数表**: 编译期生成。编译器检测到 `virtual` 关键字时为类生成虚表（函数指针数组）。派生类重写虚函数时更新表中地址。
- **虚表指针(vptr)**: 运行期对象构造时初始化。每个对象独立拥有 vptr，位于对象内存布局起始位置，指向所属类的虚表。

## 虚析构函数的作用？

**核心作用**: 确保通过基类指针删除派生类对象时，能正确调用派生类析构函数，避免资源泄漏

**原理**: 非虚析构时，`delete` 基类指针只调用基类析构函数 → 派生类资源无法释放。虚析构通过动态绑定，保证先调用派生类析构，再调用基类析构。

**适用场景**:
- 多态基类（通过基类指针管理派生类对象）
- 抽象接口类
- 派生类含动态资源

## C++11有哪些主要特性？

C++11主要特性：

- 1.类型推导
- 2.智能指针
- 3.右值引用和移动语义
- 4.constexpr 编译时计算

## 动态库和静态库的区别？

- 静态库​：通过编译器生成目标文件，再用归档工具打包成.a或.lib文件。编译时直接嵌入库代码，符号在链接阶段解析完成。适合嵌入式系统或离线环境。程序启动时**自动加载**。牺牲空间换取独立性和启动速度，适合封闭环境或资源隔离需求。
- ​动态库​：编译时添加-fPIC（位置无关码）和-shared选项，生成.so或.dll文件。运行时通过动态加载器解析符号地址。运行时通过API**手动加载**。牺牲部署复杂度换取灵活性和资源共享，适合模块化系统或高频更新场景。

## 右值引用和左值引用的区别？

**左值引用**的作用​：

- ​减少拷贝​：作为函数参数或返回值时避免数据复制。
- ​修改原对象​：通过引用直接操作原始数据。
- ​生命周期管理​：const T& 可延长临时对象的生命周期至引用作用域结束。

**右值引用**的作用：

- 通过“窃取”临时对象的资源（如堆内存），避免深拷贝。
- 完美转发：在模板中保持参数的原始值类别（左值/右值），通过forward转发。

```c++
template<typename T>
void wrapper(T&& arg) {
    target(std::forward<T>(arg)); // 保留左值/右值属性
}
```

## C++什么时候生成默认拷贝构造函数？

默认拷贝构造函数（执行浅拷贝）在以下四种情况下会被编译器自动生成：

- 类成员包含有拷贝构造函数的类对象
- 类继承自有拷贝构造函数的基类
- 类包含虚函数
- ​类存在虚继承

## C++类型推导为什么会有额外的开销？

C++的类型推导之所以会有额外的开销，是因为以下几个原因：

- 1.推导规则复杂：auto会忽略初始化表达式的顶层const、引用和数组退化。需编译器多步分析。decltype的值类别敏感，需根据表达式是变量、函数调用或带括号的左值，分别应用不同规则推导。
- 2.模板实例化负担：在模板中使用auto或decltype推导返回值时，可能触发多次模板实例化。
- 3.​意外的值拷贝:若初始化表达式返回引用，但auto未显式声明引用，会进行值拷贝。

## C++如何搜索链接到so动态库中的符号的？

C++链接到动态库的过程：

- ​动态库加载与初始化：操作系统通过 mmap 将库文件映射到进程地址空间，动态链接器解析库的依赖关系，递归加载所有依赖库。
- 符号查找顺序：动态链接器按固定顺序解析符号，先加载主程序符号表，再进行广度搜索逐层加载动态库，最后加载全局符号表
- 使用符号绑定机制：符号绑定机制分为立即绑定和延迟绑定，延迟绑定通过全局偏移表实现符号的解析。

## vector与普通数组的区别？vector扩容如何影响复杂度？

vector和数组的区别在于：vector是动态大小，能够自动管理堆内存，有边界检查，并提供了一些功能接口。vector扩容时会将老的元素复制到新开辟的内存空间中，频繁扩容会导致性能下滑。

## 进程同步的技术有哪些？

进程同步技术主要用于协调多个进程对共享资源的访问，避免竞态条件（Race Condition）和数据不一致问题。

- 互斥锁：通过锁定机制确保同一时刻只有一个进程能访问临界区资源。适用于简单共享资源的**独占访问**。

```c++
std::mutex mtx;
void critical_section() {
    std::lock_guard<std::mutex> lock(mtx); // 自动加锁
    // 访问共享资源
} // 自动解锁
```

- ​信号量：通过计数器控制**多个进程**对共享资源的访问权限。适用于资源池的管理。

```c++
#include <semaphore>
std::counting_semaphore<10> sem(3); // 允许3个进程同时访问
void access_resource() {
    sem.acquire(); // 获取信号量
    // 使用资源
    sem.release(); // 释放信号量
}
```

- 条件变量：允许进程等待特定条件成立后再继续执行，需与互斥锁配合使用。通过wait()释放锁并阻塞，通过notify_one()或notify_all()唤醒等待进程。

```c++
std::mutex mtx;
std::condition_variable cv;
bool data_ready = false;

void consumer() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, []{ return data_ready; }); // 等待条件满足
    // 消费数据
}
```

- 屏障：强制多个进程在指定同步点等待，直到所有进程到达后才继续执行。主要适用于并行计算中分阶段任务等场景。

```c++
#include <barrier>
std::barrier sync_point(5); // 等待5个进程

void worker() {
    // 阶段1任务
    sync_point.arrive_and_wait(); // 同步点
    // 阶段2任务
}
```

- 原子操作：通过硬件指令保证对单个变量的操作不可分割，避免数据竞争。主要适用于高频计数器、无锁数据结构等场景。

```c++
#include <atomic>
std::atomic<int> counter(0);

void increment() {
    counter.fetch_add(1, std::memory_order_relaxed);
}
```

- 读写锁：允许多个进程同时读取共享资源，但**写入时需独占访问**。适用于读多写少的共享数据（如配置信息）。

```c++
#include <shared_mutex>
std::shared_mutex rw_mutex;

void read_data() {
    std::shared_lock lock(rw_mutex); // 共享锁（可并发读）
    // 读取数据
}

void write_data() {
    std::unique_lock lock(rw_mutex); // 独占锁（互斥写）
    // 写入数据
}
```

## malloc和new的具体实现？

- malloc是C标准库函数，其核心是通过操作系统提供的系统调用管理堆内存。使用分配的内存块头部存储元数据，通过链表链接所有空闲块。
- new是C++运算符，其行为包含内存分配和对象构造两阶段，内存分配阶段调用全局operator new函数，默认实现内部调用malloc。对象构造阶段用placement new在已分配内存上调用构造函数。new直接返回响应的数据类型的指针。

```c++
void* memory = operator new(sizeof(MyClass));  // 调用malloc分配内存
MyClass* obj = new (memory) MyClass();    // 在memory地址调用构造函数
```

## 不相关的进程间能否使用管道实现通信？

不相关的进程之间可以通过**命名管道**实现通信。
命名管道：
命名管道通过 mkfifo() 或 mknod() 系统调用创建，在文件系统中生成一个特殊的 ​FIFO 文件​（如 ./myfifo）。该文件不存储实际数据，仅作为内核中管道缓冲区的访问入口。
任何进程只要知道该文件路径，即可通过 open() 打开管道进行读写，无需亲缘关系。进程以写模式（O_WRONLY）打开管道，调用 write() 向管道写入数据。另一进程以读模式（O_RDONLY）打开管道，调用 read() 从管道读取数据。

## 协程是什么？

C++协程（Coroutine）是C++20引入的一种轻量级并发编程机制，它允许函数在执行过程中暂停（挂起）并在稍后恢复，而无需依赖操作系统线程调度，从而简化异步编程、提高资源利用率。
协程是一种特殊函数，可在执行中主动挂起，保存当前状态（局部变量、执行位置等），后续通过协程句柄恢复执行。
C++20采用**无栈协程模型**，挂起时将上下文（局部变量、寄存器状态）存储在堆上

## C++的重载和C语言的区别在哪里？具体是如何实现？

- C++重载的实现原理是**名字修饰**。汇编阶段使用修饰名生成符号，不同参数列表对应独立符号。C++重载在编译阶段生成唯一符号，链接阶段解析符号，重载解析符号，调用函数时，编译器选择参数最匹配的重载版本。
- C语言：汇编阶段仅用函数名生成符号，同名函数导致符号重复定义。链接阶段C链接器按函数名查找地址，无法区分重载函数。

## C和C++编译出来的文件有什么区别？

和C++编译生成的可执行文件（如ELF、PE格式）在格式层面是兼容的（均可由操作系统加载执行），但因其语言特性差异，**二进制内容**存在显著区别。

- 编译器通过名字修饰将函数名、参数类型/数量/顺序编码为唯一符号。C语言仅用函数名标识符号。
- 异常处理与运行时类型信息(RTTI)：编译器在二进制中插入异常处理框架​（如try/catch的栈回退逻辑）和RTTI数据结构​（用于dynamic_cast和typeid），以支持面向对象特性。C语言没有。
- 函数调用约定与对象模型：C++成员函数调用隐含传递this指针（通常通过寄存器或栈），而C函数无此机制。C++在main()前/后插入全局/静态对象的构造/析构代码，而C程序仅按代码顺序执行。

## 多继承把子类指针转为父类指针和单继承的区别在哪里？

多继承和单继承下子类指针向父类指针的转换存在本质区别，核心差异在于**内存布局的复杂性和指针偏移机制**。

- 单继承：子类对象的内存布局为父类部分在前，子类新增部分在后。子类指针Derived*转换为父类指针Base*时，​地址不变。因父类部分位于对象起始位置，无需调整指针。子类与父类共享一个虚表指针，位于对象起始处。

```c++
class Base { int x; };
class Derived : public Base { int y; };
```

- 多继承：子类对象**按继承顺序排列**多个父类，​每个父类占据独立内存区域。转换到第一个父类​（如Base1*）时地址不变（与单继承相同）。转换到非第一个父类​（如Base2*）时，​编译器自动添加偏移量，指向Base2在子类中的起始位置。​每个含虚函数的父类在子类中**独立维护虚表指针**。

```c++
class Base1 { int a; };
class Base2 { int b; };
class Derived : public Base1, public Base2 { int c; };
```

## 有的类把析构函数声明为虚函数，什么场景下会用到？

将析构函数声明为虚函数的核心目的是**解决基类指针指向派生类对象时的资源正确释放问题**，避免内存泄漏和未定义行为。

- 多态基类（通过基类指针删除派生类对象）：当基类指针指向派生类对象，且基类析构函数非虚时，delete该指针仅调用基类析构函数，派生类的析构函数不被执行，导致派生类资源（如动态内存、文件句柄等）泄漏。
  
```c++
class Base {
public:
    virtual ~Base() {}  // 虚析构函数
};
class Derived : public Base {
public:
    ~Derived() override { /* 释放派生类资源 */ }
};

Base* obj = new Derived();
delete obj;  // 正确调用顺序：Derived::~Derived() → Base::~Base()
```

- 抽象类（含纯虚函数的接口类）:强制派生类实现析构逻辑，确保多态销毁安全。抽象类本身不可实例化，但需为纯虚析构函数提供定义（空实现即可）。

```c++
class AbstractBase {
public:
    virtual ~AbstractBase() = 0;  // 纯虚析构
};
AbstractBase::~AbstractBase() {}   // 必须定义

class Impl : public AbstractBase {
public:
    ~Impl() override { /* 资源释放 */ }
};
```

- 工厂模式返回基类指针:工厂函数返回基类指针（实际指向派生类对象），需通过基类指针统一管理对象生命周期

```c++
class Factory {
public:
    static Base* createObject() { return new Derived(); }
};

Base* obj = Factory::createObject();
delete obj;  // 依赖虚析构正确释放Derived资源
```

- 多层级继承结构：若中间层基类（非最顶层）可能被多态使用，其析构函数也需为虚函数，以确保析构链完整执行。
  
```c++
class Base { virtual ~Base(); };
class Middle : public Base { virtual ~Middle(); }; // 必须为虚
class Derived : public Middle { ~Derived(); };

Base* obj = new Derived();
delete obj;  // 调用顺序：Derived → Middle → Base
```

## unique指针在编译期如何保证是真的unique？

unique_ptr实现真的unique是靠的以下几个机制：

- **禁用拷贝语义**(核心)：unique_ptr 内部将拷贝构造函数和拷贝赋值运算符声明为 = delete，直接禁止复制行为。
- 仅支持移动语义：unique_ptr 允许通过移动操作转移所有权，转移后原指针变为 nullptr。临时右值可隐式移动。

```c++
std::unique_ptr<int> p3 = std::unique_ptr<int>(new int(10)); // 合法
```

- 编译器的静态检查:类型系统强制约束,如当尝试拷贝时，编译器检查到调用了被删除的函数，直接报错。

## 移动语义如何使用？

移动语义是C++11引入的核心特性，通过转移资源所有权而非复制资源，显著提升程序性能。绑定临时对象（右值），标记可被“窃取”资源的对象。使用move将左值强制转换为右值引用，触发移动语义。需定义移动构造函数和移动赋值运算符，并标记noexcept以保证异常安全。

```c++
std::string s1 = "Hello";
std::string s2 = std::move(s1);  // s1的资源被转移给s2，s1变为空
```

## emplace_back和push_back的区别？

两者均为容器尾部添加元素的方法，但**底层机制**和**适用场景**不同。

- push_back:先构造临时对象，再拷贝/移动到容器中.
- emplace_back:​直接在容器内存中构造对象，避免临时对象创建和拷贝/移动。减少了构造临时对象这一步，性能更优。不支持初始化列表，存在隐式类型转换风险。
结论：emplace_back的效率相对更高，因此在代码中**尽量用emplace_back**代替push_back。

## deque和vector的区别？内存布局有啥区别？

- deque:​分段连续存储，由多个固定大小的内存块（chunks）组成，通过中控器（指针数组）管理逻辑连续性。动态分配新内存块，只需更新中控器的指针，​无需移动现有元素。扩容成本更低。无法保证整体内存连续。需高频头尾操作时使用queue。
- vector:​单块连续内存，元素**物理地址连续**,容量不足时，重新分配一块更大的连续内存,严格连续，支持直接传递首地址。操作集中在尾部。

## weak_ptr如何解决循环引用？

weak_ptr 是 C++11 引入的智能指针，专为配合 shared_ptr 解决循环引用问题而设计，同时提供安全的对象访问机制。循环引用指两个或多个对象通过 shared_ptr 相互持有，导致引用计数无法归零，对象无法释放。
weak_ptr 通过​**非拥有式观察**打破循环。

```c++
class A {
    std::shared_ptr<B> b_ptr;
};
class B {
    /*出现循环引用*/
    //std::shared_ptr<A> a_ptr;
    /*修改为 weak_ptr解决循环引用*/
    std::weak_ptr<A> a_ptr;  
};
```

## weak_ptr如何升级为shared_ptr？

weak_ptr 通过 lock() 方法安全升级为 shared_ptr，确保访问对象时其未被销毁。

```c++
std::weak_ptr<A> weak_a = ...;  // 从某处获取 weak_ptr
if (auto shared_a = weak_a.lock()) {  // 尝试升级
    shared_a->do_something();       // 对象存活，安全访问
} else {
    // 对象已销毁，避免悬垂指针
}
```

weak_ptr 通过控制块中的​弱引用计数​感知对象状态：
​构造时​：复制 shared_ptr 的控制块指针，弱引用计数 +1。
​析构时​：弱引用计数 -1，若弱引用计数和强引用计数均为 0，释放控制块。
​**lock() 时**​：检查控制块中的强引用计数，决定是否构造新 shared_ptr。

## C++的左值和右值是如何使用的？

- 左值：具有持久存储位置的对象，可被取地址（&操作符），通常有变量名，可多次使用。可出现在赋值左侧，生命周期在作用域内有效。
- 右值：临时对象或字面量，无持久存储位置，不可取地址，通常为一次性使用的值。仅能出现在赋值右侧，生命周期在表达式结束时结束。

## C++14的新特性？

C++14主要是对一些C++11的已有特性做了扩展。

- 支持更灵活的类型推导，C++14支持decltype(auto),这个auto仅仅作为占位符使用。
- constexpr支持更加广泛的语法和应用，如可以使用局部变量。
- 支持更加通用的lambda表达式，允许表达式内部使用auto参数，处理泛型类型更方便。
- 支持返回类型推导
  
```c++
auto add(int x,int y)//推导出来是int类型的返回值
{
    return x+y;
}
int main()
{
    int num1=1;
    int num2=10;
    int num3=add(num1,num2);
    cout<<num3;
    return 0;
}
```

## C++17的新特性？

- 结构化绑定：允许通过一个简单的声明将元组或其他数据结构的成员绑定到变量。
- if初始化：在 if 和 switch 语句中可以直接初始化变量。

```c++
int num4 = 100;
    if (int a = 1 < num4)//C++17支持if初始化
    {
        cout << "C++17 yes" << endl;
    }
    return 0;
```

- 折叠表达式：支持更灵活的模板元编程
- constexpr lambda 表达式：允许 lambda 表达式在编译时进行求值。

```c++
int main() {
    // 定义一个 constexpr lambda 表达式，用于计算两个整数的和
    constexpr auto add = [](int x, int y) constexpr {
        return x + y;
    };

    // 在编译时调用 lambda 表达式
    constexpr int result = add(3, 4);

    // 输出结果
    std::cout << "Result: " << result << std::endl;

    return 0;
}
```

- std::optional：提供了一种可选值的容器，用于解决空指针的问题。

```c++
// 一个可能返回无效值的函数，使用 optional 来安全地表示结果
optional<int> divide(int numerator, int denominator)
{
    if (denominator == 0)
    {
        return nullopt; // 返回无值状态
    }
    return numerator / denominator;
}

int main()
{
    auto result1 = divide(10, 2);
    if (result1.has_value())
    {
        cout << "Result of 10 / 2: " << result1.value() << endl;
    }
    else
    {
        cout << "Division by zero error." << endl;
    }

    auto result2 = divide(5, 0); // 除以零，应该失败
    if (result2)
    {
        // 另一种检查方式
        cout << "Result of 5 / 0: " << *result2 << endl;
    }
    else
    {
        cout << "Division by zero error." << endl;
    }

    return 0;
}
```

- std::variant：提供了一种可变的类型安全的联合。
- filesystem：提供了一个现代化的、面向对象的文件系统操作 API。

## 如何让对象只能产生在堆上？

将对象的**析构函数设置为私有**的，因为在栈上分配对象的时候，编译器会自动调用对象的构造函数和析构函数，因此此时如果在栈上分配内存会编译报错，就将内存限制在了只能分配在堆上。

## 如何让对象只能产生在栈上？

把构造函数禁用，使其无法new对象在堆上。

## C++指针悬挂问题是什么，如何解决？

**定义**：指针指向的内存已被释放或失效，但指针仍保留原地址，访问会引发未定义行为。

**常见场景及解决方法**：

- **指针释放后未置空**：释放内存后立即置空指针（`p = nullptr`）
- **返回局部变量地址**：避免返回局部对象指针/引用，改用动态分配或静态变量
- **容器内存重分配**：容器扩容导致原有元素指针失效，避免长期持有容器元素指针
- **多指针共享同一内存**：一个指针释放内存后，其他指针也需置空或使用智能指针

## C++中的特化和偏特化是什么？

C++中的模板特化和偏特化是模板编程中用于针对特定类型或条件提供定制化实现的高级技术，旨在**优化性能、处理特殊逻辑或增强类型安全性**。

- 模板特化(全特化)：为模板的所有参数指定具体类型，完全覆盖通用模板的实现。

- 模板偏特化：仅对模板的部分参数进行特化，其余参数保持泛型。

```c++
/*模板全特化和偏特化的例子*/
#include <iostream>
using namespace std;

// 主模板
template <typename T>
class Vector {
private:
    T* data;
    size_t size;

public:
    Vector(size_t s) : size(s) {
        data = new T[size];
    }

    ~Vector() {
        delete[] data;
    }

    void info() {
        cout << "通用Vector" << endl;
    }
};

// 全特化：bool 类型
template <>
class Vector<bool> {
private:
    unsigned char* compressedData; // 使用位压缩存储 bool 数组

public:
    Vector(size_t size) {
        compressedData = new unsigned char[(size + 7) / 8];
    }

    ~Vector() {
        delete[] compressedData;
    }

    void info() {
        cout << "特化BoolVector" << endl;
    }
};

// 偏特化：所有指针类型的 Vector<T*>
template <typename T>
class Vector<T*> {
private:
    T** data;
    size_t size;

public:
    Vector(size_t s) : size(s) {
        data = new T*[size];
    }

    ~Vector() {
        delete[] data;
    }

    void info() {
        cout << "偏特化PointerVector" << endl;
    }
};

int main() {
    Vector<int> v1(10);       // 调用主模板
    v1.info();                // 输出: 通用Vector

    Vector<bool> v2(10);      // 调用全特化
    v2.info();                // 输出: 特化BoolVector

    Vector<int*> v3(10);      // 调用偏特化
    v3.info();                // 输出: 偏特化PointerVector

    return 0;
}
```

## vector的reserve和resize的区别是什么?

vector 的 resize 和 reserve 是两个用于管理容器大小和内存分配的成员函数。它们的区别如下：

- reserve:预留至少 n 个元素的内存空间，但**不创建任何元素**。如果不存在扩容的情况，内存不变化。目的就是为了提前开辟内存空间**提高性能**，避免内存碎片。
- resize：直接改变元素数量，**可能增删元素**。不存在重分配内存的情况。更改容器的大小并调整元素数量，会分配/释放内存和复制/删除元素。

## 全局变量存储在哪里？

全局变量存储在内存的**静态存储区**。和static静态变量一样。

- 已经初始化的全局变量存储于.data段
- 未初始化的全局变量存储于.bss段

## 内联函数和宏定义的区别？

内联函数和宏在C/C++中均用于优化代码性能，但两者在机制、安全性和适用场景上有本质区别。

- 内联函数：**编译**阶段展开，由编译器进行语法检查和优化，有严格的类型检查。参数仅仅求一次值。如果函数过于复杂会被编译器优化视为普通函数进行调用。
- 宏：**预处理**阶段就进行文本替换，无类型检查，

## vector,list,unordered_map,map迭代器失效问题如何解决？

- vector：push_back、insert 触发扩容 → ​所有迭代器失效。erase 删除元素 → ​被删位置及之后迭代器失效。
解决方法：

```c++
/*更新迭代器*/
for (auto it = vec.begin(); it != vec.end(); ) {
    if (*it % 2 == 0) {
        it = vec.erase(it); // 返回下一元素迭代器
    } else {
        ++it;
    }
}
```

- list:erase 仅使被删节点的迭代器失效，其他节点不受影响
解决方法：

```c++
/*删除时跳过失效节点*/
for (auto it = lst.begin(); it != lst.end(); ) {
    if (*it == 3) {
        it = lst.erase(it); // 直接更新迭代器
    } else {
        ++it;
    }
}
/*使用后置自增(推荐，主要是方便)*/
for (auto it = lst.begin(); it != lst.end(); ) {
    if (*it == 3) {
        lst.erase(it++); // 先传it位置，再自增
    } else {
        ++it;
    }
}
```

- unordered_map:插入可能触发 ​Rehash → 所有迭代器失效,删除仅使被删节点失效。
解决方法：

```c++
/*插入后重新获取迭代器*/
unordered_map<int, string> umap;
auto it = umap.find(key);
if (it == umap.end()) {
    umap.insert({key, val}); // 插入可能触发Rehash
    it = umap.find(key);     // 重新获取迭代器
}
/*删除时更新迭代器*/
for (auto it = umap.begin(); it != umap.end(); ) {
    if (it->second == "delete") {
        it = umap.erase(it);
    } else {
        ++it;
    }
}
```

- map：只有被删除元素的迭代器失效，其他迭代器安全。
解决方法：

```c++
/*更新迭代器*/
for (auto it = m.begin(); it != m.end(); ) {
    if (it->first == key) {
        it = m.erase(it); // 返回下一节点迭代器
    } else {
        ++it;
    }
}
```

## C++空的class，会主动提供哪些函数？

C++创建了一个空的class，会在特定的情况下提供一些函数。

- ​缺省构造函数​：声明无参对象时会提供。
- 缺省拷贝构造函数：对象拷贝初始化时候会触发
- 缺省析构函数：对象声明周期结束时会触发
- 缺省赋值运算符：对象复制操作时会触发

```c++
/*编译器主动构造函数的场景*/
class MyClass {
public:
    // 缺省构造函数
    MyClass() {}

    // 拷贝构造函数（缺省形式）
    MyClass(const MyClass& other) {}

    // 缺省析构函数
    ~MyClass() = default;

    // 缺省赋值运算符（由编译器生成）
    MyClass& operator=(const MyClass& other) = default;
};

int main() {
    MyClass obj1;           // 使用缺省构造函数创建对象
    MyClass obj2 = obj1;    // 使用拷贝构造函数创建对象
    MyClass obj3;
    obj3 = obj1;            // 使用赋值运算符
    return 0;
}
```

C+++11新增的移动语义，使得编译器会自动构造移动构造函数和移动赋值运算符。

- 移动构造函数：对象通过右值初始化触发
- 移动赋值运算符：对象通过右值赋值触发

```c++
#include <iostream>

class MyClass {
public:
    int a;
    int b;

    // 缺省构造函数
    MyClass() : a(0), b(0) {}

    // 拷贝构造函数
    MyClass(const MyClass& other) = default;

    // 移动构造函数
    MyClass(MyClass&& other) noexcept = default;

    // 拷贝赋值运算符
    MyClass& operator=(const MyClass& other) = default;

    // 移动赋值运算符
    MyClass& operator=(MyClass&& other) noexcept = default;

    // 析构函数
    ~MyClass() = default;
};

int main() {
    MyClass obj1;
    MyClass obj2 = std::move(obj1); // 调用移动构造函数
    MyClass obj3;
    obj3 = std::move(obj2);         // 调用移动赋值运算符
    return 0;
}
```

## C++字符串以\0结尾有什么问题？

多占用一个字节的空间，而且如果该符号在中间会造成字符串打印的截断。外部进行字符串拼接的时候会造成问题，所以最好是将字符串和长度一起传入比较安全。

## C++变量名是否占用内存空间？

局部变量名不占用地址，编译后变为地址，全局变量名占用地址，C++编译时会存储符号表，用于调试，不占用运行资源。
动态库运行加载时会加载动态符号表，占用运行内存。

## C++异常的开销是什么？

C++异常处理的核心开销来源：

- **栈展开**：当异常抛出时，运行时系统需逆向遍历调用栈，释放局部对象并查找匹配的catch块。
- **异常对象管理**：异常对象通常通过new在堆上创建，涉及内存分配和拷贝/移动构造，增加内存和析构负担。
- 处理程序查找与类型匹配：运行时需解析异常处理表，线性匹配catch块类型，调用栈越深耗时越长。
- 隐性的全局开销：数据结构维护​：即使未使用try/catch，编译器仍需生成异常处理元数据（如对象构造状态跟踪），导致代码体积增加5%-10%​，轻微影响运行效率。

## C++如何解决头文件重复包含的问题？

很多标准库是使用2种方法的。有2种解决方法：

- program once:主流编译器都支持，且性能更优。
- ifndef:所有编译器都支持，但是使用这个宏之后还是会进到对应的代码段，性能稍逊一筹。

```c++
/*方法1*/
# pragma once
/*方法2*/
# ifndef
```

## C++深拷贝和浅拷贝的区别？

​深拷贝​​和 ​浅拷贝是对象复制的两种核心机制，主要区别在于对**动态资源**（如堆内存）的处理方式。

- 浅拷贝：仅复制对象的成员变量值，多个对象共享同一块动态内存，易导致悬垂指针、双重释放​的问题，性能开销小。

```c++
class Shallow {
public:
    int* data;
    Shallow(int val) { data = new int(val); }
    // 默认拷贝构造函数（浅拷贝）
    Shallow(const Shallow& other) : data(other.data) {}
    ~Shallow() { delete data; }
};

int main() {
    Shallow obj1(10);
    Shallow obj2 = obj1; // 浅拷贝
    // obj1 和 obj2 的 data 指向同一内存
    // 此处代码就是删除了obj1导致后续触发了double free的错误
    delete obj1.data;
    cout<<"obj2.data: "<<*obj2.data<<endl;
    return 0;
}
```

- 深拷贝：复制成员变量值，并为指针指向的资源分配新内存，复制内容。但是需手动实现拷贝构造函数和赋值运算符重载。优点是**资源独立，无共享风险**。

```c++
#include <iostream>
using namespace std;

class Deep {
public:
    int* data;

    // 默认构造函数
    Deep() : data(new int(0)) {}  // 可指定默认值，如 0

    Deep(int val) : data(new int(val)) {}

    // 拷贝构造函数（深拷贝）
    Deep(const Deep& other) {
        data = new int(*other.data);
    }

    // 赋值运算符重载（深拷贝）
    Deep& operator=(const Deep& other) {
        if (this != &other) {
            delete data;
            data = new int(*other.data);
        }
        return *this;
    }

    ~Deep() {
        delete data;
    }
};

int main() {
    Deep d1(5);
    Deep d2(d1); // 拷贝构造
    Deep d3;     // 现在可以调用默认构造函数
    d3 = d1;     // 赋值操作
    //验证是否正确实现深拷贝
    cout << "d1: " << *d1.data << endl;
    delete d1.data;
    cout << "d3: " << *d3.data << endl;
    return 0;
}
```

## SFINAE是什么？

SFINAE是 C++ 模板编程中的核心机制，用于在编译期根据类型特性选择或禁用特定的模板重载。其核心思想是：​当模板参数替换导致无效代码时，编译器不会报错，而是静默忽略该模板候选，继续**尝试其他匹配的重载版本**。

适应场景：

- ​条件启用模板函数

```c++
template <typename T>
typename std::enable_if<std::is_integral<T>::value, void>::type
process(T val) { /* 处理整型 */ }

template <typename T>
typename std::enable_if<std::is_floating_point<T>::value, void>::type
process(T val) { /* 处理浮点型 */ }
```

- 检测类型成员或方法:结合 decltype 和 std::void_t 检查类型是否支持特定操作

```c++
template <typename, typename = void>
struct has_serialize : std::false_type {};

template <typename T>
struct has_serialize<T, std::void_t<decltype(std::declval<T>().serialize())>> 
    : std::true_type {};
```

- 重载决策优化:在多个模板重载中，SFINAE 帮助编译器选择最匹配的版本，避免歧义。

```C++
void process(double val);  // 普通函数
template <typename T>     // 模板函数
void process(T val);
```

## C++中Static全局变量，全局变量，和extern变量的区别

全局变量、static全局变量和extern变量这三者的区别在于**作用域**、链接属性和**存储方式**的不同：

- 全局变量：作用于整个程序，存储于静态存储区
- static全局变量：仅在该源文件内可见，对其他文件不可见，仅可被内部链接，存储于静态存储区。不同文件内可以同名。
- extern变量：仅在声明所在文件内有效，不分配存储空间，生命周期是跟随实际定义的变量。不会进行初始化。

## 完美转发？

完美转发是 C++11 引入的核心特性，用于在函数模板中无损传递参数的原始值类别（左值/右值）和常量性，避免不必要的拷贝并正确触发移动语义。

- 通用引用：统一接收任意类型的参数
- 引用折叠规则：编译器根据传入参数的类型自动应用折叠规则
- forward语义还原：恢复参数的原始值类别。

```c++
#include <iostream>
#include <utility>  // std::forward, std::move

// 处理左值的函数重载
void process(int& x) {
    std::cout << "Lvalue: " << x << "\n";
}

// 处理右值的函数重载
void process(int&& x) {
    std::cout << "Rvalue: " << x << "\n";
}

// 通用转发函数模板（使用万能引用 T&&）
template <typename T>
void forwarder(T&& arg) {
    // 使用 std::forward<T> 保留参数的值类别（左值或右值）
    process(std::forward<T>(arg));
}

int main() {
    int a = 10;

    // 调用 forwarder 并传递一个左值
    forwarder(a);          // 输出: Lvalue: 10

    // 调用 forwarder 并传递一个右值（字面量）
    forwarder(20);         // 输出: Rvalue: 20

    // 调用 forwarder 并传递一个被 std::move 转换为右值的变量
    forwarder(std::move(a)); // 输出: Rvalue: 10

    return 0;
}
```

## 右值引用是如何提⾼性能的?

右值引⽤主要是通过**避免不必要的拷贝**操作来提高代码的性能的。右值引用实际上就是变相延长了对象的声明周期，不需要再创建一个临时对象，也就省去了创建临时变量时产生的开销，因此省掉了一次构造。

## 进程的虚拟内存布局？

![C++内存图](image-2.png)
从**本质上**来看，c++内存模型分成包含.text, .rodata, .data，.bss，堆，栈，内存映射区，内核空间。

- text（代码段）段存储的是程序源代码编译后的机器指令，是只读的。
- rodata（只读数据段）存放的是程序中的只读数据，⼀般是程序⾥⾯的只读变量和字符串常量。
- data（数据段）存放的是已经初始化了的全局静态变量和局部静态变量。
- bss 存放的是未初始化的全局静态变量和局部静态变量。
- 堆：⼀般由程序员分配释放， 若程序员不释放，存放⼀些new创建出来的对象。
- 栈：由编译器⾃动分配释放 ，存放函数的参数值，局部变量的值等。
- 内核空间：是操作系统内存管理的⼀部分，用于存储和运行操作系统内核的代码和数据

## C++堆和栈的区别？

- 堆：堆上的内存是动态分配的，程序在运行时可以根据需要分配和释放内存。堆的大小通常比栈⼤得多，因此可以用于存储较⼤的数据结构和对象。
- 栈：栈上的内存⽣命周期与函数调用相关。局部变量在函数被调用时自动分配内存，函数返回时自动释放内存。栈的大小相对较⼩，适用于存储较⼩的数据结构和对象。

## 栈何时会溢出？

- 递归调用层次过深：如果一个程序中存在递归调用，而递归调用的层次过深，会导致栈空间不足，发生栈溢出。
- 局部变量占用过多空间：当一个函数中声明了过多的局部变量，或者某个局部变量占用的内存空间过大时，会导致栈空间快速耗尽，发生栈溢出。
- 大量函数调用嵌套：如果程序中存在大量的函数调用嵌套，而栈空间不足以容纳所有的函数调用信息，也会导致栈溢出。

## deque,list,set,multiset,map的对比?

- deque:双向队列实现，存储空间连续，支持随机访问，性能比vector低。适合头尾部频繁操作且需要随机访问的场景。
- list:由双向链表实现，存储空间不连续，只能通过迭代器访问，插入删除效率高，但是每个位置都需要分配额外空间存储前驱元素和后继元素。适用于在**任意位置频繁插入/删除**的场景。
- set:红黑树实现，存储空间不连续，只能通过迭代器访问，适用于有序集合且**元素不重复**的场景。
- multiset:红黑树实现，存储空间不连续，只能通过迭代器访问，默认使用less仿函数进行排序，也可以自定义，适用于有序集合且**元素重复**的场景。
- map:由**红黑树**实现,红黑树是一种平衡二叉搜索树,存储空间**不连续**; 存储的元素是键值对元素。只能通过迭代器进行访问。默认使用less仿函数进行排序，map映射容器不允许重复的键。

## C++的内存对齐？

C++中的内存对齐​（Memory Alignment）是一种由编译器自动实施的内存优化机制，其核心目的是**提升CPU访问内存的效率**，并确保数据在特定硬件架构上能够安全访问。**经过内存对⻬之后，CPU 的内存访问速度⼤⼤提升。**
目的：

- **性能优化**：CPU访问对齐的内存（如4字节数据起始地址为4的倍数）通常只需一次内存操作；若未对齐，则可能触发多次访问或硬件异常，尤其在RISC架构。
- 硬件兼容性：**有利于平台移植**。某些处理器（如SPARC、早期ARM）要求数据严格对齐，否则抛出硬件异常。在ARM64上，normal memory非对齐访问不会有问题。但是device memory非对齐访问会报bus error

内存对齐一般是按从大到小的顺序去对齐的。这样可以减少内存对齐所填充的字节数，优化空间。C++的类的虚指针也会参与内存对齐。

**内存对齐的规则**：(简单来说就是结构体里最大的和编译器的默认对齐系数选较小值，然后如果要节约空间可以把最大的类型放最前面，节约空间)

- 结构体第一个成员的偏移量（offset）为 0，以后每个成员相对于结构体首地址的offset 都是该成员大小与有效对齐值中较小那个的整数倍，如有需要编译器会在成员之间加上填充字节。
- 结构体的总大小为有效对齐值的整数倍，如有需要编译器会在最末一个成员之后加上填充字节。

## c++编译链接的过程?

C++的程序从源代码到可执行程序经历了以下步骤：

- 预处理：预处理用于将所有的#include头文件以及宏定义替换成其真正的内容，预处理之后得到的仍然是文本文件。
- 编译：预处理之后的程序转换成特定汇编代码的过程。
- 汇编：将汇编代码转换成机器代码，产生一个目标文件。
- 链接：将多个目标文以及所需的库文件(.so等)链接成最终的可执行文件。

## 栈的速度为什么比堆要快？

栈的读取速度比堆上的要快，主要是由于以下几个原因：

- **数据结构**的特点：栈是一种线性数据结构，其操作是基于栈顶的，因此可以通过简单的指针操作来读取栈上的数据。相比之下，堆是一种树形数据结构，要读取堆上的数据可能需要进行指针的跳转和内存的查找操作，因此相对更为复杂和耗时。
- **内存布局**的连续性：栈上的内存分配是连续的，数据项之间存储的地址是相邻的，这使得栈上的数据读取更为高效，因为可以通过栈指针进行连续的内存读取操作。而堆上的内存分配是动态的，可能是分散的，需要通过指针跳转来访问不同的内存块，导致读取速度较慢。
- **硬件优化**：由于栈的读取操作频繁且简单，因此处理器和编译器通常会对栈上的操作进行优化，例如采用特定的指令集或硬件机制来提高栈操作的执行效率。相比之下，堆上的内存操作较为复杂，难以进行同样程度的优化

## A.cpp 调用 B.cpp 中的方法，发生了什么？

首先经过编译器处理，A.cpp和B.cpp都变成了目标文件A.o和B.o。在进行链接的过程中，A.O和B.O的符号表会合并成一个全局符号表，链接器会利用这个全局符号表，来对A调用的B的方法进行符号决议，判断是否存在唯一的符号，如果存在，就会对该符号进行重定义，对地址修正，最后生成exe文件，如果符号决议失败，就会在链接时报错。

## B.dll 调用 B.dll 发生了什么？

在链接的过程中，A.dll和B.dll动态库引用的相关信息，比如动态库的名字，符号表和重定位等信息会复制到可执行文件中。在文件开始加载和链接时，会使用动态链接的方法，结合这些信息，来对使用的动态库的符号进行重定位，确定调用的符号的地址，完成调用。

## explicit关键字是什么作用？

explicit 关键字在 C++ 中用于禁止**编译器进行隐式类型转换**，强制要求开发者显式调用构造函数或转换运算符，提高代码的安全性和可读性。

- 禁止隐式类型转换
- 禁止拷贝初始化
- 防止隐式转换链

```c++
#include <iostream>
#include <string>

// 定义一个支持显式构造的类
class MyClass
{
public:
    // 使用 explicit 防止隐式转换
    explicit MyClass(int value) : data(value)
    {
        std::cout << "Explicit constructor called with value: " << data << std::endl;
    }

    void printData() const
    {
        std::cout << "Data: " << data << std::endl;
    }

private:
    int data;
};

// 演示没有 explicit 的情况
class MyOtherClass
{
public:
    MyOtherClass(int value) : data(value)
    {
        std::cout << "Non-explicit constructor called with value: " << data << std::endl;
    }

    void printData() const
    {
        std::cout << "MyOtherClass Data: " << data << std::endl;
    }

private:
    int data;
};

int main()
{
    // 显式调用 MyClass 构造函数（必须的，因为构造函数被声明为 explicit）
    MyClass obj1(10);
    obj1.printData();

    // 下面这行会编译失败，因为 explicit 禁止了隐式转换
    // MyClass obj2 = 20;

    // 正确写法：显式调用构造函数
    MyClass obj2 = MyClass(20);
    obj2.printData();

    // 对于没有 explicit 的类，允许隐式转换
    MyOtherClass obj3 = 30; // 隐式转换合法
    obj3.printData();

    return 0;
}
```

## 使用lambda表达式，捕获局部变量时有什么规则？

Lambada表达式的捕获规则主要有：值捕获，引用捕获，隐式捕获和显示捕获。

- 值捕获：使用值捕获时，lambda 表达式会复制外部作用域的局部变量，并在lambda 表达式内部使用它们的副本。这意味着捕获的变量在 lambda 表达式创建时就被复制，lambda 表达式内部的操作不会影响原始变量的值。
- 引用捕获：使用引用捕获时，lambda 表达式会获取外部作用域的局部变量的引用。这意味着lambda表达式内部对变量的操作会影响到原始变量。
- 隐式捕获：通过在捕获列表中使用 = 或 & 符号，可以实现隐式捕获。使用= 捕获外部作用域的所有变量的副本，而使用 & 捕获所有变量的引用。
- 显式捕获：在捕获列表中，可以指定要捕获的特定变量，并且可以同时使用值捕获和引用捕获。

## delete和free的区别?

delete和free虽然都用于释放动态内存，但它们在设计目标、行为机制和适用场景上存在本质区别。

- delete:C++​关键字，专用于释放new分配的内存。支持释放单个对象（delete ptr）或数组（delete[] ptr）。
- free:用于释放malloc/calloc/realloc分配的内存，仅接受指针参数（free(ptr)）

```c++
/*delete和free的区别*/
#include <iostream>
#include <cstdlib>
using namespace std;

class MyClass
{
public:
    MyClass() { cout << "Constructor called!" << endl; }
    ~MyClass() { cout << "Destructor called!" << endl; }
    static void print() { cout << "Hello from MyClass" << endl; }
};

int main()
{
    // 使用 new 分配内存并调用构造函数
    auto* obj1 = new MyClass();
    MyClass::print();
    // 使用 delete：调用析构函数并释放内存
    delete obj1;
    cout << "-----------------------------" << endl;
    // 使用 malloc 仅分配原始内存，不会调用构造函数
    auto* obj2 = (MyClass*)malloc(sizeof(MyClass));
    new(obj2) MyClass(); // 手动调用构造函数（定位 new）
    MyClass::print();

    // 手动调用析构函数
    obj2->~MyClass();
    // 使用 free 释放内存
    free(obj2);
    return 0;
}
```

## c++重载和重写的区别?

重载是编译时多态，​参数列表必须不同，作用域是同一作用域。

```c++
class Calculator {
public:
    int add(int a, int b) { return a + b; }          // 重载：参数类型不同
    double add(double a, double b) { return a + b; } // 重载：参数类型不同
    void print(int x) { cout << "Int: " << x; }      // 重载：参数数量不同
    void print(int x, string s) { cout<< s << ": " << x; }
};
```

重写是​运行时多态​，通过虚表（vtable）在运行时决定调用哪个函数。参数列表、返回类型必须与基类虚函数完全一致，可以使用override显式标记。

```c++
class Animal {
public:
    virtual void sound() { cout << "Animal sound" << endl; } // 基类虚函数
};

class Dog : public Animal {
public:
    void sound() override { cout << "Dog barks" << endl; }   // 重写基类虚函数
};

int main() {
    Animal* animal = new Dog();
    animal->sound(); // 输出 "Dog barks"（运行时多态）
    delete animal;
}
```

## 引用在声明时是否可以不初始化？

**不可以**。引用必须在声明时立即绑定到一个有效对象，否则编译失败。
如果C++的引用在声明时不立刻绑定一个对象会有以下几种情况发生：
- 编译失败：主流编译器如GCC等规定引用在声明时**必须绑定**到一个已存在的有效对象，如果没有不会生成可执行文件。
- 运行时出现未定义行为：若通过某些方式绕过编译检查，会出现访问一块未知内存地址的情况，会出现崩溃、数据损坏等后果。
- 出现悬空引用：引用绑定到临时对象或已被销毁的对象，然后对象被销毁后会出现悬空的情况

## C++中的原子变量？

原子变量是C++11引入的核心多线程同步工具，用于实现无锁线程安全操作，避免数据竞争并优化性能。
原子操作意味着一个操作要么未开始，要么已经完成，不会有处于中间状态的情况。
原子变量通过内存序参数解决编译器和CPU乱序执行问题，确保多线程环境下操作的可见性和顺序一致性。
原子变量的**底层实现原理**：

- X86架构：使用lock前缀指令（如lock add）锁定总线，阻止其他核心访问内存，确保操作独占执行。
- ARM架构：通过LDREX/STREX指令实现乐观锁

原子变量的适用场景：计数器、标志位等轻量同步场景，提升并发效率，但是复杂场景下仍然需要借助互斥锁等工具。
