---
title: C++——智能指针剖析
date: 2024-11-19 23:31:14
categories:
- C++
- C++知识
tags: 
- shared_ptr
- weak_ptr
- unique_ptr
- enable_from_this_shared
typora-root-url: ./..
---



参考：

[恋恋风辰官方博客](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/23rkkdskq4fAYGIkaTEP8c45CHR)

[动态内存管理 - cppreference.com](https://zh.cppreference.com/w/cpp/memory#.E6.99.BA.E8.83.BD.E6.8C.87.E9.92.88)

[SRombauts/shared_ptr： 一个最小的 shared/unique_ptr 实现，用于处理 boost/std：：shared/unique_ptr 不可用的情况。](https://github.com/SRombauts/shared_ptr)

[C++智能指针_c++ 智能指针-CSDN博客](https://blog.csdn.net/LCZ411524/article/details/143648637)

[当我们谈论shared_ptr的线程安全性时，我们在谈论什么 - 知乎](https://zhuanlan.zhihu.com/p/416289479)

[C++ 智能指针 - 全部用法详解-CSDN博客](https://blog.csdn.net/cpp_learner/article/details/118912592)

------

# 序言

传统的手动管理内存方式（如 `new` 和 `delete`）虽然灵活，但也容易引发**内存泄漏**（new的对象在作用域结束后没有被及时释放）、**悬空指针**（指针的指向对象已被删除或释放，但仍有其他指针保留了对该内存位置的引用）和**重复释放**（一个指针指向的内存被多次重复释放）等问题。随着项目规模的扩大和代码复杂性的增加，这些问题不仅让程序员疲于奔命，也直接影响了软件的可靠性和可维护性。

智能指针就是为了实现类似于Java中的垃圾回收机制。Java的垃圾回收机制使程序员从繁杂的内存管理任务中彻底的解脱出来，在申请使用一块内存区域之后，无需去关注应该何时何地释放内存，Java将会自动帮助回收。但是出于效率和其他原因（可能C++设计者不屑于这种傻瓜式的编程方式），C++本身并没有这样的功能，其繁杂且易出错的内存管理也一直为广大程序员所诟病。

更进一步地说，智能指针的出现是为了满足管理类中指针成员的需要。包含指针成员的类需要特别注意复制控制和赋值操作，原因是复制指针时只复制指针中的地址，而不会复制指针指向的对象（一块内存地址可能被多个对象所指向）。当类的实例在析构的时候，可能会导致**垂悬指针**问题（即指针的指向对象已被删除或释放，但仍有其他指针保留了对该内存位置的引用）。

**管理类中指针成员的方法一般有两种方式：**一种是采用值型类，这种类是给指针成员提供值语义（value semantics），当复制该值型对象时，会得到一个不同的新副本。这种方式典型的应用是string类。另外一种方式就是智能指针，实现这种方式的指针所指向的对象是共享的。

智能指针不仅提供了内存管理的自动化，还增强了代码的安全性和可读性，是现代 C++ 中推荐的内存管理方式之一。本篇文章旨在系统地介绍 C++ 标准库中的三种常用智能指针：`std::unique_ptr`、`std::shared_ptr` 和 `std::weak_ptr`。

智能指针并不是指针，而是行为类似于指针的类对象，这种对象具有指针不包含的其他功能。简单来说，智能指针能帮助我们管理动态分配的内存，它会帮助我们自动释放new出来的内存，从而**避免 `new` 和 `delete`引发的一系列问题**，比如内存泄漏、悬空指针和重复释放等。

C++里面有四个智能指针：`auto_ptr`、`shared_ptr`、`unique_ptr`、`weak_ptr`。其中后三个是C++11支持的，并且第一个已经在C++11弃用，这里我们只学习后三个。

# 1. shared_ptr

这三类智能指针模板都定义了类似指针的对象，可以将`new`获得（直接或间接）的地址赋给这种对象，当智能指针过期时，其析构函数将使用`delete`来释放内存。因此，如果将`new`返回的地址赋给这些对象，将无需记住稍后释放这些内存：**在智能指针过期时，这些内存会自动释放（RAII）**。

> 我们可以回顾一下我们在[并发编程（10）](https://www.aichitudou.cn/2024/11/12/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%EF%BC%8810%EF%BC%89%E2%80%94%E2%80%94%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B%E5%92%8C%E5%8E%9F%E5%AD%90%E6%93%8D%E4%BD%9C/)这篇文章中提到的问题：`shared_ptr`是线程安全的吗？我只简单做了回答，在这篇文章中，我将在下面详细分析。

## 1.1 shared_ptr 内存模型

`shared_ptr`有以下两个作用：

1. std::`shared_ptr`使用**引用计数**，每一个`shared_ptr`的拷贝都指向相同的内存。只有在最后一个`shared_ptr`副本对象析构的时候，内存才会被释放。
2. `sharedd_ptr`共享被管理的对象，同一时刻可以有多个`shared_ptr`拥有对象的所有权（多线程中，不同线程可以对指向同一内存的`sharedd_ptr`**副本**中的数据进行安全访问或修改，但多个线程不能对**同一个**`sharedd_ptr`对象中的数据进行修改），当最后一个`shared_ptr`对象销毁时，被管理对象自动销毁。

简单来说，`shared_ptr`实现包含了两个部分：

- 一个指向堆上创建的对象的裸指针 `raw_ptr`。
- 一个指向内部隐藏的、共享的管理对象 `shared_count_object`。其中`use_count`是当前这个堆上对象被多少对象引用了，简单来说就是引用计数。

![共享指针 UML](/images/$%7Bfiilename%7D/687474703a2f2f7a68616f79616e2e776562736974652f78696e7a68692f6370702f68746d6c2f706963732f7368617265642e706e67.png)

<center>图片来源：https://github.com/SRombauts/shared_ptr</center>

如上图所示，`shared_ptr`内部包含了两个指针，一个`Ptr to T`指向目标管理对象`T object`，另一个`Ptr to Control Block`指向控制块`Control Block`。控制块包含了一个`引用计数(reference count)`、一个`弱计数(weak count)`和`其他数据(other data)`（比如删除器、分配器等）。

引用计数会累加共享同一块资源（内存）的`shared_ptr`对象数量，是`shared_ptr`的核心，在任何情况下都是线程安全的（因为引用计数的实现过程是原子操作）。

> 为了满足线程安全的要求，引用计数通常使用类似于 `std::atomic::fetch_add` 的操作并结合 `std::memory_order_relaxed` 进行递增（递减操作则需要更强的内存排序，以确保控制块能够安全销毁）。

简单举一个例子：

```cpp
std::shared_ptr<int> p1(new int(1));
std::shared_ptr<int> p2=p1;
```

`shared_ptr`有很多构造函数，这里使用的构造函数原型为：

```cpp
template< class Y >
explicit shared_ptr( Y* ptr );

template< class Y >
shared_ptr& operator=( const shared_ptr<Y>& r ) noexcept;
```

二者的内存模型如下所示：

![在这里插入图片描述](/images/$%7Bfiilename%7D/db2e0303e59f4c5c99a40f388e6bb3be.png)

<center>图片来源：https://blog.csdn.net/LCZ411524/article/details/143648637</center>

很明显，p1和p2都指向同一内存空间`T Object`，而且引用计数为2，只有当p1和p2都被释放后，引用计数减为0的同时，智能指针管理的对象才会被释放。

## 1.2 shared_ptr的使用

上面多次说过，通过`new`和`delete`创建的对象存在很多隐患，但我还要在这里重复提醒：

1. 当一个函数返回**局部变量**的指针（非new创建）时，外部使用该指针可能会造成崩溃或逻辑错误。因为局部变量随着函数的右}释放了。
2. 当在作用域内使用new创建的对象在作用域结束后仍未被delete，那么该内存不存在任何对象指向（内存泄漏）。
3. 如果多个指针指向同一个堆空间，其中一个释放了堆空间，使用其他的指针时会造成崩溃（悬空指针）。
4. 对一个指针多次delete，会造成double free问题（重复释放）。
5. 两个类对象A和B，分别包含对方类型的指针成员，互相引用时如何释放是个问题。

### 1.2.1 make_shared

我们可以通过构造函数来创建一个智能指针，也可以通过`make_shared`来构造智能指针，但更推荐后者，因为：

1. `std::make_shared` **减少了内存分配的次数**：
   - **使用 `new` 创建：** 当直接使用 `std::shared_ptr` 时，需要两次内存分配：
     1. 为所管理的对象分配内存。
     2. 为 `std::shared_ptr` 的控制块（控制引用计数和资源信息）分配内存。
   - **使用 `make_shared`：** `std::make_shared` 会在一次内存分配中同时分配对象和控制块的内存，避免了额外的内存分配。

```cpp
#include <memory>

// 使用 new 创建
std::shared_ptr<int> sp1(new int(10));  // 两次内存分配

// 使用 make_shared 创建
std::shared_ptr<int> sp2 = std::make_shared<int>(10);  // 一次内存分配
```

2. 直接使用 `new` 创建 `std::shared_ptr` 可能引发异常时的资源泄漏问题。
   - 如果在 `std::shared_ptr` 的构造过程中发生异常，`new` 分配的资源可能无法正确释放，导致内存泄漏。
   - `std::make_shared` 是异常安全的，因为其分配和构造过程是一体化的，保证资源不会泄漏。

```cpp
// 错误代码
void exception_test() {
    std::shared_ptr<int> sp(new int[100]);  // 动态分配数组
    throw std::runtime_error("Error occurred");  // 如果异常发生，数组内存泄漏
}

// 正确代码
void exception_test() {
    std::shared_ptr<int> sp = std::make_shared<int[100]>();  // 异常安全，资源会正确管理
}
```

3. 当直接使用 `new` 时，需要确保动态分配的内存与 `std::shared_ptr` 的删除器匹配。
   - 如果使用默认删除器管理动态分配的数组，会导致未定义行为（数组不会被正确释放）。
   - `std::make_shared` 自动匹配删除器，避免了这种错误。

```cpp
// 错误代码
void test() {
    std::shared_ptr<int> sp(new int[10]);  // 错误！默认删除器无法正确释放数组
}
// 正确代码
void test() {
    std::shared_ptr<int[]> sp = std::make_shared<int[]>(10);  // 正确，删除器自动匹配
}
```

但注意，当存在以下情况时，不应该使用`make_shared`来构造`shared_ptr`对象，而应直接构造`shared_ptr`对象：

1. 需要自定义删除器

   `std::make_shared` 自动使用 `delete` 来销毁对象，但如果我们创建对象管理的资源不是通过`new`分配的内存，那么需要我们自定义一个删除器来销毁该内存；或者我们需要为 `std::shared_ptr` 提供自定义的删除逻辑（例如释放资源时需要执行额外的操作），那么 `std::make_shared` 就不适用了。在这种情况下，我们需要通过`shared_ptr`的构造函数来创建对象，并传递一个自定义的删除器。

2. 创造对象的构造函数是保护或私有时

   **当我们想要创建的对象没有公有的构造函数时，`make_shared`无法使用！！！**

3. 从已有裸指针构造，当对象已通过 new 分配，或由第三方库返回裸指针时，不能用 `make_shared`

4. 使用 `make_shared` 时，对象和控制块的内存是连续的。若希望对象销毁后立即释放其内存（即使仍有 `weak_ptr` 引用控制块），需直接构造 `shared_ptr`。因为 **`std::make_shared`** 分配的对象和控制块的内存是连续的，当所有 `std::shared_ptr` 销毁后，对象和控制块的**连续内存块不会立即释放**，因为控制块仍需维护弱引用计数（供 `std::weak_ptr` 使用），只有当所有 `std::weak_ptr` 也销毁后（强引用和弱引用都为0时），**整个内存块（控制块+对象）才会释放**。

   而通过 **`std::shared_ptr`** 直接构造，当所有 `std::shared_ptr` 销毁后，对象的内存（通过 `new` 分配的独立内存块）**立即释放**，控制块的内存仍存在，直到所有 `std::weak_ptr` 销毁后才释放。

   因此 make_shared 会意外延迟内存释放的时间。

------

简单使用：

```cpp
//定义一个指向整形5的指针
auto psint2 = make_shared<int>(5);
//判断智能指针是否为空
if (psint2 != nullptr)
{
    cout << "psint2 is " << *psint2 << endl;
}

auto psstr2 = make_shared<string>("hello zack");
if (psstr2 != nullptr && !psstr2->empty())
{
    cout << "psstr2 is " << *psstr2 << endl;
}
```

对于智能指针的使用和普通的内置指针没什么区别，通过判断指针是否为`nullptr`可以判断是否为空指针。通过`->`可以取指针内部的成员方法或者成员变量。

当我们需要获取内置类型（管理资源）时，可以通过智能指针的方法`get()`返回其底层管理的内置指针。

> 注意，通过`get()`函数返回的内置指针时要注意以下问题：

1. 我们不能主动通过`delete`回收该指针，要交给智能指针自己回收，否则会造成double free或者使用智能指针产生崩溃等问题。
2. 也不能用`get()`返回的内置指针初始化另一个智能指针，因为两个智能指针引用一个内置指针会出现问题，比如一个释放了另一个不知道就会导致崩溃等问题。 因为`get()` 方法返回的原始指针（即裸指针），不增加智能指针对对象的引用计数或所有权管理

```cpp
std::shared_ptr<int> sp1 = std::make_shared<int>(10);
int* raw_ptr = sp1.get();

// 错误：使用裸指针初始化另一个 shared_ptr
std::shared_ptr<int> sp2(raw_ptr);  // 错误，sp2 和 sp1 都会管理同一个内存
```

这里，`raw_ptr` 是 `sp1` 管理的对象的裸指针，但 `raw_ptr` 不会增加对象的引用计数，也不会管理其生命周期。当我们通过 `raw_ptr` 初始化 `sp2` 时，`sp2` 会成为一个**新的智能指针，指向相同的内存区域**。由于 `sp1` 和 `sp2` 都管理同一个内存对象，但它们并没有共享引用计数。裸指针的生命周期与智能指针不同，它不被智能指针的生命周期管理，这可能会导致以下错误：

- **多次释放同一内存**：如果两个智能指针都拥有相同的裸指针，而其中一个智能指针释放了这个指针所管理的资源，另一个智能指针会在其析构时试图释放相同的资源。这会导致“双重释放”错误，通常会导致程序崩溃。
- **悬挂指针**：如果原始智能指针在另一个智能指针之前被销毁，那么另一个智能指针会变成一个悬挂指针。虽然这个智能指针指向有效内存，但该内存已被释放，访问它会导致未定义行为（通常会崩溃）。

> `get()`用来将指针的访问权限传递给代码，只有在确定代码不会delete裸指针的情况下，才能使用get。**特别是**，永远不要用get初始化另一个智能指针或者为另一个智能指针赋值。

### 1.2.2 shared_ptr与new结合

我们可以传给`shared_ptr`一个new构造的指针对象：

```cpp
auto psint = shared_ptr<int>(new int(5));
auto psstr = shared_ptr<string>(new string("hello zack"));
```

原型在上面也说过，是

```cpp
template< class Y >
explicit shared_ptr( Y* ptr );
```

因为该构造函数是`explicit`的。因此，我们**不能将一个内置指针隐式转换为一个智能指针**，必须使用直接初始化形式来初始化一个智能指针：

```cpp
//错误，不能用内置指针隐式初始化shared_ptr
// shared_ptr<int> psint2 = new int(5);
//正确，显示初始化
shared_ptr<string> psstr2(new string("good luck"));
```

除了智能指针之间的赋值（赋值构造函数）外，还以通过一个智能指针构造另一个

```cpp
shared_ptr<string> psstr2(new string("good luck"));
//可以通过一个shared_ptr 构造另一个shared_ptr
shared_ptr<string> psstr3(psstr2);
shared_ptr<string> psstr4;
psstr4 = psstr2;
```

**以上方法构造的智能指针都共享同一个引用计数**。

在构造智能指针的同时，可以指定自定义的删除方法替代`shared_ptr`本身内置的`delete`操作：

```cpp
//可以设置新的删除函数替代delete
shared_ptr<string> psstr4(new string("good luck for zack"), delfunc);

void delfunc(string *p)
{
    if (p != nullptr)
    {
        delete (p);
        p = nullptr;
    }

    cout << "self delete" << endl;
}
```

我们实现了自己的delfunc函数作为删除器，回收了内置指针，并且打印了删除信息。这样当psstr4执行析构时，会打印”self delete”。

### 1.2.3 reset()

- `reset()`不带参数时，若智能指针`s`是唯一指向该对象的指针，则释放，并置空。若智能指针`s`不是唯一指向该对象的指针，则引用计数减一，同时将`s`置为空。
- `reset()`带参数时，若智能指针`s`是唯一指向该对象的指针，则释放并指向新的对象。若智能指针`s`不是唯一指向该对象的指针，则引用计数减一，并指向新的对象。

```cpp
p.reset() ; //将p重置为空指针，所管理对象引用计数 减1
p.reset(p1); //将p重置为p1（的值）,p管控的对象计数减1，p接管对p1指针的管控
p.reset(p1,d); //将p重置为p1（的值），p 管控的对象计数减1并使用d作为删除器
```

`reset()`的功能是为`shared_ptr`重新开辟一块新的内存，让`shared_ptr`绑定这块内存

```cpp
shared_ptr<int> p(new int(5));
// p重新绑定新的内置指针
p.reset(new int(6));
```

上述代码为p重新绑定了新的内存空间。

`reset`常用的情况是判断智能指针**是否独占内存**，如果引用计数为1，也就是自己独占内存就去修改，否则就为智能指针绑定一块新的内存进行修改，防止多个智能指针共享一块内存，一个智能指针修改内存导致其他智能指针受影响。

```cpp
//如果引用计数为1，unique返回true
if (!p.unique())
{
    //还有其他人引用，所以我们为p指向新的内存
    p.reset(new int(6));
}
// p目前是唯一用户
*p = 1024;
```

使用智能指针的另一个好处就是，当程序崩溃时，智能指针也能保证内存空间被回收

```cpp
void execption_shared()
{
    shared_ptr<string> p(new string("hello zack"));
    //此处导致异常
    int m = 5 / 0;
    //即使崩溃也会保证p被回收
}
```

即使运行到 m = 5 / 0处，程序崩溃，智能指针p也会被回收。有时候我们传递个智能指针的指针不是new分配的，那就需要我们自己给他传递一个删除器：

```cpp
void delfuncint(int *p)
{
    cout << *p << " in del func" << endl;
}

void delfunc_shared()
{
    int p = 6;
    shared_ptr<int> psh(&p, delfuncint);
}
```

如果不传递`delfuncint`，会造成`p`被智能指针`delete`，因为`p`是栈空间的变量，用`delete`会导致崩溃。

### 1.2.4 转移所有权

`std::shared_ptr` 是一个共享式的智能指针，允许多个 `shared_ptr` 实例共同管理同一个资源。资源的所有权是通过 **引用计数** 来管理的，因此你可以有多个 `shared_ptr` 共享同一个资源。

- **复制构造函数**：当一个 `shared_ptr` 被复制时，引用计数增加，所有权没有“转移”，只是增加了共享者。
- **移动构造函数**：当一个 `shared_ptr` 被用作另一个 `shared_ptr` 的构造参数时，原 `shared_ptr` 的所有权转移给目标 `shared_ptr`，同时原 `shared_ptr` 的引用计数会减少。
- **移动赋值运算符**：当一个 `shared_ptr` 被赋值给另一个 `shared_ptr` 时，所有权转移。

举例说明：

```cpp
// 创建一个 shared_ptr，指向一个整型值
std::shared_ptr<int> p1 = std::make_shared<int>(10);
std::cout << "p1 use_count: " << p1.use_count() << std::endl;  // 输出 1

// 通过移动构造函数将所有权转移给 p2
std::shared_ptr<int> p2(std::move(p1));  // 移动构造
std::cout << "p1 use_count after move: " << p1.use_count() << std::endl;  // 输出 0
std::cout << "p2 use_count after move: " << p2.use_count() << std::endl;  // 输出 1
```

在 `p1` 到 `p2` 的转移过程中，`p2` 接管了原本由 `p1` 管理的资源，移动操作后，`p1` 的引用计数为 0，表示 `p1` 不再拥有资源，`p2` 的引用计数变为 1，表示 `p2` 成为唯一的所有者。

```cpp
// 创建一个 shared_ptr，指向一个整型值
std::shared_ptr<int> p1 = std::make_shared<int>(20);
std::cout << "p1 use_count: " << p1.use_count() << std::endl;  // 输出 1

// 移动赋值
std::shared_ptr<int> p2;
p2 = std::move(p1);  // 移动赋值
std::cout << "p1 use_count after move: " << p1.use_count() << std::endl;  // 输出 0
std::cout << "p2 use_count after move: " << p2.use_count() << std::endl;  // 输出 1
```

`p1` 通过移动赋值操作将资源的所有权转移给了 `p2`。移动赋值后，`p1` 的引用计数变为 0，因为它不再管理该资源。`p2` 的引用计数变为 1，表示 `p2` 现在是该资源的唯一所有者。

> **`shared_ptr`** 通过引用计数机制共享资源，`std::move()` 触发移动构造函数，转移所有权，且原指针不再管理资源，引用计数相应减少。

### 1.2.5 通过智能指针共享数据

我们定义一个 `StrBlob` 类，该类通过 `shared_ptr` 实现智能指针管理，用于共享一个 `vector<string>` 类型的容器。

```cpp
class StrBlob
{
public:
    //定义类型，用于表示 StrBlob 中存储的元素的数量
    typedef std::vector<string>::size_type size_type;
    StrBlob();
    //通过初始化列表构造
    StrBlob(const initializer_list<string> &li);
    //返回vector大小
    size_type size() const { return data->size(); }
    //判断vector是否为空
    bool empty()
    {
        return data->empty();
    }
    //向vector写入元素
    void push_back(const string &s)
    {
        data->push_back(s);
    }
    //从vector弹出元素
    void pop_back();
    //访问头元素
    std::string &front();
    //访问尾元素
    std::string &back();

private:
    shared_ptr<vector<string>> data;
    //检测i是否越界
    void check(size_type i, const string &msg) const;
    // 判断容器是否无元素的标志
    string badvalue;
};
```

该类只实现了默认构造函数和一个带有初始化列表参数的构造函数，后者允许我们通过初始化列表（例如：`{"one", "two", "three"}`）来初始化 `StrBlob` 对象。但因为`StrBlob`未重载赋值运算符，也没有实现拷贝构造函数，所以`StrBlob`对象之间的赋值是**浅拷贝**（浅拷贝只赋值对象本身的值或引用，并不会复制对象所引用的内存或对象，也就是浅拷贝只创建了一个新的对象，并将原对象的指针或引用直接复制到新对象中，而没有复制指针所指向的数据），因而内部成员data会随着`StrBlob`对象的赋值修改引用计数（浅拷贝）。

当然我们也可以实现拷贝构造函数和赋值运算符，但只能用浅拷贝的方式实现，不能实现深拷贝（即拷贝指针指向的对象，而不是拷贝指针本身，这样即使其他指针消失，但仍然不影响该指针指向这块内存）

```cpp
// 默认构造函数
StrBlob::StrBlob()
{
    data = make_shared<vector<string>>();
}
// 复制构造函数// 列表初始化
StrBlob::StrBlob(const StrBlob &other)
{
    data = sb.data;
}
// 列表初始化
StrBlob::StrBlob(const initializer_list<string> &li)
{
    data = make_shared<vector<string>>(li);
}
// 赋值运算符
StrBlob& operator=(const StrBlob &other)
{
    if (this != &other) {
        data = other.data;
    }
    return *this;
}
```

> 注意，将`data = other.data`修改为`data(std::make_shared<std::vector<std::string>>(*other.data))`或`data = std::make_shared<std::vector<std::string>>(*other.data)`即可更正为深拷贝。

**浅拷贝**：只复制对象的值或引用，多个对象共享同一个动态分配的内存空间，修改其中一个对象的数据会影响到另一个对象。

**深拷贝**：复制对象本身的值，并且对对象引用的内存进行递归复制，确保每个对象拥有独立的内存，修改其中一个对象的数据不会影响另一个对象。

![4000b04ed8b33ea0df35a01f5972dfb](/images/$%7Bfiilename%7D/4000b04ed8b33ea0df35a01f5972dfb.jpg)

<center>图片来源：C++ primer plus：深拷贝</center>

实现检查越界的函数：

```cpp
//检测i是否越界
void StrBlob::check(size_type i, const string &msg) const
{
    if (i >= data->size())
    {
        throw out_of_range(msg);
    }
}
```

实现front，访问首元素：

```cpp
string& StrBlob::front()
{
    //不要返回局部变量的引用
    if (data->size() <= 0)
    {
        return badvalue;
    }
    return data->front();
}
```

如果队列为空，返回一个空的字符串。但是如果我们直接构造一个空字符串返回，这样就返回了局部变量的引用，局部变量会随着函数结束而释放，造成安全隐患。所以我们可以返回类的成员变量`badvalue`，作为队列为空的标记。当然如果不能容忍队列为空的情况，可以通过抛出异常来处理，那我们用这种方式改写front：

```cpp
string &StrBlob::front()
{
    check(0, "front on empty StrBlob");
    return data->front();
}
```

同样实现back()和pop_back()：

```cpp
string &StrBlob::back()
{
    check(0, "back on empty StrBlog");
    return data->back();
}
void StrBlob::pop_back()
{
    check(0, "back on pop_back StrBlog");
    data->pop_back();
}
```

这样我们通过定义StrBlob类，达到共享vector的方式。多个StrBlob操作的是一个vector向量。

我们可以通过`use_count`函数获得当前托管指针的引用计数：

```cpp
void StrBlob::printCount()
{
    cout << "shared_ptr use count is " << data.use_count() << endl;
}
```

### 1.2.6 shared_ptr是线程安全的吗？

现在我们可以很简单的回答这个问题：**并不是**。

> 引用计数是线程安全的！！！

`shared_ptr`仅有引用计数是**线程安全**的，因为在`shared_ptr`的控制块中，引用计数变量使用类似于 `std::atomic::fetch_add` 的操作并结合 `std::memory_order_relaxed` 进行递增（递减操作则需要更强的内存排序，以确保控制块能够安全销毁）。其关键在于使用了**原子操作**对引用计数进行增加或减少，所以是线程安全的。    而且，因为引用计数是线程安全的，多个线程可以安全地操作引用计数和访问管理对象，即使这些 `shared_ptr` 实例是同一对象的副本且共享所有权也是如此，所以**管理共享资源的生命周期是线程安全的**，不用担心因为多线程操作导致资源提早释放或延迟释放。

> `shared_ptr`本身并不是线程安全的！！！

但是**`shared_ptr`本身并不是线程安全的**，`shared_ptr` 对象实例包含一个指向控制块的指针和一个指向底层元素的指针。这两个指针的操作在多个线程中并没有同步机制。因此，如果多个线程同时访问同一个 `shared_ptr` 对象实例并调用非 `const` 成员函数（如 `reset` 或 `operator=`），这些操作会导致对这些指针的并发修改，进而引发数据竞争（就像我们在[并发编程（10](https://www.aichitudou.cn/2024/11/12/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%EF%BC%8810%EF%BC%89%E2%80%94%E2%80%94%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B%E5%92%8C%E5%8E%9F%E5%AD%90%E6%93%8D%E4%BD%9C/)）中说的一样，独立的原子操作当然是线程安全的，但是如果原子操作依赖于非原子操作，那么这个过程可能就是非线程安全的）。举例：

***情况一：当多线程操作同一个shared_ptr对象时***

```cpp
// 按指针传入
void fn(shared_ptr<A>* sp) {
    ...
}
// 按引用传入
void fn(shared_ptr<A>& sp) {
        ...
    if (..) {
        sp = other_sp;
    } else if (...) {
        sp = other_sp2;
    }
}

std::thread t1(fn, std::ref(sp1));
std::thread t2(fn, std::ref(sp1));
```

如果我们将`shared_ptr`对象的**指针或引用**传入给可调用对象，当创建不同线程对`shared_ptr`进行修改时，比如修改其指向（如 `reset` 或 `operator=`）。`sp`原先指向的引用计数的值要减去`1`，`other_sp`指向的引用计数值要加`1`。然而这几步操作加起来并不是一个原子操作（[并发编程（10)](https://www.aichitudou.cn/2024/11/12/并发编程（10）——内存模型和原子操作/)在原子操作中说过，并不是所有原子操作都是线程安全的，如果原子操作依赖于非原子操作，那么这个过程可能就是非线程安全的，这里的条件判断并不是原子操作），如果多少线程都在修改sp的指向的时候，那么可能会出问题。比如在导致计数在操作减一的时候，其内部的指向，已经被其他线程修改过了。引用计数的异常会导致某个管理的对象被提前析构，后续在使用到该数据的时候触发`core dump`。

如果不调用`shared_ptr`的`非const`成员函数修改`shared_ptr`，那么就是线程安全的。

***情况二：当多线程操作不同shared_ptr对象时***

如果不是同一 `shared_ptr` 对象（管理的数据是同一份，引用计数共享，但`shared_ptr`不是同一个对象），每个线程读写的指针也不是同一个，引用计数又是线程安全的，那么自然不存在数据竞争，可以安全的调用所有成员函数。

```cpp
// 按值传递
void fn(shared_ptr<A> sp) {
    ...
    if (..) {
        sp = other_sp;
    } else if (...) {
        sp = other_sp2;
    }
}
std::thread t1(fn, std::ref(sp1));
std::thread t2(fn, std::ref(sp1));
```

这时候每个线程内看到的sp，他们所管理的是同一份数据，用的是同一个引用计数。但是各自是不同的对象，当发生多线程中修改sp指向的操作的时候，是不会出现非预期的异常行为的，也就是线程安全的。

> `shared_ptr`所管理的对象不是线程安全的！！！

尽管前面我们提到了如果是按值捕获（或传参）的`shared_ptr`对象，那么是该对象是线程安全的。然而话虽如此，但却可能让人误入歧途。因为我们使用`shared_ptr`更多的是操作其中的数据，对齐管理的数据进行读写，而不是修改`shared_ptr`的指向。尽管在按值捕获的时候`shared_ptr`本身是线程安全的，我们不需要对此施加额外的同步操作（比如加解锁、条件变量、call_once 和once_flag ），但是这并不意味着`shared_ptr`所管理的对象是线程安全的！

`shared_ptr`本身和`shared_ptr`管理的对象是两个东西，并不是同一个！！！*如果我们要多线程处理shared_ptr所管理的资源，我们需要主动的对其施加额外的同步操作（比如加解锁、条件变量、call_once 和once_flag）*。

如果shared_ptr管理的数据是STL容器，那么多线程如果存在同时修改的情况，是极有可能触发core dump的。比如多个线程中对同一个[vector](https://zhida.zhihu.com/search?content_id=180638993&content_type=Article&match_order=1&q=vector&zhida_source=entity)进行push_back，或者对同一个map进行了insert。甚至是对STL容器中并发的做clear操作，都有可能出发core dump，当然这里的线程不安全性，其实是其所指向数据的类型的线程不安全导致的，并非是shared_ptr本身的线程安全性导致的。尽管如此，由于shared_ptr使用上的特殊性，所以我们有时也要将其纳入到shared_ptr相关的线程安全问题的讨论范围内。

除了STL容器的并发修改操作（这里指的是修改容器的结构，并不是修改容器中某个元素的值，后者是线程安全的，前者不是），protobuf的Message对象也是不能并发操作的，比如一个线程中修改Message对象（set、add、clear），另外一个线程也在修改，或者在将其序列化成字符串都会触发core dump。

STL容器如何解决线程安全可以参考这篇文章：[C++ STL容器如何解决线程安全的问题？ - 知乎](https://www.zhihu.com/question/29987589/answer/1483744520)

简单来说，保证STL容器的线程安全有两种方式：

1. 对其施加同步操作，使容器在进行增删改查时是线程安全的，但是在高并发下，同步操作会造成性能开销
2. 如果是非关联容器，比如vector，那就固定vector的大小，避免动态扩容（无push_back），我们可以使用resize来实现（不是reserve）。reserve就是预留内存。为的是避免内存重新申请以及容器内对象的拷贝。说白了，reserve()是给push_back()准备的！而resize除了预留内存以外，还会调用容器元素的构造函数，不仅分配了N个对象的内存，还会构造N个对象。从这个层面上来说，resize()在时间效率上是比reserve()低的。但是在多线程的场景下，用resize再合适不过。你可以resize好N个对象，多线程不管是读还是写，都是通过容器的下标访问【operator[]】来访问元素，不要push_back新元素。所谓的『写操作』在这里不是插入新元素，而是修改旧元素。而且，我们也可以将固定大小的vector修改成一个环形队列，索引通过原子变量来保证索引的安全。
3. 至于非关联容器，比如map，我记忆中还是使用互斥好一点



最后，有很多人可能认为引用计数是通过智能指针的静态成员变量所管理的，但这很明显是错的：

```cpp
shared_ptr<A> sp1 = make_shared<A>(x);
shared_ptr<A> sp2 = make_shared<A>(y);
```

两个完全不相干的sp1和sp2，只要模板参数`T`是同一个类型，即使管理的资源不是同一个，但如果使用静态成员变量管理引用计数，那么二者就会共享同一个计数。

# 2. weak_ptr

我们在`shared_ptr`说到了，`new和delete`可能会引发**内存泄漏**问题（作用域内new的对象在作用域结束后仍未delete，此时没有任何对象指向这片内存），但是`shared_ptr`本身也可能会引发内存泄漏问题，即**循环引用问题**。

## 2.1 循环引用问题

shared_ptr 循环引用问题是指**两个或多个对象之间通过`shared_ptr`相互引用，导致对象无法被正确释放，从而造成内存泄漏。**常见的情况是两个对象A和B，它们的成员变量互相持有了对方的shared_ptr。当A和B都不再被使用时，它们的引用计数不会降为0，无法被自动释放。比如：

```cpp
class Girl;

class Boy {
public:
	Boy() {
		cout << "Boy 构造函数" << endl;
	}
	~Boy() {
		cout << "~Boy 析构函数" << endl;
	}
	void setGirlFriend(shared_ptr<Girl> _girlFriend) {
		this->girlFriend = _girlFriend;
	}
private:
	shared_ptr<Girl> girlFriend;
};

class Girl {
public:
	Girl() {
		cout << "Girl 构造函数" << endl;
	}
	~Girl() {
		cout << "~Girl 析构函数" << endl;
        
	}
	void setBoyFriend(shared_ptr<Boy> _boyFriend) {
		this->boyFriend = _boyFriend;
	}
private:
	shared_ptr<Boy> boyFriend;
};

void useTrap() {
	shared_ptr<Boy> spBoy(new Boy());
	shared_ptr<Girl> spGirl(new Girl());
	// 陷阱用法
	spBoy->setGirlFriend(spGirl);
	spGirl->setBoyFriend(spBoy);
	// 此时boy和girl的引用计数都是2
    cout << "r_count of spBoy is : " << spBoy.use_count() << endl;
    cout << "r_count of spGirl is : " << spGirl.use_count() << endl;
}

int main(void) {
	useTrap();
	system("pause");
	return 0;
}
```

我们通过`useTrap()`函数创建了两个shared_ptr对象，其中spBoy用于存储一个Boy类，spGirl用于存储一个Girl类；但是，Boy类中有一个智能指针变量用于存储Girl类，而Girl类中有一个智能指针变量用于存储Boy类。如果给智能指针spBoy和spGirl管理对象的成员变量赋值，那么会造成循环引用问题。此时，创建的智能指针无法被销毁，因为引用计数总不为0。

代码输出为：

```cpp
Boy 构造函数
Girl 构造函数
r_count of spBoy is : 2
r_count of spGirl is : 2
```

确实，因为二者的引用计数总为2或1，这两个类不能被正确析构。

有没有方法解决这个问题呢？这时候我们就用到了智能指针**`weak_ptr`**。

------

**`weak_ptr`**是一种**弱引用**，不会增加对象的引用计数，在对象释放时会自动设置为`nullptr`。它**只可以**从一个 `shared_ptr` 或另一个 `weak_ptr` 对象构造， 它的构造和析构不会引起引用记数的增加或减少（当然，`weak_ptr`其实不需要析构函数，因为它不需要管理和释放资源，即没有RAII）。 同时`weak_ptr` 没有重载`operator*`和`operator->`（不支持访问资源），但可以使用 **weak_ptr.lock()** 获得一个可用的 `shared_ptr` 对象，当对象已经释放时会返回一个空`shared_ptr`。

那么，上面的代码可以修改为：

```cpp
class Girl;

class Boy {
public:
	Boy() {
		cout << "Boy 构造函数" << endl;
	}
	~Boy() {
		cout << "~Boy 析构函数" << endl;
	}
	void setGirlFriend(shared_ptr<Girl> _girlFriend) {
		this->girlFriend = _girlFriend;
		// 在必要的使用可以转换成共享指针
		shared_ptr<Girl> sp_girl;
		sp_girl = this->girlFriend.lock();
		cout << "r_count of spGirl is : " << sp_girl.use_count() << endl;
		// 使用完之后，再将共享指针置NULL即可
		sp_girl = NULL;
	}
private:
	weak_ptr<Girl> girlFriend;
};

class Girl {
public:
	Girl() {
		cout << "Girl 构造函数" << endl;
	}
	~Girl() {
		cout << "~Girl 析构函数" << endl;
	}
	void setBoyFriend(shared_ptr<Boy> _boyFriend) {
		this->boyFriend = _boyFriend;
	}

private:
	shared_ptr<Boy> boyFriend;
};

void useTrap() {
	shared_ptr<Boy> spBoy(new Boy());
	shared_ptr<Girl> spGirl(new Girl());
	spBoy->setGirlFriend(spGirl);
    cout << "r_count of spGirl is : " << spGirl.use_count() << endl;
	spGirl->setBoyFriend(spBoy);
    cout << "r_count of spBoy is : " << spBoy.use_count() << endl;
    cout << "r_count of spGirl is : " << spGirl.use_count() << endl;
}
int main(void) {
	useTrap();
	system("pause");
	return 0;
}
```

我们将`Boy`类的私有成员变量类型由`shared_ptr`更换为了`weak_ptr`，此时，对该变量赋值不会造成引用计数的增加，自然就解决了循环引用问题。

代码输出为：

```
Boy 构造函数
Girl 构造函数
r_count of spGirl is : 3
r_count of spGirl is : 1
r_count of spBoy is : 2
r_count of spGirl is : 1
~Girl 析构函数
~Boy 析构函数
```

`spGirl`的引用计数之所以为3是因为，在调用`setGirlFriend`函数时，按值传入一个`spGirl`对象，在函数内部，拥有一个`spGirl`副本，此时引用计数为2；当创建了一个`shared_ptr＜Girl>`类型的对象`sp_girl`，并调用`weak_ptr＜Girl>`的`lock`函数获取`shared_ptr＜Girl>`赋予给`sp_girl`时，引用计数变为了3。但它们都是局部变量，所以当函数作用域结束后，都会被自动释放，所以最后引用计数变为了1。

`spBoy`内部的成员变量类型是`shared_ptr`而不是`weak_ptr`，所以引用计数为2，但是，当`spGirl`被释放后，`spBoy`的引用计数为1，此时spBoy也可以正常释放。

## 2.2 其他成员函数

### 2.2.1 use_count()

`weak_ptr`同样可以通过调用`use_count()`方法获取当前观察资源的引用计数

```cpp
shared_ptr<int> sp(new int(10));
weak_ptr<int> wp1(sp);
// 或者
weak_ptr<int> wp2;
wp2 = sp;

cout << wp1.use_count() << endl; //结果为输出1
cout << wp2.use_count() << endl; //结果为输出1
```

### 2.2.2 expired()

通过`expired()`成员函数去检查指向的资源是否过期：判断当前`weak_ptr`智能指针是否还有托管的对象，有则返回`false`，无则返回`true`。

如果返回`true`，等价于 `use_count() == 0`，即已经没有托管的对象了；当然，可能还有析构函数进行释放内存，但此对象的析构已经临近（或可能已发生）。

```cpp
std::weak_ptr<int> gw;

void f() {
	// expired：判断当前智能指针是否还有托管的对象，有则返回false，无则返回true
	if (!gw.expired()) {
		std::cout << "gw is valid\n";	// 有效的，还有托管的指针
	} else {
		std::cout << "gw is expired\n";	// 过期的，没有托管的指针
	}
}

int main() {
	{
		auto sp = std::make_shared<int>(42);
		gw = sp;
		f();
	}
	// 当{ }体中的指针生命周期结束后，再来判断其是否还有托管的指针
	f();

	return 0;
}
```

代码输出：

```cpp
gw is valid
gw is expired
```

在 { } 中，sp的生命周期还在，gw还在托管着make_shared赋值的指针(sp)，所以调用f()函数时打印"gw is valid\n";
当执行完 { } 后，sp的生命周期已经结束，已经调用析构函数释放make_shared指针内存(sp)，gw已经没有在托管任何指针了，调用expired()函数返回true，所以打印"gw is expired\n";

### 2.2.3 lock()

可以通过调用lock()成员函数，获取监视的`shared_ptr`：

- 使用`lock`将资源锁住，`lock`会将`weap_ptr`转为`shared_ptr`，即使`weap_ptr`指向的`shared_ptr`资源被释放也不影响使用，当对象已经释放时会返回一个空`shared_ptr`

- 如果要访问`weap_ptr`指向的数据，必须使用`lock`将`weap_ptr`转为`shared_ptr`才能访问到。

- 在多线程中，要防止一个线程在使用智能指针，而另一个线程删除指针指针问题，可以使用`weak_ptr`的`lock()`方法。

```cpp
auto sp = std::make_shared<int>(42); // 创建一个共享指针;
std::weak_ptr<int> wp(sp);

shared_ptr<int> p;
if (!wp.expired())
{
    p = wp.lock();  // 如果要取到weak_ptr中的，需要先使用lock将weak_ptr转为shard_ptr才能取值；
    sp.reset();
    cout << *p << endl;              // 42
    cout << p.use_count() << endl;   // 1
    cout << sp.use_count() << endl;  // 0
}
```

# 3. enable_from_this_shared

在一个类的成员函数中，我们不能直接将`this`指针作为`shared_ptr`返回，而需要通过派生`std::enable_shared_from_this`类，通过其方法`shared_from_this`来返回指针。**原因**是`std::enable_shared_from_this`类中有一个`weak_ptr`，这个`weak_ptr`用来观察`this`智能指针，调用`shared_from_this()`方法其实是调用内部这个`weak_ptr`的`lock()`方法，将所观察的`shared_ptr`返回。

```cpp
class MyClass : public std::enable_shared_from_this<MyClass>
{
public:
	shared_ptr<MyClass> GetSelf() {
		//return shared_ptr<MyClass>(this);  直接返回this的共享智能指针，如果直接返回MyClass会被析构两次
		return shared_from_this();
	}
	MyClass() {
		cout << "MyClass()" << endl;
	};
	~MyClass() {
		cout << "~MyClass()" << endl;
	};
};

int main()
{
	shared_ptr<MyClass> sp1(new MyClass);
    cout << sp1.use_count()<<endl;
	shared_ptr<MyClass> sp2 = sp1->GetSelf();
    cout << sp1.use_count()<<endl;
    cout << sp2.use_count()<<endl;
}
```

- 在外面创建`MyClass`对象的智能指针和通过对象返回`this`的智能指针都是安全的，因为`shared_from_this()`是`std::enable_shared_from_this＜MyClass>`内部的`weak_ptr`调用`lock()`方法之后返回的智能指针。在离开作用域之后，`sp1`和`sp2`会自动析构，其引用计数减为0，`MyClass`对象会被析构，不会出现`MyClass`对象被析构两次的问题。
- 需要注意的是，获取自身智能指针的函数仅在`shared_ptr`的构造函数被调用之后才能使用，因为`enable_shared_from_this`内部的`weak_ptr`只有通过`shared_ptr`才能构造。

上述代码的输出为：

```
MyClass()
1
2
2
~MyClass()
```

很明显，`sp1`和`sp2`共用同一个引用计数，共享资源，但确实两个`shared_ptr`对象（可以修改指向而不影响其他`shared_ptr`对象）。

> 但注意，你不能直接将`this`指针作为`shared_ptr`返回回来！！！

我在前面说过，不能将智能指针通过`get()`函数返回的裸指针用于初始化或`reset`另一个指针。通过这种方法初始化的智能指针，其实和原本在类内部构造的智能指针是两个独立的对象，它们不共享引用计数，仅仅只是管理的资源相同。如果多次析构，会造成同一个资源被重复析构两次的问题。

所以，不要将`this`指针作为`shared_ptr`返回回来，因为`this`指针本质上是一个裸指针，因此，可能会导致重复析构

```cpp
class MyClass
{
public:
	shared_ptr<MyClass> GetSelf() {
		return shared_ptr<MyClass>(this);//不要这样做
	}
	MyClass() {
		cout << "MyClass()" << endl;
	};
	~MyClass() {
		cout << "~MyClass()" << endl;
	};
};

int main()
{
	// sp1与sp2都会调用new MyClass的析构函数，一个对象析构两次
	shared_ptr<MyClass> sp1(new MyClass);
	shared_ptr<MyClass> sp2 = sp1->GetSelf();
	return 0;
}
```

在这个例子中，由于用同一个指针（this)构造了两个智能指针sp1和sp2，而他们之间是没有任何关系的，在离开作用域之后this将会被构造的两个智能指针各自析构，导致重复析构的错误。

# 4. unqiue_ptr

`unique_ptr`和`shared_ptr`不同，`unique_ptr`不允许所指向的内容被其他指针共享，所以`unique_ptr`不允许拷贝构造和赋值。

```cpp
void use_uniqueptr()
{
    //指向double类型的unique指针
    unique_ptr<double> udptr;
    //一个指向int类型的unique指针
    unique_ptr<int> uiptr(new int(42));
    // unique不支持copy
    // unique_ptr<int> uiptr2(uiptr);
    // unique不支持赋值
    // unique_ptr<int> uiptr3 = uiptr;
}
```

尽管`unqiue_ptr`不能拷贝或赋值，但可以通过调用`release()`、`reset()`或`移动语义`将指针的所有权从一个（非const）unique_ptr转移给另一个`unique`：

| 特性         | `reset()`                        | `release()`                                            |
| ------------ | -------------------------------- | ------------------------------------------------------ |
| **适用范围** | `shared_ptr` 和 `unique_ptr`     | 仅适用于 `unique_ptr`                                  |
| **作用**     | 销毁当前管理的对象，或接管新对象 | 放弃所有权，返回原始指针，不销毁对象                   |
| **后果**     | 对象的生命周期由智能指针管理     | 用户需手动管理返回的原始指针，若未释放可能导致内存泄漏 |
| **返回值**   | 无                               | 返回被管理的原始指针                                   |

**a. release()**

- `release()` 会释放 `unique_ptr` 的所有权，**返回原始指针**，同时将当前 `unique_ptr` 置为空。
- 释放后，需要手动管理返回的裸指针（可以将其转移到另一个 `unique_ptr` 中）。
- 使用 `release()` 后，原来的 `unique_ptr` 不再管理资源，必须确保资源由新的管理者接管，否则会导致内存泄漏。
- `release()` 是 `unique_ptr` **独有**的函数，其他智能指针均没有 。

**b. reset()**

- `reset()` 会释放当前 `unique_ptr` 所管理的资源，并接管一个新的指针。
- 可以通过 `reset()` 将一个裸指针直接交给新的 `unique_ptr`。
- 调用 `reset()` 后，原来的 `unique_ptr` 被释放，接管新资源。

**c. std::move**

- `std::unique_ptr` 支持**移动构造**和**移动赋值**，可以安全地将所有权从一个 `unique_ptr` 转移到另一个。
- 转移后，原来的 `unique_ptr` 不再拥有资源（变为 `nullptr`）。

```cpp
void use_uniqueptr()
{
    //定义一个upstr
    unique_ptr<string> upstr(new string("hello zack"));
    std::cout << "upstr: " << *upstr << "\n";
    // upstr.release()返回其内置指针，并将upstr置空
    // 用upstr返回的内置指针初始化了upstr2
    unique_ptr<string> upstr2(upstr.release());
    std::cout << "upstr2: " << *upstr2 << "\n";
    unique_ptr<string> upstr3(new string("hello world"));
    std::cout << "upstr3: " << *upstr3 << "\n";
    //将upstr3的内置指针转移给upstr2
    // upstr2放弃原来的内置指针，指向upstr3返回的内置指针。
    upstr2.reset(upstr3.release());
    std::cout << "upstr2: " << *upstr2 << "\n";
    // 通过移动语义将upstr2的所有权转移给upstr
    upstr = std::move(upstr2);
    std::cout << "upstr: " << *upstr << "\n";
    // 通过移动语义将upstr的所有权转移给upstr4
    unique_ptr<string> upstr4(std::move(upstr));
    std::cout << "upstr4: " << *upstr4 << "\n";
}
```

输出为：

```
upstr: hello zack
upstr2: hello zack
upstr3: hello world
upstr2: hello world
upstr: hello world
upstr4: hello world
```

不能拷贝`unique_ptr`的规则有一个例外：**我们可以“拷贝“或”赋值”一个将要被销毁的（非const）`unique_ptr`**。最常见的例子是从函数返回一个`unique_ptr`：

```cpp
std::unique_ptr<int> createUniquePtr() {
    auto ptr = std::make_unique<int>(42); // 创建局部 unique_ptr
    return ptr; // 返回时自动调用移动构造
}

int main() {
    std::unique_ptr<int> myPtr = createUniquePtr(); // 接收函数返回值
    std::cout << "Value: " << *myPtr << std::endl;
    return 0;
}
```

虽然这个过程好像确实是在调用拷贝构造函数创建了一个副本，但`std::unique_ptr` 的拷贝构造和拷贝赋值是被**明确删除**的（`= delete`），无法直接复制。

因为`std::unique_ptr`支持移动构造和移动赋值，所以当函数返回一个局部的 `std::unique_ptr` 时，C++ 会隐式应用移动构造，并不会真的尝试拷贝它。

我在[文章](https://zhuanlan.zhihu.com/p/892218451)中简单说过，编译器会根据传入参数自动选择构造函数：

- 如果对象的拷贝构造函数可用，但移动构造函数也存在，`push_back` 会选择合适的构造函数：
  - 如果传递的是一个左值，会调用拷贝构造函数。
  - 如果传递的是一个右值，会调用移动构造函数。
- 如果对象的拷贝构造函数不可用，但移动构造函数存在，`push_back` 的选择：
  - 如果传递的是一个左值，会直接报错，编译器无法将左值隐式转换为右值从而调用移动。
  - 如果传递的是一个右值，会调用移动构造函数。

在这里，函数返回一个临时的`unique_ptr`变量，是右值。而`unique_ptr`禁止拷贝或复制，但允许移动拷贝或移动赋值，所以编译器会自动调用`unique_ptr`的移动构造函数。因此，从函数返回时并不会违反 `std::unique_ptr` 的独占所有权规则。

