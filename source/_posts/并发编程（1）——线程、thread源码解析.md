---
title: 并发编程（1）——线程、thread源码解析
date: 2024-11-03 12:48:34
categories:
- C++
- 并发编程
tags: 
- 线程
- join、detach
- RAII
- thread参数传递及调用原理
typora-root-url: ./..
---

# 一、day1

今天学习的内容包括：

1）线程如何发起（普通函数、仿函数、labda函数，类的成员函数，以及仿函数可能会遇到“最烦恼的解析”问题）；⭐⭐⭐⭐⭐

2）线程的等待、detach，以及子线程使用主线程资源可能会遇到的风险；

3）线程的异常处理（如何通过RAII技术，也就是线程守卫实现）；

4）线程中隐式转换同样会造成风险，类型和子线程使用主线程资源可能会遇到的风险相似；

5）线程中即使传入实参的是左值，形参类型是引用，但仍然会将其拷贝而不是使用传入值；

6）`std::ref`源码解析

6）C++中thread参数传递和调用原理（解释了为什么thread传入的参数若不经过std::ref包装，均会作为右值被保存使用）⭐⭐⭐⭐⭐

参考：

[C++ 并发编程(1) 线程基础，为什么线程参数默认传参方式是值拷贝？_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1FP411x73X/?vd_source=cb95e3058c2624d2641da6f4eeb7e3a1)

[恋恋风辰官方博客](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/2TayNx5QxbGTaWW5s48vMjtuvCB)

[命名空间 - this_thread | 爱编程的大丙](https://subingwen.cn/cpp/this_thread/#1-get-id)

[ModernCpp-ConcurrentProgramming-Tutorial/md/详细分析/01thread的构造与源码解析.md at main · Mq-b/ModernCpp-ConcurrentProgramming-Tutorial · GitHub](https://github.com/Mq-b/ModernCpp-ConcurrentProgramming-Tutorial/blob/main/md/详细分析/01thread的构造与源码解析.md)

[【C++并发编程实战】 thread 源码实现 && 向线程函数传递参数 - 知乎](https://zhuanlan.zhihu.com/p/717266327)

------

# 1. 线程的建立

**进程**：一个进程是一个正在运行的程序的实例，拥有自己的地址空间、代码、数据和系统资源。一个进程可以包含一个或多个线程。进程也是程序的⼀次执⾏过程，是系统运⾏程序的基本单位；系统运⾏⼀个程序即是 ⼀个进程从创建、运⾏到消亡的过程。

**线程**：线程是进程中的⼀个执⾏单元，负责当前进程中程序的执⾏，⼀个进程中⾄少有⼀个线程。⼀个进程中是可以有多个线程的，这个应⽤程序也可以称之为多线程程序。

简⽽⾔之：***⼀个程序运⾏后⾄少有⼀个进程，⼀个进程中可以包含多个线程***

## 1.1 线程如何发起

C++线程自C++11及以后的版本中被统一，在包含头文件`<thread>`之后，通过使用 `std::thread` 定义线程对象（该对象可以通过构造函数接受一个可调用对象，比如**函数、仿函数、lambda函数等**），该对象可以启动线程执行回调逻辑（执行传入的可调用对象），**线程的发起是在创建对象的同时被启动。**

**std::thread()** 的原型为：

```cpp
template <class F, class... Args>
explicit thread(F&& f, Args&&... args);
```

- `F`：可调用对象的类型，可以是函数指针、函数对象、lambda 表达式等。
- `Args`：可变参数模板，表示传递给可调用对象的参数，比如参数列表中的形参。

###   1.1.1 普通函数

可以使用普通函数作为可调用对象传入给线程对象。

```cpp
#include <thread>

void thread_hello(std::string str) {
    std::cout << str << std::endl;
}

std::string str = "hello world!";
std::thread t(thread_hello, str); // 发起线程
```

### 1.1.2 仿函数

可以使用仿函数作为可调用对象传入给线程对象。

```cpp
class background_task {
public:
    void operator() {
        std::cout << "str is " << std::endl;
    }
};
```

> **注意，传入仿函数作为可调用对象时不能加 '()'**

```cpp
std::thread t2(background_task());
t2.join();
```

这样是错误的：

1）`background_task()` 会被解释为一个函数声明，而不是对象的创建。因为编译器会将`t2`当成一个函数对象, 返回一个`std::thread`类型的值，函数的参数为一个**函数指针**，该函数指针返回类型为`background_task`, 没有参数。导致编译器将原本应该是对象的构造解析为函数的声明。

可以理解为：

```cpp
"std::thread (*)(background_task (*)())"
```

这是因为C++中***“最烦恼的解析”***造成的，比如：

```cpp
class MyClass {
public:
    MyClass() {}
};

// 返回对象是MyClass ，函数名为obj，无参数
MyClass obj(); // 这不是对象的声明，而是一个函数的声明
```

在上面的代码中，`MyClass obj()`; 被编译器解析为一个返回 `MyClass` 类型的函数 `obj`，而不是一个 `MyClass` 类型的对象。这种情况被称为“最烦恼的解析”，导致编译器将原本应该是对象的构造解析为函数的声明。原因：

- **语法规则**：C++ 的语法规则允许使用类名后跟括号的形式来声明函数（仿函数）。如果没有其他上下文，编译器会选择这种解析方式。
- **上下文歧义**：在某些情况下，编译器无法明确判断你是想要创建一个对象还是声明一个函数，因此选择最符合语法的解析方式。

为了避免最烦恼的解析，可以使用如下方法：

```cpp
// 1. 使用花括号
MyClass obj{}; // 这明确表示对象的构造
// 2. 使用额外的括号
MyClass obj((1)); // 使用额外的括号，避免解析为函数声明
// 3. 使用指针
MyClass* obj = new MyClass(); // 使用指针来创建对象
```

所以我们如果使用仿函数作为可调用对象传入时，可以这样做：

```cpp
class background_task {
public:
    void operator() {
        std::cout << "str is " << std::endl;
    }
};

// 1.创建对象
background_task task; // 创建对象
std::thread t2(task); // 传入对象和参数
// 2. 使用花括号
std::thread t2{ background_task() }; // 使用花括号初始化对象
// 3. 使用指针
background_task* task = new background_task(); // 创建对象并使用指针
std::thread t2(*task);
// 4. 使用临时对象
std::thread t2((background_task())); // 使用额外的括号
```

但如果仿函数中有参数，那么就不会造成"最烦恼的解析”，因为上下文有解释，我是要调用仿函数（因为传入的参数和仿函数对应，构造函数与其不对应），比如：

```cpp
class background_task {
public:
    void operator()(std::string str) {
        std::cout << "str is " << str << std::endl;
    }
};

std::string str = "hello world!";
// 不会发生报错
std::thread t2(background_task(), str);
t2.join();
```

> ***"最烦恼的解析”一般*在无传入参数的情况下发生。**

2）这样并没有创建一个可调用对象。因为 `background_task()` 并没有创建对象（这样是在**调用仿函数，而不是把仿函数作为对象**），t2 线程实际上没有被正确初始化。

### 1.1.3 lambda函数

可以使用lambda函数作为可调用对象传入给线程对象。

```cpp
std::thread t4([](std::string  str) {
    std::cout << "str is " << str << std::endl;
},  hellostr);
t4.join();
```

### 1.1.4 类的成员函数

```cpp
class X
{
public:
    void do_lengthy_work() {
        std::cout << "do_lengthy_work " << std::endl;
    }
};
void bind_class_oops() {
    X my_x;
    // 还得传入类的指针，因为类的成员函数会调用类的成员变量
    std::thread t(&X::do_lengthy_work, &my_x);
    t.join();
}
```

注意：如果 thread 绑定的回调函数是**普通函数**，可以在函数前加`&`或者不加`&`，因为编译器默认将普通函数名作为函数地址，如下两种写法都正确。

```cpp
void thead_work1(std::string str) {
    std::cout << "str is " << str << std::endl;
}
std::string hellostr = "hello world!";
//两种方式都正确
std::thread t1(thead_work1, hellostr);
std::thread t2(&thead_work1, hellostr);
```

> **但是如果是绑定类的成员函数，必须添加& （还得传入类的指针，因为类的成员函数会调用类的成员变量）**

### 1.1.5 move

若传递给线程的参数是独占的，也就是不支持拷贝赋值和构造，但我们可以通过 `std::move` 的方式将参数的所有权转移给线程，如下

```cpp
void deal_unique(std::unique_ptr<int> p) {
    std::cout << "unique ptr data is " << *p << std::endl;
    (*p)++;
    std::cout << "after unique ptr data is " << *p << std::endl;
}
void move_oops() {
    // p是独占有权的，不能被复制、赋值，但可以转移所有权
    auto p = std::make_unique<int>(100);
    std::thread  t(deal_unique, std::move(p));
    t.join();
    //不能再使用p了，p已经被move废弃
   // std::cout << "after unique ptr data is " << *p << std::endl;
}
```

> **注意，若参数的占有权被转移，那么该参数就不能管理之前保存的值，丧失了对该值的占有权**

## 1.2 join()

> 虽然使用 `std::thread` 创建的线程在结束时会自动释放其资源，但在主线程（或创建线程的线程）中仍需要等待其子线程结束。**我们需要在主线程中显式调用`join()`函数等待子线程的结束，**子线程结束后主线程才会继续运行。

原因如下：

1. 如果创建的子线程在其执行过程中没有被主线程等待，那么当主线程结束或被销毁时，操作系统将会终止这个子线程，这可能导致子线程的资源（如内存、文件句柄等）不会被释放，产生**资源泄漏**。
2. 如果不调用 `join()`，主线程在没有等待子线程结束的情况下继续执行，可能会导致程序在子线程完成之前就结束，从而**未能正确处理子线程的结果**（子线程的结果可能不会被主线程处理）。
3. 如果主线程需要依赖于子线程完成某些任务（例如数据处理或文件写入），需要通过 `join()` 确保子线程在主线程继续执行之前完成，可以避免因**数据未更新而导致的不一致性**。

> 线程的回收通过线程的析构函数来完成，即执行`terminate`操作。

```cpp
// 1.发起线程
std::string str = "hello world!";
std::thread t1(thread_hello, str); 
// 让主线程暂停执行 1 秒钟，确保子线程能够执行完毕
std::this_thread::sleep_for(std::chrono::seconds(1));
// 2.主线程等待子线程结束
t1.join();
```

## 1.3 detach()

可以使用`detach`允许子线程采用分离的方式在后台**独自运行**，不受主线程影响。主线程和子线程执行各自的任务，使用各自的资源。

> 注意：当一个线程被分离后，**主线程将无法直接管理它**，也无法使用`join()`等待被分离的线程结束。处理日志记录或监控任务这些线程一般会让其在后台持续运行，使用`detach`。

```cpp
struct func {
    int& _i;
    func(int & i): _i(i){}
    void operator()() {
        for (int i = 0; i < 3; i++) {
            _i = i;
            std::cout << "_i is " << _i << std::endl;
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }
    }
};
void oops() {
        int some_local_state = 0;
        func myfunc(some_local_state);
        std::thread functhread(myfunc);
        //隐患，访问局部变量，局部变量可能会随着}结束而回收或随着主线程退出而回收
        functhread.detach();    
}
// detach 注意事项
oops();
//防止主线程退出过快，需要停顿一下，让子线程跑起来detach
std::this_thread::sleep_for(std::chrono::seconds(1));
```

`detach`使用时有一些**风险**，比如上述代码。

当主线程调用`oops`时，会创建一个线程执行`myfunc`的重载`()`运算符，然后将主线程将`oops`创建的一个线程分离。但注意，当`oops`执行到 `'}'` 时，局部变量 `some_local_state` 会被释放，但**引用（这里是引用传递而不是按值传递，按值传递不会引起该错误，因为线程中已经有一个自己的拷贝副本了**）该局部资源的子线程 `functhread` 却仍然在后台运行，容易发生错误。

我们可以采取一些措施解决该问题：

1. 通过**智能指针传递局部变量**，因为引用计数会随着赋值增加，可保证局部变量在使用期间不被释放，避免悬空指针的问题（网络编程中学习的伪闭包原理）。
2. **按值传递**，将局部变量的值作为参数传递而不是按引用传递，这么做需要局部变量有拷贝复制的功能，而且拷贝耗费空间和效率。
3. 使用 `join()` 确保局部变量的生命周期，保证局部变量被释放前线程已经运行结束，但是可能会影响运行逻辑。

## 1.4 get_id()

应用程序启动之后默认只有一个线程，这个线程一般称之为**主线程或父线程**，通过线程类创建出的线程一般称之为**子线程**，每个被创建出的线程实例都对应一个**线程ID**，这个ID是**唯一**的，可以通过这个ID来区分和识别各个已经存在的线程实例，这个获取线程ID的函数叫做`get_id()`，函数原型如下：

```cpp
std::thread::id get_id() const noexcept;
```

线程 id 是 `thread` 类中唯一私有成员`_ Thr` 的公有成员`_Thrd_id_t _Id`;

使用方法如下：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std;

void func1(int num, string str){return;}

void func2(){return;}

int main()
{
    cout << "主线程的线程ID: " << this_thread::get_id() << endl;
    thread t1(func1, 1, "a");
    thread t2(func2);
    cout << "线程t1 的线程ID: " << t1.get_id() << endl;
    cout << "线程t2 的线程ID: " << t2.get_id() << endl;
}
```

## 1.5 异常处理&joinable()

当启动一个子线程时，子线程和主线程是并发运行的。如果主线程由于某种原因崩溃（例如未捕获的异常），则整个进程将会终止（主线程崩溃或者结束时，主进程会回收所有线程的资源），这意味着所有正在运行的线程，包括子线程（不管有没有被detach）都会被强制结束，导致子线程未完成的操作（如数据库写入）被中断。这可能会导致子线程待写入的信息丢失。

为了防止主线程崩溃导致子线程异常退出，可以在主线程中捕获可能抛出的异常，在捕获到异常后，可以选择在主线程中等待所有子线程完成。这样可以确保即使主线程遇到问题，子线程仍然能够完成其操作，并安全地结束。

```cpp
struct func {
    int& _i;
    func(int & i): _i(i){}
    void operator()() {
        for (int i = 0; i < 3; i++) {
            _i = i;
            std::cout << "_i is " << _i << std::endl;
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }
    }
};

void catch_exception() {
    int some_local_state = 0;
    func myfunc(some_local_state);
    std::thread  functhread{ myfunc };
    try {
        // 可能引发崩溃的程序
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }
    catch (std::exception& e) {
        functhread.join();
        throw;
    }
    functhread.join();
}
```

但是这样太过于繁琐，我们还得捕获异常后将对应的线程进行`join`，但如果我们有多个线程和多个异常呢？难道还要一个个的组合，写异常处理？之前学习的通过协程实现异步服务器中设计了一个逻辑层，其中逻辑层中对逻辑队列消息的处理就是对上面代码的简化。逻辑层首先会创建一个线程处理逻辑队列的消息，并一直while循环（条件变量挂起，防止队列为空时仍然循环浪费资源），该线程仅有在逻辑层的析构函数被调用时才会结束。在逻辑层的析构函数中，首先将一个标志位置为true，表示逻辑队列的消息处理线程可以退出；然后使用条件变量的`notify.one()`函数唤醒该线程，如果队列有数据那么将所有数据都处理完；最后，析构函数会等待该线程的所有消息处理完才会析构完成。也就是**RAII技术**，如下：

```cpp
LogicSystem::~LogicSystem() {
	std::cout << "逻辑层成功析构" << std::endl;
	_b_stop = true;
	_consume.notify_one(); // 唤醒逻辑处理线程
	_worker_thread.join(); // 等待逻辑消息线程处理完成
}
```

详细内容可参考：

[网络编程（19）——C++使用asio协程实现并发服务器 - 知乎](https://zhuanlan.zhihu.com/p/957175334)

那么，我们也可以使用相同的思维方法来对上面这段代码进行简化处理，即**线程守卫：**

```cpp
class thread_guard {
private:
    std::thread& _t;
public:
    explicit thread_guard(std::thread& t):_t(t){}
    ~thread_guard() {
        //join只能调用一次
        if (_t.joinable()) {
            _t.join();
        }
    }
    thread_guard(thread_guard const&) = delete;
    thread_guard& operator=(thread_guard const&) = delete;
};
```

- `joinable()` 是 `std::thread` 的一个成员函数，返回一个布尔值，指示线程是否可连接（即是否已创建且尚未调用 `join()` 或 `detach()`），如果`_t`是一个有效的线程对象且没有调用`join()` 或 `detach()`，那么调用`join`等待该线程结束。

我们可以将需要保护的线程（可能发生异常错误的线程）传递给thread_guard创建一个实例，如果主线程异常发生，保护子线程实例的析构函数会自动调用，确保主线程发生异常时，子线程也能被正确管理，防止资源泄漏

举例：

```cpp
void auto_guard() {
    int some_local_state = 0;
    func my_func(some_local_state);
    std::thread  t(my_func);
    thread_guard g(t);
    //主线程可能会造成异常的程序代码
    std::cout << "auto guard finished " << std::endl;
}
auto_guard();
```

如上例所示，通过`thread_guard` 构造一个新实例来保护线程t，那么即使在 `auto_guard` 函数中发生异常，`thread_guard` 也会确保线程t被正确管理，避免资源泄漏。

## 1.6 hardware_concurrency()

`thread `线程类还提供了一个**静态方法**，用于**获取当前计算机的CPU核心数**，我们可以根据这个结果在线程池中创建出数量相等的线程，每个线程独自占有一个CPU核心，这些线程就不用分时复用CPU时间片，此时程序的并发效率是最高的。函数原型为：

```cpp
static unsigned hardware_concurrency() noexcept;
```

使用方法：

```cpp
#include <iostream>
#include <thread>

int main()
{
    int num = std::thread::hardware_concurrency();
    std::cout << "CPU number: " << num << std::endl;
}
```

## 1.7 慎重使用隐式转换

C++中经常可以看到一些隐式转换，比如short转换为int、char*转换为string等，但这些隐式转换在线程的调用上可能会造成崩溃问题。

```cpp
void print_str(int i, std::string const s) {
    std::cout <<"i is "<<i<<" str is " << s << std::endl; 
}

void danger_oops(int som_param) {
    char buffer[1024];
    sprintf(buffer, "%i", som_param);
    //在线程内部将char const* 转化为std::string
    //指向字符的常量指针  char * const  指针的地址是常量，即指针本身不能被修改（不能让它指向其他地方），但可以通过该指针修改它指向的字符数据。
    //指向常量字符的指针  const char * 指向的内容不能变
    // const char* 和 char const* 都表示的常量指针，即指针所指向的字符数据是常量，不能通过该指针修改。
    std::thread t(print_str, 3, buffer);
    t.detach();
    std::cout << "danger oops finished " << std::endl;
}
```

`buffer` 是一个局部变量，在线程 t 启动后，`danger_oops` 函数会继续执行并最终结束。当 `danger_oops` 函数返回后，`buffer` 会被销毁，导致线程在执行 `print_str` 时尝试访问一个无效的内存地址。因为当定义一个线程变量`thread t` 时，传递给线程 `t` 的参数`buffer`会被保存到`thread`的成员变量中。而在线程对象`t`内部启动并运行线程时，参数才会被传递给调用函数`print_str`，但此时`danger_oops` 函数可能已经返回，局部变量 `buffer` 被销毁，导致线程在执行 `print_str` 时尝试访问一个无效的内存地址**（传入的是char，即字符串首地址，如果传入的不是地址，也不是按引用传递，而是按值传递，那么子线程会创建一个拷贝副本，即使局部变量被释放，子线程仍然可以继续工作）**，虽然我们确实是按值传递，接收的类型是`string`，而不是`string&`和`string`，但因为传入的是 `char` 恰好可以通过**隐式**转换变为`string`，所以此时相当于传入的是`string`，而形参类型也是`string`。这部分内容可以参考上面1.3将的`detach`。

所以我们只需**将隐式转换变为显示转换**即可，将`char*`字符串显示转换为`string`，那么传入的就是一个`string`对象，此时，子线程会创建一个拷贝对象，即使`danger_oops`函数返回，子线程也不会指向空的对象。

```cpp
void safe_oops(int some_param) {
    char buffer[1024];
    sprintf(buffer, "%i", some_param);
    std::thread t(print_str, 3, std::string(buffer));
    t.detach();
}
```

## 1.8 如何在线程中使用引用

> 在创建线程时，使用 `std::thread` 来传递参数时，***参数是以拷贝的方式传递的***。即使你传入的是一个左值（如一个变量），`std::thread` 会在内部创建该参数的拷贝。但是在main函数中，如果传入的实参是左值，形参类型是引用，那么函数**不会创建副本**，而是直接对传入的值进行修改。

**主线程**：当在主线程中调用函数时，参数是按值传递还是按引用传递取决于函数的参数声明。如果函数的参数是引用类型（如 `int&`），那么传递的是对原始变量的引用，可以直接修改这个变量。

```cpp
void change_param(int& param) {
    param++; // 修改引用的值
}

int main() {
    int value = 5;
    change_param(value); // 这里传递的是 value 的引用
    std::cout << value; // 输出 6
}
```

**子线程**：当在子线程中调用函数时，**即使**参数在函数定义中是引用类型（如 `int&`），如果在 `std::thread` 创建线程时直接传递一个变量（如 `some_param`），这个变量仍会被复制到线程中，子线程内部的修改不会影响主线程中的原始变量。

```cpp
void change_param(int& param) {
    param++; // 修改引用的值
}

void ref_oops() {
    int some_param = 5;
    std::thread t2(change_param, some_param); // 传递的是 some_param 的拷贝
    t2.join();
    // some_param 的值仍然是 5
}
```

而且，上面这段代码和下面这段代码相同，都会报错：

```cpp
void change_param(int& param) {
    param++;
}
void ref_oops(int some_param) {
    std::cout << "before change , param is " << some_param << std::endl;
    //需使用引用显示转换
    std::thread  t2(change_param, some_param);
    t2.join();
    std::cout << "after change , param is " << some_param << std::endl;
}
```

即使函数 `change_param` 的参数为`int&`类型，我们传递给`t2`的构造函数为`some_param`，也不会达到在`change_param`函数内部修改关联到外部`some_param`的效果。

**因为`some_param`是外部传给函数`ref_oops`实参的拷贝**（左值，**这里的拷贝不是右值，它仍然可以取地址，右值一般只会在字面常量、表达式返回值、函数非左值引用返回值中出现**），左值传递给线程`thread`的构造函数之后会被保存为**右值引用**（`thread`内部通过`move`，传入左值会被保存为右值，如果传入右值类型不会变化），右值如果传给调用对象`change_param`就会报错。因为`change_param`中的参数是左值引用，左值引用不能接收右值。有两种方法可以修正：

**方法一**：修改 `change_param` 的参数为 `const` 引用类型

```cpp
void change_param(const int& param) {
}

void ref_oops(int some_param) { // 将 some_param 改为引用类型
    std::cout << "before change, param is " << some_param << std::endl;
    std::thread t2(change_param, some_param); // 使用 std::ref 显式传递引用
    t2.join();
    std::cout << "after change, param is " << some_param << std::endl;
}
```

但**缺点**是，不能对`some_param`进行修改了，因为`const int&` 既可以用于传递左值引用，也可以用于传递右值，唯独不能修改传递过来的值。

**方法二**：传递 `std::ref`

```cpp
void change_param(int& param) {
    param++;
}
void ref_oops(int some_param) {
    std::cout << "before change, param is " << some_param << std::endl;
    std::thread t2(change_param, std::ref(some_param)); // 使用 std::ref 显式传递引用
    t2.join();
    std::cout << "after change, param is " << some_param << std::endl;
}
```

*第一种方法是直接修改可调用对象参数列表的类型，使其可以接受右值类型。*

*第二种方法其实还是将左值参数通过`ref`进行包装，使得`thread`内部不会将其引用类型`delay`，这样传递给调用对象的参数就仍是左值引用，而不是右值引用。*

> **那么如果我传递的是一个左值，而不是实参的拷贝呢，会不会还有问题？**

```cpp
void change_param(int& param) {
    param++;
}
void ref_oops(） {
    int some_param = 5;
    std::cout << "before change , param is " << some_param << std::endl;
    //需使用引用显示转换
    std::thread  t2(change_param, some_param);
    t2.join();
    std::cout << "after change , param is " << some_param << std::endl;
}
```

该段函数中，我们传给线程调用对象`change_param`的参数是一个左值，而`change_param`形参的类型是引用，那么这样按理说应该是正确的，即线程内部对`some_param`的处理会影响到外部的`some_param`。**但是**，要注意**线程无视引用，即使你传入的是左值，形参是引用，参数同样会被拷贝，除非你按引用传入（`ref`），或者传入的实参本来就是个引用。**

```cpp
void ref_oops(） {
    int some_param = 5;
    std::cout << "before change , param is " << some_param << std::endl;
    //需使用引用显示转换
    std::thread  t2(change_param, std::ref(some_param));
    t2.join();
    std::cout << "after change , param is " << some_param << std::endl;
}
```

线程调用中，左值同样要加`ref`显式变为引用。可以参考1.8刚开始。

# 2. std::this_thread

我们可以调用命名空间 `std::this_thread` 中的四个公共成员函数对我们创建的线程进行相关的操作。

## 2.1 get_id()

和 `thread` 类的公共成员函数 `get_id()` 相同，用于获取当前线程的 **ID**，函数原型同样为：

```cpp
thread::id get_id() noexcept;
```

使用方法：

```cpp
#include <iostream>
#include <thread>

void func()
{
    std::cout << "子线程: " << std::this_thread::get_id() << std::endl;
}

int main()
{
    std::cout << "主线程: " << std::this_thread::get_id() << std::endl;
    std::thread t(func);
    t.join();
}
```

我们既即可通过调用 `t.get_id()` 在主线程中获取子线程 `t` 的线程**ID**，也可以在子线程任务中调用 `std::this_thread::get_id()` 获取子线程的线程**ID**。只不过前者返回给主线程中使用，后者返回给子线程使用。

## 2.2 sleep_for()

线程和进程的执行有很多相似之处，在计算机中启动的多个线程都需要占用CPU资源，但是CPU的个数是有限的并且每个CPU在同一时间点不能同时处理多个任务。为了能够实现**并发**处理，多个线程都是**分时复用CPU时间片**，快速的交替处理各个线程中的任务。因此多个线程之间需要**争抢CPU时间片**，抢到了就执行，抢不到则无法执行（因为默认所有的线程优先级都相同，内核也会从中调度，不会出现某个线程永远抢不到CPU时间片的情况）。

命名空间 `this_thread` 中提供了一个休眠函数 `sleep_for()`，调用这个函数的线程会马上**从运行态变成阻塞态并在这种状态下休眠一定的时长**，因为阻塞态的线程已经让出了CPU资源，代码也不会被执行，所以线程休眠过程中对CPU来说没有任何负担。这个函数是函数原型如下，参数需要指定一个休眠时长，是一个时间段：

```cpp
template <class Rep, class Period>
  void sleep_for (const chrono::duration<Rep,Period>& rel_time);
```

示例程序如下：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std;

void func()
{
    for (int i = 0; i < 10; ++i)
    {
        this_thread::sleep_for(chrono::seconds(1));
        cout << "子线程: " << this_thread::get_id() << ", i = " << i << endl;
    }
}

int main()
{
    thread t(func);
    t.join();
}
```

在`func()`函数中使用了`this_thread::sleep_for(chrono::seconds(1));`之后，每循环一次程序都会阻塞**1**秒钟，也就是说每隔**1**秒才会进行一次输出。需要注意的是：**程序休眠完成之后，会从阻塞态重新变成就绪态，就绪态的线程需要再次争抢CPU时间片，抢到之后才会变成运行态，这时候程序才会继续向下运行。**

## 2.3 sleep_until()

命名空间`this_thread`中提供了另一个休眠函数`sleep_until()`，和`sleep_for()`不同的是它的参数类型不一样

- `sleep_until()`：指定线程阻塞到某一个指定的时间点`time_point`类型，之后解除阻塞
- `sleep_for()`：指定线程阻塞一定的时间长度`duration` 类型，之后解除阻塞

该函数的函数原型如下：

```cpp
template <class Clock, class Duration>
  void sleep_until (const chrono::time_point<Clock,Duration>& abs_time);
```

示例程序如下：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std;

void func()
{
    for (int i = 0; i < 10; ++i)
    {
        // 获取当前系统时间点
        auto now = chrono::system_clock::now();
        // 时间间隔为2s
        chrono::seconds sec(2);
        // 当前时间点之后休眠两秒
        this_thread::sleep_until(now + sec);
    }
}

int main()
{
    thread t(func);
    t.join();
}
```

`sleep_until()`和`sleep_for()`函数的功能是一样的，只不过前者是基于时间点去阻塞线程，后者是基于时间段去阻塞线程，项目开发过程中根据实际情况选择最优的解决方案即可。

## 2.4 yield()

在线程中调用 `yield()` 函数之后，处于运行态的线程会主动让出自己已经抢到的CPU时间片，最终变为就绪态（就绪态的线程需要再次争抢CPU时间片，抢到之后才会变成运行态，这时候程序才会继续向下运行），这样其它的线程就有更大的概率能够抢到CPU时间片了。

> 使用这个函数的时候需要注意一点，线程调用了`yield()`之后会主动放弃CPU资源，但是这个变为就绪态的线程会马上参与到下一轮CPU的抢夺战中，不排除它能继续抢到CPU时间片的情况，这是概率问题。

```cpp
void yield() noexcept;
```

函数对应的示例程序如下：

```cpp
#include <iostream>
#include <thread>
using namespace std;

void func()
{
    for (int i = 0; i < 100000000000; ++i)
    {
        cout << "子线程: " << this_thread::get_id() << ", i = " << i << endl;
        this_thread::yield();
    }
}

int main()
{
    thread t(func);
    thread t1(func);
    t.join();
    t1.join();
}
```

在上面的程序中，执行`func()`中的for循环会占用大量的时间，在极端情况下，如果当前线程占用CPU资源不释放就会导致其他线程中的任务无法被处理，或者该线程每次都能抢到CPU时间片，导致其他线程中的任务没有机会被执行。解决方案就是每执行一次循环，让该线程主动放弃CPU资源，重新和其他线程再次抢夺CPU时间片，如果其他线程抢到了CPU时间片就可以执行相应的任务了。

# 3. thread源码解析

thread参数传递涉及到**引用折叠**问题，即

- 左值引用+左值引用->左值引用
- 左值引用+右值引用->左值引用
- 右值引用+右值引用->右值引用

> **凡是折叠中出现左值引用，优先将其折叠为左值引用**

在类型推断中，如果传入的是一个左值，模板类型会自动将其推断为一个左值引用；而传入右值，模板类型会将其推断为右值：

```cpp
template <class F, class... Args>
auto commit(F&& f, Args&&... args) -> std::future<decltype(f(args...))> {}

int m = 5;
commit([](int& m){}, m); // 1
commit([](int& m){}, 2); // 2
commit([](int& m){}, std::ref(m)); // 3 
commit([](int& m){}, std::move(m)); // 4
```

- 对于1：Args推断m是int&类型，经过折叠后，int& &&->int&，仍然是int&。（注意，不会将其推断为int，虽然m确实是int类型，但是左值的类型在模板参数中会被视为它本身的引用类型）
- 对于2：Args推断m是int类型，经过折叠后，int&&->int&&，是int&&，右值引用。（注意，右值会被推断为int类型而不是int&&）
- 对于3：Args推断m是int&类型，并且经过ref包装后，thread和async内部不会对其使用delay解除cv修饰符和引用。
- 对于4：Args推断m是int&&类型，经过折叠后，int&& &&->int&，仍然是int&&。

------

如果需要向子线程传递参数，直接向`std::thread`的构造函数传递参数即可。比如：

```cpp
void f(int i, std::string const& s);
std::thread t(f, 3, "hello");
```

不过请务必牢记，子线程具有**内部存储空间**，参数先默认地复制到该处，子线程才能直接访问这些参数。这些副本被当作临时变量，以右值的形式传递给子线程上的可调用对象**（也就是如果不显式的将实参以引用ref的方式传入，那么即使子线程的可调用对象的形参是引用类型，可调用对象仍然使用的是传入参数的拷贝，因为子线程首先将传入值复制到子线程的内部存储空间，然后将副本右值的形式传递给子线程上的可调用对象**）。

在上述例子中，即使函数f有引用参数 std::string const& s，参数仍然以复制的方式传递。

请注意，尽管函数f的第二个形参为 std::string类型，但是"hello"仍然以指针`char const *`的形式传入到子线程的内存空间中，当指针被拷贝至子线程的内存以后， 才转换为std::string类型。

## 3.1 数据成员

`std::thread` 只有一个私有数据成员**_Thr**：

```cpp
private:
    _Thrd_t _Thr;
```

`_Thrd_t` 是一个结构体，它有两个数据成员：

```cpp
using _Thrd_id_t = unsigned int;
struct _Thrd_t { // thread identifier for Win32
    void* _Hnd; // Win32 HANDLE
    _Thrd_id_t _Id;
};
```

这个结构体的 `_Hnd` 成员是指向线程的句柄，句柄允许 C++ 程序与底层操作系统线程进行交互，如等待线程结束、获取线程信息等；`_Id` 成员就是保有线程的 ID。

在64 位操作系统，因为**内存对齐**（**内存对齐要求通常是基于最大成员的对齐方式**，这里必须保证结构体的大小是最大成员大小的倍数，这里最大成员是指针8，所以结构体的大小必须是8的倍数），指针 8 ，无符号 int 4，这个结构体 `_Thrd_t` 就是占据 16 个字节（尽管成员总共只占用12字节，但为了使整个结构体的大小为16字节，编译器会在结构体末尾添加4个字节的填充）。也就是说 `sizeof(std::thread)` 的结果应该为 16。

## 3.2 构造函数

### 3.2.1 函数原型

`std::thread`有四个[构造函数](https://link.zhihu.com/?target=https%3A//zh.cppreference.com/w/cpp/thread/thread/thread)，分别是：

**1）默认构造函数**，构造不关联线程的新 std::thread 对象。

```cpp
thread() noexcept : _Thr{} {}
```

[值初始化](https://link.zhihu.com/?target=https%3A//zh.cppreference.com/w/cpp/language/value_initialization%23%3A~%3Atext%3D%E5%87%BD%E6%95%B0%E7%9A%84%E7%B1%BB%EF%BC%89%EF%BC%8C-%2C%E9%82%A3%E4%B9%88%E9%9B%B6%E5%88%9D%E5%A7%8B%E5%8C%96%E5%AF%B9%E8%B1%A1%2C-%EF%BC%8C%E7%84%B6%E5%90%8E%E5%A6%82%E6%9E%9C%E5%AE%83)了数据成员 `_Thr` ，这里的效果相当于给其成员`_Hnd`和`_Id`都进行[零初始化](https://link.zhihu.com/?target=https%3A//zh.cppreference.com/w/cpp/language/zero_initialization)。

这里的默认构造函数不接受任何参数，并且被标记为 **noexcept**，这意味着它保证不抛出异常。它能创建但不立即执行任何线程的thread对象，这样的对象通常称为**空线程对象**。在C++中创建线程时，可以先声明一个空的线程对象，**稍后**再将其与实际的执行函数关联起来。

**2）移动构造函数**，转移线程的所有权，将 _Other 的线程对象 _Thr 的所有权转移到新创建的线程对象中。此调用后 other 失去了其线程的所有权。

```cpp
thread(thread&& _Other) noexcept : _Thr(_STD exchange(_Other._Thr, {})) {}
```

[_STD](https://link.zhihu.com/?target=https%3A//github.com/microsoft/STL/blob/8e2d724cc1072b4052b14d8c5f81a830b8f1d8cb/stl/inc/yvals_core.h%23L1934)是一个宏，展开就是 **::std::**，也就是 ***[::std::exchange](https://link.zhihu.com/?target=https%3A//zh.cppreference.com/w/cpp/utility/exchange)*** ，将 `_Other._Thr` 赋为 `{}`（也就是置空，通常是一个无效的线程状态），返回`_Other._Thr` 的旧值用以初始化当前对象的数据成员 **`_Thr`（转移所有权）**。

`std::exchange`是C++14引入的一个实用函数，它用于交换两个值并返回被交换掉的旧值。

```cpp
#include <iostream>
#include <utility>

int main() {
    int a = 10;
    int b = 20;

    int result = std::exchange(a, b);
    std::cout << "a: " << a << ", result: " << result << std::endl;  // 输出 a: 20, result: 10

    return 0;
}
```

输出结果：

```cpp
a: 20, result: 10
```

a的值和b的值进行交换，所以a的值为10，`std::exchange(a, b);`的返回结果是**左操作数**，即**旧值**。

**3）复制构造函数被定义为弃置的**，std::thread 不可复制。两个 std::thread 不可表示一个线程，std::thread 对线程资源是**独占所有权**。

```cpp
thread(const thread&) = delete;
```

**4）**构造新的 `std::thread` 对象并将它与执行线程关联。**表示新的执行线程开始执行**。⭐ ⭐⭐⭐ ⭐**（重要）**

```cpp
template <class _Fn, class... _Args, enable_if_t<!is_same_v<_Remove_cvref_t<_Fn>, thread>, int> = 0>
    _NODISCARD_CTOR_THREAD explicit thread(_Fn&& _Fx, _Args&&... _Ax) {
        _Start(_STD forward<_Fn>(_Fx), _STD forward<_Args>(_Ax)...);
    }
```

该构造函数是最常使用的，同时也是最复杂的。

- **_Fn**：这是传递给线程的**可调用对象类型**。它可以是普通函数、lambda表达式、函数对象等。
- **_Args…**：这是一个「参数包」，代表可变参数类型，包含传递给 _Fn 的参数，允许传入任意数量和类型的参数。
- **enable_if_t**：这是一个 ***[SFINAE](https://link.zhihu.com/?target=https%3A//zh.cppreference.com/w/cpp/language/sfinae)***（Substitution Failure Is Not An Error）技术，用于在模板实例化过程中进行**条件编译**。这里的条件 `!is_same_v<_Remove_cvref_t<_Fn>, thread>` 确保 **_Fn** 的类型在去除 **const/volatile 修饰和引用**后，不是 `std::thread` 类型本身，从而避免将 `std::thread` 对象作为函数参数，进一步**避免线程的拷贝。**
- `_Fn&&` 和 `_Args&&` 被称为**转发引用**，它们会根据传入参数的类型自动推断为左值引用或右值引用
  - 如果传入的参数是右值（比如使用 `std::move`），则 `_Fn&&` 和 `_Args&&` 会因为**引用折叠**被推导为右值引用；如果是左值，则会被推导为左值引用。
    - 当传入`_Args`的类型是int&（左值）时，后面加&&->int& &&折叠为int&
      - 如果传递的是左值，类型 `T` 会被推导为**左值引用**类型，即 `int&`。这是因为左值的类型在模板参数中会被视为它本身的引用类型。

    - 当传入_Args的类型是int&（左值引用）时，后面加&&->int& &&折叠为int&
    - 当传入_Args的类型是int（右值）时，后面加&&->int&&折叠为int&&
    - 当传入_Args的类型是int&&（右值引用）时，后面加&&->int&&折叠为int&&

- `std::forward<_Fn>(_Fx) 和 std::forward<_Args>(_Ax)...` 会保留参数的值类别（左值或右值），确保可以进行适当的移动或拷贝，**避免传递参数时的临时对象（右值）被强制转换为左值的问题**。

### 3.2.2 关于第四个构造函数的一些疑问

> 1. 关于这个约束你可能有问题，因为`std::thread`他并没有`operator()`的重载，不是可调用类型，也就是说不能将 `std::thread` 作为可调用参数传入，那么这个`enable_if_t`的意义是什么呢？

```cpp
struct X{
    X(X&& x)noexcept{} // 移动构造函数
    template <class Fn, class... Args> 
    X(Fn&& f,Args&&...args){} // 模板构造函数
    X(const X&) = delete;
};

X x1{ [] {} };
X x2{ x }; // 选择到了有参构造函数，不导致编译错误
```

在上段代码中，创建了一个 X 对象 x1，通过模板构造函数，传入了一个 Lambda 表达式（无参数的空函数）。模板构造函数匹配成功，因此 x1 被成功构造。

当试图通过已有的 X 对象 x1 创建另一个 X 对象 x2 时，编译器会选择**模板构造函数**。这是因为 x1 是一个 X 类型的对象，而模板构造函数可以接受任意类型（包括 X），并且与参数类型的匹配规则使得它可以接受一个 X 对象。这个过程不会导致编译错误，因为***模板构造函数并不依赖于传入的对象是否是可调用的（构造函数的选择是基于类型匹配和参数的匹配，而不是基于可调用性）***，尽管 x1 不是可调用类型，编译器选择了这个构造函数来匹配。

以上这段代码可以正常的通过编译。这是重载决议的事情，但我们知道，**std::thread是不可复制的，**这种代码自然不应该让它通过编译，选择到我们的有参构造，所以我们添加一个约束让其不能选择到我们的有参构造：

```cpp
template <class Fn, class... Args, std::enable_if_t<!std::is_same_v<std::remove_cvref_t<Fn>, X>, int> = 0>
```

这样，这段代码就会正常的出现编译错误，信息如下：

```cpp
error C2280: “X::X(const X &)”: 尝试引用已删除的函数
note: 参见“X::X”的声明
note: “X::X(const X &)”: 已隐式删除函数
```

> 2. **_NODISCARD_CTOR_THREAD**是什么？

**_NODISCARD_CTOR_THREAD** 的实现：

```cpp
#define _NODISCARD_CTOR_THREAD                                                     
    _NODISCARD_CTOR_MSG("This temporary 'std::thread' is not joined or detached, " 
                        "so 'std::terminate' will be called at the end of the statement.")
```

**_NODISCARD_CTOR_THREA**是一个宏定义，防止线程在不适当的时候被销毁。也就是一段警告消息，用于提醒开发者，如果一个临时的 std::thread 对象在声明结束时既没有加入（joined）也没有分离（detached），程序将调用std::terminate终止执行。

## 3.3 [_Start](https://link.zhihu.com/?target=https%3A//github.com/microsoft/STL/blob/8e2d724cc1072b4052b14d8c5f81a830b8f1d8cb/stl/inc/thread%23L72-L87)

在第四个构造函数中，使用了[_Start](https://link.zhihu.com/?target=https%3A//github.com/microsoft/STL/blob/8e2d724cc1072b4052b14d8c5f81a830b8f1d8cb/stl/inc/thread%23L72-L87) 函数 ，该函数用于将构造函数的参数全部**完美转发**，是第四个构造函数的核心。⭐ ⭐⭐⭐ ⭐

```cpp
{
    _Start(_STD forward<_Fn>(_Fx), _STD forward<_Args>(_Ax)...);
}
```

_Start 函数的实现如下：

```cpp
template <class _Fn, class... _Args>
void _Start(_Fn&& _Fx, _Args&&... _Ax) {
    using _Tuple                 = tuple<decay_t<_Fn>, decay_t<_Args>...>;
    auto _Decay_copied           = _STD make_unique<_Tuple>(_STD forward<_Fn>(_Fx), _STD forward<_Args>(_Ax)...);
    constexpr auto _Invoker_proc = _Get_invoke<_Tuple>(make_index_sequence<1 + sizeof...(_Args)>{});

    _Thr._Hnd =
        reinterpret_cast<void*>(_CSTD _beginthreadex(nullptr, 0, _Invoker_proc, _Decay_copied.get(), 0, &_Thr._Id));

    if (_Thr._Hnd) { // ownership transferred to the thread
        (void) _Decay_copied.release();
    } else { // failed to start thread
        _Thr._Id = 0;
        _Throw_Cpp_error(_RESOURCE_UNAVAILABLE_TRY_AGAIN);
    }
}
```

### 3.3.1 定义元组来存储函数对象和函数的参数

```cpp
using _Tuple = tuple<decay_t<_Fn>, decay_t<_Args>...>;
```

`std::decay_t` 是一个类型特征，它用于将**类型“衰变”**，也就是说它可以用于

- 移除引用（将引用类型转换为其基础类型）
- 移除 const 和 volatile 修饰符
- 移除数组和函数类型的修饰符（将数组和函数转换为指针类型）

`_Tuple`：表示存储用户可调用对象及其参数的**元组类型**。



### 3.3.2 创建元组实例

```cpp
auto _Decay_copied = _STD make_unique<_Tuple>(_STD forward<_Fn>(_Fx), _STD forward<_Args>(_Ax)...);
```

`std::forward` 用于完美转发参数，可以确保传递给其他函数的参数保持其原有的类型；

```cpp
_STD forward<_Fn>(_Fx), _STD forward<_Args>(_Ax)...
```

上段代码将 _Fx 和 _Ax 参数转发到构造函数中，这里 _Fx 和 _Ax 是传入的参数，它们的类型分别对应 _Fn&& 和 _Args&&（引用折叠） 。通过该实例，可以创建一个独占有权的unique_ptr保存一个Tuple对象，Tuple对象包含函数对象和函数的参数。也就是说，这行代码的目的是存**储传入的可调用对象**和**形参的副本**。

自此以后，传递给可调用对象的实参其实都是从我们构造的元组中取出来的，而且是通过`std::move`的方式取出来，根本就不是原本传入给thread的参数了，所以在可调用对象内部对形参类型为左值引用的参数进行修改时，自然也不会对外部传入thread的参数产生影响。因为存储在元组中的参数是副本（通过传入参数构造了一个元组实例，那么从元组取出的数组自然也不是原本数据了），并不是原本传入的参数，而是我们构造的元组实例存储的对象数据。

> 可调用对象的类型没有发生改变，但传给可调用对象的参数其实是形参的**副本**，而不是形参

### 3.3.3 定义线程启动函数

```cpp
constexpr auto _Invoker_proc = _Get_invoke<_Tuple>(make_index_sequence<1 + sizeof...(_Args)>{})
```

调用**[_Get_invoke](https://link.zhihu.com/?target=https%3A//github.com/microsoft/STL/blob/8e2d724cc1072b4052b14d8c5f81a830b8f1d8cb/stl/inc/thread%23L65-L68)** 函数，传入 _Tuple 类型和一个**参数序列的索引序列（为了遍历形参包）**。这个函数用于获取一个函数指针，指向了一个静态成员函数 `_Invoke`，它是线程实际执行的函数。

其中 **_Get_invoke** 和 **_Invoke** 的实现：

```cpp
// _Get_invoke 函数的实现
template <class _Tuple, size_t... _Indices>
 _NODISCARD static constexpr auto _Get_invoke(index_sequence<_Indices...>) noexcept {
     return &_Invoke<_Tuple, _Indices...>;
 }

// _Invoke 函数的实现
template <class _Tuple, size_t... _Indices>
static unsigned int __stdcall _Invoke(void* _RawVals) noexcept /* terminates */ {
    // adapt invoke of user's callable object to _beginthreadex's thread procedure
    const unique_ptr<_Tuple> _FnVals(static_cast<_Tuple*>(_RawVals));
    _Tuple& _Tup = *_FnVals.get(); // avoid ADL, handle incomplete types
    _STD invoke(_STD move(_STD get<_Indices>(_Tup))...);
    _Cnd_do_broadcast_at_thread_exit(); // TRANSITION, ABI
    return 0;
}
```

#### a.  _Get_invoke

```cpp
// _Get_invoke 函数的实现
template <class _Tuple, size_t... _Indices>
 _NODISCARD static constexpr auto _Get_invoke(index_sequence<_Indices...>) noexcept {
     // 返回的是一个指向特定模板实例 _Invoke 的函数指针，并没有调用该函数
     return &_Invoke<_Tuple, _Indices...>;
 }
```

`_Get_invoke` 函数很简单，就是接受一个元组类型，和形参包的索引，传递给 `_Invoke` 静态成员函数模板，实例化，获取它的函数指针（并没有传递给`_Invoke`任何实参参数，仅仅只是实例化了这个模板函数，并获取它的指针返回）。

> 注意：`return&_Invoke<_Tuple, _Indices...>;` 实际上没有直接给 `_Invoke` 函数提供参数，这是因为它是在返回一个实例化后的函数指针，而不是调用这个函数。返回的函数指针在线程启动时会被调用，并传递一个参数`（void* _RawVals）`，在 `_Invoke` 中进行处理。
>  `Get_invoke` 中没有调用`_Invoke`函数，`_Invoke`函数只在线程启动时会被调用

`std::index_sequence` 是一个类型，表示一个由一系列整数（索引）构成的序列，常用于参数包展开，让我们能够在模板中以索引方式访问和操作参数。`std::index_sequence` 经常和 `std::make_index_sequence<N>` 和 `std::index_sequence_for<Ts...>`一起配套使用，前者用于生成一个 `std::index_sequence`，其中包含从 0 到 N-1 的索引，后者用于生成一个 `std::index_sequence`，其大小与类型参数包 Ts 相同，并且索引顺序与类型参数的顺序相同，比如：

```cpp
// std::make_index_sequence<3> 生成的类型为：
std::index_sequence<0, 1, 2>
// std::index_sequence_for<int, double, char> 生成的类型为：
std::index_sequence<0, 1, 2>
```

举例：

```cpp
// 通过索引展开参数包的函数
template <typename... Args, std::size_t... Is>
void printArgs(std::index_sequence<Is...>, Args&&... args) {
    // C++17 的折叠表达式，用于依次输出每个参数的值和索引
    ((std::cout << "Argument " << Is << ": " << args << std::endl), ...);
}

// 接口函数，生成索引序列
template <typename... Args>
void print(Args&&... args) {
    // 首先完美转发，然后生成一个序列std::index_sequence<0, 1, 2, 3>，最后传给printArgs函数
    printArgs(std::index_sequence_for<Args...>(), std::forward<Args>(args)...);
}

int main() {
    print(10, 3.14, "Hello", 'A');
    return 0;
}
```

输出为：

```cpp
Argument 0: 10
Argument 1: 3.14
Argument 2: Hello
Argument 3: A
```

#### b. _Invoke

当线程启动时，`_Invoke`会被调用；而在Get_invoke函数中，只会获得一个实例化的 `_Invoke`指针，并没有调用该`_Invoke`函数（没有给`_Invoke`赋予参数，仅仅只实例化）。

```cpp
// _Invoke 函数的实现
template <class _Tuple, size_t... _Indices>
static unsigned int __stdcall _Invoke(void* _RawVals) noexcept /* terminates */ {
    // adapt invoke of user's callable object to _beginthreadex's thread procedure
    const unique_ptr<_Tuple> _FnVals(static_cast<_Tuple*>(_RawVals));
    _Tuple& _Tup = *_FnVals.get(); // avoid ADL, handle incomplete types
    _STD invoke(_STD move(_STD get<_Indices>(_Tup))...);
    _Cnd_do_broadcast_at_thread_exit(); // TRANSITION, ABI
    return 0;
}
```

`_Invoke` 是重中之重，它是线程实际执行的函数。当线程启动时，`_Invoke `函数被调用并传入一个`_Tuple`对象(`_RawVals`就相当于我们构造的元组实例，只不过将它的类型转换为了`void*`)，包含了可调用对象及其参数。如你所见它的形参类型是 `void*` ，这是必须的，要符合 `_beginthreadex` 执行函数的类型要求。虽然是 `void*`，但是我可以将它转换为 `_Tuple*` 类型，构造一个独占智能指针指向包含可调用对象及其参数的元组，然后调用 `get()` 成员函数获取底层指针，解引用指针，得到元组的引用初始化_ `_Tup `(我们一开始构造的元组实例)。此时，我们就**可以调用可调用对象**（用户传给thread的可调用对象）

这里有一个形参包展开，`_STD get<_Indices>(_Tup))...`，`_Tup` 就是 `std::tuple` 的引用，我们使用 `std::get<>` 获取元组存储的数据，需要传入一个索引，这里就用到了 `_Indices`。展开之后，就等于 invoke 就接受了我们构造 std::thread 传入的可调用对象，调用可调用对象的参数，invoke 就可以执行了。

```cpp
_STD invoke(_STD move(_STD get<_Indices>(_Tup))...);
```

使用 `std::invoke` 调用存储在 _Tup 中的**可调用对象**。`std::get<_Indices>(_Tup)` 提取元组中的元素，`_STD move`确保将元素以**右值**形式传递，避免不必要的复制。其实也就是将可调用对象和参数副本传递给`std::invoke`函数罢了。

```cpp
CONSTEXPR17 auto invoke(_Callable&& _Obj, _Ty1&& _Arg1, _Types2&&... _Args2) noexcept(
    noexcept(_Invoker1<_Callable, _Ty1>::_Call(
        static_cast<_Callable&&>(_Obj), static_cast<_Ty1&&>(_Arg1), static_cast<_Types2&&>(_Args2)...)))
```

而`std::invoke`函数内部调用了`_Call`函数，`_Call`函数的作用就是**调用可调用对象，并传递给参数**，可以理解为向change_param（在1.6中举例子的用的函数）传递int类型的右值数据（因为是通过`std::move`传递的）

```
change_param(int&& _Arg1)
```

这与`change_param`的定义不符合，`change_param`参数为左值引用， 不能绑定右值，也就是编译错误的原因。

> 所以，传给可调用对象的实参并不是用户传给thread的参数，而是线程内部会将传入的参数先进行delay（解除cv和引用）并保存到`_Decay_copied （tuple）`实例中，然后在`_Invoke` 函数调用可调用对象时，使用 std::move 将其以**右值的方式**传递至可调用对象。也就是说，**传给可调用对象的参数是二手(经过一系列处理)的，并不是传给thread的参数**。

### 3.3.4 启动线程

```cpp
_Thr._Hnd = reinterpret_cast<void*>(_CSTD _beginthreadex(nullptr, 0, _Invoker_proc, _Decay_copied.get(), 0, &_Thr._Id))
```

调用 `_beginthreadex` 函数来启动一个线程，并将线程句柄存储到 `_Thr._Hnd` 中。传递给线程的参数为 `_Invoker_proc`（之前通过 `_Get_invoke` 生成的一个函数指针，指向用于执行用户可调用对象的 `_Invoke` 函数）和 `_Decay_copied.get()`（存储了函数对象和参数的副本的指针）。

> 这行代码的整体作用是使用 `_beginthreadex` 创建一个新线程，执行 `_Invoker_proc` 函数，并将相关的参数传递给它，新线程的句柄被存储在 `_Thr._Hnd` 中，以便后续对线程进行管理。

### 3.3.5 其他

```cpp
if (_Thr._Hnd) { // ownership transferred to the thread
        (void) _Decay_copied.release();
    } else { // failed to start thread
        _Thr._Id = 0;
        _Throw_Cpp_error(_RESOURCE_UNAVAILABLE_TRY_AGAIN);
    }
```

- 如果线程句柄

  ```
  _Thr._Hnd
  ```

  不为空，则表示线程已成功启动，将独占指针的所有权转移给线程 	

  - 释放独占指针的所有权，因为已经将参数传递给了线程（原本的一手数据已经二手传递给了可调用对象）

- 如果线程启动失败，则进入这个分支 

  - 将线程ID设置为0
  - 抛出一个 C++ 错误，表示资源不可用，请再次尝试

## 3.4 std::ref

> 为什么`std::ref`可以保存参数的引用呢？实现在thread修改参数值，影响到外部传入参数值的效果？

```cpp
template <class _Ty>
_NODISCARD _CONSTEXPR20 reference_wrapper<_Ty> ref(_Ty& _Val) noexcept {
    return reference_wrapper<_Ty>(_Val);
}
```

`reference_wrapper`是一个类类型，说白了就是**将参数的地址和类型保存起来**。

```cpp
_CONSTEXPR20 reference_wrapper(_Uty&& _Val) noexcept(noexcept(_Refwrap_ctor_fun<_Ty>(_STD declval<_Uty>()))) {
    _Ty& _Ref = static_cast<_Uty&&>(_Val);
     _Ptr      = _STD addressof(_Ref);
}
```

当我们要使用这个类对象时，自动转化为**取内部参数的地址里的数据**即可，就达到了和实参关联的效果

```cpp
_CONSTEXPR20 operator _Ty&() const noexcept {
    return *_Ptr;
}
_NODISCARD _CONSTEXPR20 _Ty& get() const noexcept {
    return *_Ptr;
}
```

所以我们可以这么理解通过s`std::ref`传递给thread构造函数的参数：`std::Ref`传入的参数仍然作为**右值**被保存，如`ref(int)`实际是作为`reference_wrapper(int)`对象保存在threa的类成员里。而调用的时候触发了**仿函数`()`**进而获取到**外部实参的地址内**的数据。

## 3.5 总结

通过对thread源码的解读，也就明白了为什么1.6的疑问：

> 当在子线程中调用函数时，为什么**即使**参数在函数定义中是引用类型（如 int&），如果在 std::thread 创建线程时直接传递一个变量（如 some_param），这个变量仍会被复制到线程中，子线程内部的修改不会影响主线程中的原始变量？

因为thread的实现中将类型先经过 **decay（解除cv、引用）** 处理，如果要传递引用，则必须用类包装一下才行，使用**std::ref**（不会被decay解除）函数就会返回一个包装对象。

然后传给可调用对象的实参并不是用户传给thread的参数，而是线程内部会将传入的参数先进行delay（解除cv和引用）并保存到`_Decay_copied （tuple）`实例中，然后在`_Invoke` 函数调用可调用对象时，使用 std::move 将其以**右值的方式**传递至可调用对象。也就是说，**传给可调用对象的参数是二手(经过一系列处理，最后传入我们构造元组实例中的数据副本)的，并不是传给thread的参数**。

