---
# type: "about"
title: 并发编程（6）——future、promise、async，线程池
date: 2024-11-04 13:58:01
categories:
- 并发编程
tags: 
- future
- promise
- async
- 线程池
- packaged_task
- shared_future
typora-root-url: ./.. 
---



# 六、day6

今天学习如何使用std::future、std::async、std::promise。主要内容包括：





# 1. future与async

在上一节中，我们等待地铁到站的过程中可以通过条件变量提醒我们是否到站，而本节中我们可以通过`std::future`处理地铁到站的情况。举个例子：我们在车站等车，可能会做一些别的事情打发时间，比如玩手机、和友人聊天等。不过，我们始终在等待一件事情：***车到站***。

C++ 标准库将这种事件称为 [future](https://zh.cppreference.com/w/cpp/thread#.E6.9C.AA.E6.9D.A5.E4.BD.93)。它用于处理线程中需要等待某个事件的情况，线程知道预期结果。等待的同时也可以执行其它的任务。

C++ 标准库有两种 future，都声明在 [`future`](https://zh.cppreference.com/w/cpp/header/future) 头文件中：独占的 [`std::future`](https://zh.cppreference.com/w/cpp/thread/future) 、共享的 [`std::shared_future`](https://zh.cppreference.com/w/cpp/thread/shared_future)。它们的区别与 `std::unique_ptr` 和 `std::shared_ptr` 类似。同一事件仅仅允许关联唯一一个`std::future` 实例，但可以关联多个 `std::shared_future` 实例。它们都是模板，它们的模板类型参数，就是其关联的事件（函数）的**返回类型**。当多个线程需要访问一个独立 `future` 对象时， 必须使用互斥量或类似同步机制进行保护。而多个线程访问同一共享状态，若每个线程都是通过其自身的 `shared_future` 对象副本进行访问，则是安全的。

> `std::future` 是**只能移动**（拷贝构造和拷贝赋值被delete）的，其所有权可以在不同的对象中互相传递，但只有一个对象可以获得特定的同步结果。而 `std::shared_future` 是**可复制的**，多个对象可以指代同一个共享状态。

## 1.1 async与future的配合使用

假设需要执行一个耗时任务并获取其返回值，但是并不急切的需要它。那么就可以启动新线程执行，但是***`std::thread` 不能直接从线程获取返回值（可以使用引用或指针直接将数据存储至指定内存，而不用显式返回**）*。不过我们可以使用 [`std::async`](https://zh.cppreference.com/w/cpp/thread/async) 函数模板。

使用 *`std::async` 启动一个**异步任务**（也就是创建一个子线程执行相关任务，主线程可以执行自己的任务），它会返回一个 `std::future` 对象*，这个对象和任务关联，将持有任务最终执行后的结果。当需要任务执行结果的时候，只需要调用 [`future.get()`](https://zh.cppreference.com/w/cpp/thread/future/get) 成员函数，就会**阻塞**当前线程直到 `future` 为就绪为止（即任务执行完毕），返回执行结果。[`future.valid()`](https://zh.cppreference.com/w/cpp/thread/future/valid) 成员函数检查 future 当前是否关联共享状态，即是否当前关联任务。如果还未关联，或者任务已经执行完（调用了 get()、set()），都会返回 **`false`**。

举一个例子：

```c++
#include <iostream>
#include <future>
#include <chrono>
// 定义一个异步任务
std::string fetchDataFromDB(std::string query) {
    // 模拟一个异步任务，比如从数据库中获取数据
    std::this_thread::sleep_for(std::chrono::seconds(5));
    return "Data: " + query;
}
int main() {
    // 使用 std::async 异步调用 fetchDataFromDB
    // 使用 resultFromDB 存储 fetchDataFromDB 返回的结果
    std::future<std::string> resultFromDB = std::async(std::launch::async, fetchDataFromDB, "Data");
    // 在主线程中做其他事情
    std::cout << "Doing something else..." << std::endl;
    // 从 future 对象中获取数据
    // 在 get 调用之后，主线程一直被阻塞，直至 fetchDataFromDB 函数返回结果
    std::string dbData = resultFromDB.get();
    std::cout << dbData << std::endl;
    return 0;
}
```

在这个示例中，`std::async` 创建了一个新的线程（或从内部线程池中挑选一个线程）并**自动**与一个 `std::promise` 对象相关联。`std::promise` 对象被传递给 `fetchDataFromDB` 函数，函数的返回值被存储在 `std::future` 对象中。在主线程中，我们可以使用 `std::future::get` 方法从 `std::future` 对象中获取数据（使用`std::future::get` 方法后调用该函数的线程处于阻塞状态，直至收到数据）。

> 注意，在使用 `std::async` 的情况下，我们必须使用 `std::launch::async` 标志来明确表明我们希望函数异步执行

上面代码的输出为：

```c++
Doing something else...
Data: Data
```

显然，在使用 `std::async` 异步调用 `fetchDataFromDB`函数时，会创建一个子线程执行`fetchDataFromDB`函数，而主线程继续做其他事件，比如输出`Doing something else...`。当我们需要`fetchDataFromDB`函数的返回结果时，显式调用`std::future::get` 方法获取（在这个过程中，主线程会被阻塞，直至从`future`对象`resultFromDB`中获取到函数的返回结果，并将其存储至`string`类型`dbData`中）

## 1.2 async

与 `std::thread` 一样，`std::async` 支持***任意可调用对象***，以及**传递**调用参数。包括支持使用 `std::ref` ，以及`std::move`。我们下面详细聊一下 `std::async` 参数传递的事。

std::async支持**所有**[可调用(Callable)](https://zh.cppreference.com/w/cpp/named_req/Callable)对象，并且也是默认按值复制（原因可以参考我之前写的关于thread函数源码解析那部分的文章），**必须使用 `std::ref` 才能传递引用**（左值引用）。并且它和 `std::thread` 一样，内部会将保有的参数副本转换为**右值表达式进行传递**，这是为了那些**只支持移动的类型**，左值引用没办法引用右值表达式，所以如果不使用 `std::ref`，这里 `void f(int&)` 就会导致编译错误，如果是 `void f(const int&)` 则可以通过编译，不过引用的不是我们传递的局部对象。

```c++
void f(const int& p) {}
void f2(int& p ){}

int n = 0;
std::async(f, n);   // OK! 可以通过编译，不过引用的并非是局部的n
std::async(f2, n);  // Error! 无法通过编译
```

n是一个左值，传入async之后的处理过程和thread一样。首先，async内部会将其cv修饰符和引用类型去除，然后保存到一个tuple元组中，这个元组存储了可调用对象和去除修饰后的参数，最后将元组中的参数通过`std::move`传递给可调用对象。

所以，这里n是左值，传入async后变为右值，如果将该类型传入给`f`那么可以编译，因为`const int&`可以接受右值，但是`f2`不接受右值，只接受左值，所以`f2`会报错，表示传入类型错误。

除此之外，async 和 thread 一样，可以接受**只移动类型**：

```c++
struct move_only{
    move_only() { std::puts("默认构造"); }
    move_only(move_only&&)noexcept { std::puts("移动构造"); }
    move_only& operator=(move_only&&) noexcept {
        std::puts("移动赋值");
        return *this;
    }
    move_only(const move_only&) = delete;
};

move_only task(move_only x){
    std::cout << "异步任务 ID: " << std::this_thread::get_id() << '\n';
    return x;
}

int main(){
    move_only x;
    std::future<move_only> future = std::async(task, std::move(x));
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "main\n";
    move_only result = future.get();  // 等待异步任务执行完毕
}
```

定义一个只能移动、不能复制的结构体`move_only`。

`std::async` 会在一个**新线程**中异步执行 `task` 函数。如果传入的是 `std::move(x)`，那么 主线程中`x` 的资源会被移动到 `task` 中。移动构造函数会被调用，表示资源已成功转移。async内部接受右值，thread也可以。

### 1.2.1 async的[执行策略](https://zh.cppreference.com/w/cpp/thread/launch)

`std::async`除传递可调用对象、对象参数之外，还需要传递枚举值（也叫策略，比如上面的std::launch::async），这些策略在`std::launch`枚举中定义。除了`std::launch::async`之外，还有以下策略：

1. `std::launch::deferred`：这种策略意味着任务将在需要结果时**同步**执行。惰性求值，**不创建线程**，等待 `future` 对象调用 `wait` 或 `get` 成员函数的时候执行任务。
2. `std::launch::async` 在不同**线程上**执行异步任务。
3. `std::launch::async | std::launch::deferred`：这种策略是上面两个策略的组合。任务可以在一个单独的线程上异步执行，也可以延迟执行，具体取决于实现。

**默认**情况下，`std::async`使用`std::launch::async | std::launch::deferred`策略。这意味着任务可能异步执行，也可能延迟执行，具体取决于实现。典型情况是，如果系统资源充足，并且异步任务的执行不会导致性能问题，那么系统可能会选择在新线程中执行任务。但是，如果系统资源有限，或者延迟执行可以提高性能或节省资源，那么系统可能会选择延迟执行。

> 然而值得注意的是，在 MSVC STL 的实现中，`launch::async | launch::deferred` 与 `launch::async` 执行策略**毫无区别**，这一点我们将单独写一篇解析async的源码。
>
> 简而言之，使用 `std::async`，只要不是 `launch::deferred` 策略，那么 MSVC STL 实现中都是必然在线程中执行任务。因为是线程池，所以执行新任务是否创建新线程，任务执行完毕线程是否立即销毁，***不确定***。

举个例子验证：

```c++
void f(){
    std::cout << std::this_thread::get_id() << '\n';
}

int main(){
    std::cout << std::this_thread::get_id() << '\n';
    auto f1 = std::async(std::launch::deferred, f);
    f1.wait(); // 在 wait() 或 get() 调用时执行，不创建线程
    auto f2 = std::async(std::launch::async,f); // 创建线程执行异步任务
    auto f3 = std::async(std::launch::deferred | std::launch::async, f); // 实现选择的执行方式
}
```

如果系统资源充足的情况下，代码首先会将主线程的`id1`打印出来，然后执行`std::launch::deferred`策略，因为该策略不会创建新线程，所以执行的线程同样是主线程；最后，分别使用`std::launch::async`和`std::launch::deferred | std::launch::async`执行函数，发现二者分别创建了一个新的线程。

输出如下：

```c++
140371524962112
140371524962112
140371520648768
140371512256064
```

## 1.3 future的wait和get

1. **std::future::get()**:

`std::future::get()` 是一个阻塞调用，用于获取 `std::future` 对象表示的值或异常。如果异步任务还没有完成，`get()` 会**阻塞当前线程**，直到任务完成。如果任务已经完成，`get()` 会立即返回任务的结果。重要的是，**`get()` 只能调用一次**，因为它会移动或消耗掉 `std::future` 对象的状态。一旦 `get()` 被调用，或者被**移动**，`std::future` 对象就不能再被用来获取结果。

2. **std::future::wait()**:

`std::future::wait()` 也是一个阻塞调用，但它与 `get()` 的主要区别在于 `wait()` 不会返回任务的结果。它只是等待异步任务完成。如果任务已经完成，`wait()` 会立即返回。如果任务还没有完成，`wait()` 会**阻塞当前线程**，直到任务完成。与 `get()` 不同，**`wait()` 可以被多次调用**，它不会消耗掉 `std::future` 对象的状态。

总结一下，这两个方法的主要区别在于：

- `std::future::get()` 用于获取并返回任务的结果，而 `std::future::wait()` 只是等待任务完成。
- `get()` 只能调用一次，而 `wait()` 可以被多次调用。
- 如果任务还没有完成，`get()` 和 `wait()` 都会阻塞当前线程，但 `get()` 会一直阻塞直到任务完成并返回结果，而 `wait()` 只是在等待任务完成。

我们可以使用std::future的wait_for()或wait_until()方法来检查异步操作是否已完成。这些方法返回一个表示操作状态的std::future_status值。

```c++
if(fut.wait_for(std::chrono::seconds(0)) == std::future_status::ready) {  
    // 操作已完成  
} else {  
    // 操作尚未完成  
}
```

### 1.3.1 常见的两个问题

1. 如果从 `std::async` 获得的 [`std::future`](https://zh.cppreference.com/w/cpp/thread/future) 没有被移动或绑定到引用，那么在完整表达式结尾， `std::future` 的**[析构函数](https://zh.cppreference.com/w/cpp/thread/future/~future)将阻塞，直到异步任务完成**。因为临时对象的生存期是从其创建开始，到表达式结束时（即完整表达式的末尾）自动结束。

   ```c++
   std::async(std::launch::async, []{ f(); }); // 临时量的析构函数等待 f()
   std::async(std::launch::async, []{ g(); }); // f() 完成前不开始
   ```

   如上述代码段，`std::async` 创建了一个临时的 `std::future` 对象，它将持有异步任务的结果，但它并没有将该 `std::future` 对象保存，所以在表达式结束时会被销毁。这导致`std::future` 的析构函数在任务完成前被调用，从而**阻塞主线程**，直到 `f()` 执行完成。

   > `std::future` 的析构函数类似于RAII机制，在析构调用的时候执行类似于线程的`join()`函数，等待async任务执行完毕后才完全销毁。

   后续的异步任务（例如 `g()`）会等待 `f()` 完成后才开始执行，因为任务的执行被前一个任务的完成所**阻塞**。

   ```c++
   auto future1 = std::async(std::launch::async, []{ f(); });
   auto future2 = std::async(std::launch::async, []{ g(); });
   ```

   为了确保异步任务的正确执行，我们需要将 `std::future` 绑定到一个变量，以延长其生存期（避免提前调用析构函数）。

2. 被移动的 `std::future` 没有所有权，失去共享状态，不能调用 `get`、`wait` 成员函数。

   ```c++
   auto t = std::async([] {});
   std::future<void> future{ std::move(t) }; // 将t的所有权转移
   t.wait();   // Error! 抛出异常
   ```

    `std::future` 对象`t`的所有权被转移给 `std::future` 对象`future`，不可以对`t`调用 `get`、`wait` 成员函数。如同没有线程资源所有权的 `std::thread` 对象调用 `join()` 一样错误，这是移动语义的基本语义逻辑。

## 1.4 异常处理

如果async在执行可调用对象期间发生了异常，`future`对象会将该异常保存下来，我们可以调用 `std::future::get` 方法来获取这个异常。

```c++
void may_throw()
{
    // 这里我们抛出一个异常。在实际的程序中，这可能在任何地方发生。
    throw std::runtime_error("Oops, something went wrong!");
}
int main()
{
    // 创建一个异步任务
    std::future<void> result(std::async(std::launch::async, may_throw));
    try
    {
        // 获取结果（如果在获取结果时发生了异常，那么会重新抛出这个异常）
        result.get();
    }
    catch (const std::exception &e)
    {
        // 捕获并打印异常
        std::cerr << "Caught exception: " << e.what() << std::endl;
    }
    return 0;
}
```

async在异步执行可调用对象`may_throw`时，发生了异常（我们这里手动抛出一个异常表示代码发生异常），我们在try块中调用`get()`函数获取`future`对象的内容，如果内容是正常值，那么代码继续运行；如果内容是异常值，那么会直接被catch块捕获，并对其进行相应的处理。

输出：

```
Caught exception: Oops, something went wrong!
```

# 2. future与 packaged_task

`std::packaged_task`和`std::future`是C++11中引入的两个类，它们用于处理异步任务的结果。

`std::packaged_task`是一个可调用目标（函数、lambda 表达式、bind 表达式或其它函数对象），它包装了一个任务，该任务可以在另一个线程上（**异步**）运行。它可以捕获任务的**返回值或异常**，并将其存储在`std::future`对象中。

```c++
template <class _Ret, class... _ArgTypes>
class packaged_task<_Ret(_ArgTypes...)> {}
```

packaged_task 是一个模板类型，其中：

- `_Ret`：表示可调用目标的返回类型
- `_ArgTypes...`：可调用目标接受的参数类型

```c++
std::packaged_task<double(int, int)> 
std::packaged_task<void()> 
```

所以上面初始化的第一个packaged_task实例表示，接受任何返回类型是double，参数是int,int的可调用对象。

第二个packaged_tas实例表示，，接受任何返回类型是void，且无参数的可调用对象。

> **`std::packaged_task` 重载了 `operator()` 运算符**，并通过重载的 `operator()` 来执行包装的可调用对象。比如在命令`std::packaged_task<void(int)> task(myFunction); `中，执行`task()`其实就算再执行`myFunction()`.

以下是使用`std::packaged_task`和`std::future`的基本步骤：

1. 创建一个`std::packaged_task`对象，该对象包装了要执行的任务。
2. 调用`std::packaged_task`对象的`get_future()`方法，该方法返回一个与任务关联的`std::future`对象。
3. 在另一个**线程**上调用`std::packaged_task`对象的`operator()`，以执行任务。
4. 在需要任务结果的地方，调用与任务关联的`std::future`对象的`get()`方法，以获取任务的返回值或异常。

举例：

```c++
int my_task() {
    std::this_thread::sleep_for(std::chrono::seconds(5));
    std::cout << "my task run 5 s" << std::endl;
    return 42;
}
void use_package() {
    // 创建一个包装了任务的 std::packaged_task 对象，表示返回类型是int，无参数 （Ⅰ）
    std::packaged_task<int()> task(my_task); 
    // 获取与任务关联的 std::future 对象 （Ⅱ）
    std::future<int> result = task.get_future();
    // 在另一个线程上执行任务 （Ⅲ）  
    std::thread t(std::move(task));
    t.detach(); // 将线程与主线程分离，以便主线程可以等待任务完成  
    // 等待任务完成并获取结果  
    int value = result.get();
    std::cout << "The result is: " << value << std::endl;
}
```

因为 `task` 本身是重载了 `operator()` 的，是可调用对象，自然可以传递给 `std::thread` 执行，以及传递调用参数。唯一需要注意的是我们使用了 **`std::move`** ，这是因为 **`std::packaged_task` 只能移动，不能复制**。

> 简而言之，其实 `std::packaged_task` 就是一个“包装”类而已，它本身并没什么特殊的，老老实实执行我们传递的任务，且方便我们获取返回值。
>
> 其实我们使用`std::async`和 `std::packaged_task` 都可以，只不过后者更加精细一些而已。

# 3. promise

C++11引入了`std::promise`和`std::future`两个类，用于实现异步编程。`std::promise`用于在某一线程中设置某个值或异常，之后通过 `std::promise` 对象所创建的`std::future`对象异步获得这个值或异常。

示例：

```c++
#include <iostream>
#include <thread>
#include <future>
void set_value(std::promise<int> prom) {
    // 设置 promise 的值
    prom.set_value(10);
}
int main() {
    // 创建一个 promise 对象
    std::promise<int> prom;
    // 获取与 promise 相关联的 future 对象
    std::future<int> fut = prom.get_future();
    // 在新线程中设置 promise 的值
    std::thread t(set_value, std::move(prom));
    // 在主线程中获取 future 的值
    std::cout << "Waiting for the thread to set the value...\n";
    std::cout << "Value set by the thread: " << fut.get() << '\n';
    t.join();
    return 0;
}
```

在上面的代码中，我们首先创建了一个`std::promise<int>`对象，然后通过调用`get_future()`方法获取与之相关联的`std::future<int>`对象。然后，我们在**新线程**中通过调用`set_value()`方法设置`promise`的值，并在**主线程**中通过调用`fut.get()`方法获取这个值。注意，在调用`fut.get()`方法时，如果`promise`的值还没有被设置，则该方法会**阻塞当前线程**，直到值被设置为止。

```c++
// 程序输出
Waiting for the thread to set the value...
promise set value successValue set by the thread: 10
```

> 同样的 `std::promise` **只能移动**，**不可复制**，所以我们使用了 `std::move` 进行传递。

------

除了 `set_value()` 函数外，`std::promise` 还有一个 [`set_exception()`](https://zh.cppreference.com/w/cpp/thread/promise/set_exception) 成员函数，它接受一个 [`std::exception_ptr`](https://zh.cppreference.com/w/cpp/error/exception_ptr) 类型的参数，这个参数通常通过 [`std::current_exception()`](https://zh.cppreference.com/w/cpp/error/current_exception) 获取，用于指示当前线程中抛出的异常。然后，`std::future` 对象通过 `get()` 函数获取这个异常，如果 `promise` 所在的函数有异常被抛出，则 `std::future` 对象会重新抛出这个异常，从而允许**主线程**捕获并处理它。

```c++
#include <iostream>
#include <thread>
#include <future>
void set_exception(std::promise<void> prom) {
    try {
        // 抛出一个异常
        throw std::runtime_error("自定义异常");
    } catch(...) {
        // 设置 promise 的异常
        prom.set_exception(std::current_exception());
    }
}
int main() {
    // 创建一个 promise 对象
    std::promise<void> prom;
    // 获取与 promise 相关联的 future 对象
    std::future<void> fut = prom.get_future();
    // 在新线程中设置 promise 的异常
    std::thread t(set_exception, std::move(prom));
    // 在主线程中获取 future 的异常
    try {
        std::cout << "Waiting for the thread to set the exception...\n";
        fut.get(); // 获得异常，被后面的catch捕获
    } catch(const std::exception& e) {
        std::cout << "Exception set by the thread: " << e.what() << '\n';
    }
    t.join();
    return 0;
}
```

1. 在异常设置函数`set_exception`中：
   - 首先抛出一个异常 `std::runtime_error`。
   - 捕获到 `std::runtime_error`异常后，使用 `prom.set_exception(std::current_exception())` 将当前异常设置到 `std::promise` 对象中。这会使得与这个 `promise` 相关联的 `future` 在被获取时抛出相同的异常。

2. 主线程中调用 `fut.get()`，此时**主线程会阻塞**，直到 `set_exception` 函数完成并设置了异常。如果 `fut.get()` 捕获到异常，则会在 `catch` 块中输出异常消息。

运行结果：

```
Waiting for the thread to set the exception...
Exception set by the thread: 自定义异常

```

------

> 我们写的是 `promise<int>` ，但是为什么没有使用 `set_value` 设置值？

如果 promise 已经存储值或者异常，再次调用 `set_value`（`set_exception`） 会抛出 [std::future_error](https://zh.cppreference.com/w/cpp/thread/future_error) 异常，将错误码设置为 [`promise_already_satisfied`](https://zh.cppreference.com/w/cpp/thread/future_errc)。这是因为 `std::promise` 对象只能是存储值或者异常其中**一种**，而**无法共存**。

简而言之，`set_value` 与 `set_exception` **二选一**，如果先前调用了 `set_value` ，就不可再次调用 `set_exception`，反之亦然（不然就会抛出异常）。

# 4. shared_future

之前的例子中我们一直使用 `std::future`，但 `std::future` 有一个局限：**future 是一次性的**，它的结果只能被一个线程获取。`get()` 成员函数只能调用一次，当结果被某个线程获取后，`std::future` 就无法再用于其他线程。如果需要进行多次 `get` 调用（**多个线程等待同一个异步操作的结果**），可以考虑 `std::shared_future`。

`std::future` 与 `std::shared_future` 的区别就如同 `std::unique_ptr`、`std::shared_ptr` 一样。同一事件仅仅允许关联唯一一个`std::future` 实例，但可以关联多个 `std::shared_future` 实例

> `std::future` 是**只能移动**的，其所有权可以在不同的对象中互相传递，但只有一个对象可以获得特定的同步结果。而 `std::shared_future` 是**可复制的**，多个对象可以指代同一个共享状态。

在多个线程中对**同一个 `std::shared_future` 对象进行操作时（如果没有进行同步保护）存在条件竞争。而多个线程访问同一共享状态，若每个线程都是通过其自身的 `shared_future` 对象副本**进行访问，则是安全的。因为`std::shared_future` 的设计允许在创建副本后，多个线程可以独立地获取结果，而无需担心条件竞争，因为每个线程操作的是不同的对象。

> 如果需要多个线程对一个`std::shared_future` 对象进行操作，需要将`std::shared_future` **按值传递**给不同线程，切记，**不能按引用传递**。

我这里就不举例子说明按值传递和按引用传递的区别了，直接举例解释`shared_future`是如何使用的：

```c++
#include <iostream>
#include <thread>
#include <future>

void myFunction(std::promise<int>&& promise) {
    // 模拟一些工作
    std::this_thread::sleep_for(std::chrono::seconds(1));
    promise.set_value(42); // 设置 promise 的值
}
void threadFunction(std::shared_future<int> future) {
    try {
        int result = future.get();
        std::cout << "Result: " << result << std::endl;
    }
    catch (const std::future_error& e) {
        std::cout << "Future error: " << e.what() << std::endl;
    }
}
void use_shared_future() {
    std::promise<int> promise;
    std::shared_future<int> future = promise.get_future();
    std::thread myThread1(myFunction, std::move(promise)); // 将 promise 移动到线程中
    // 使用 share() 方法获取新的 shared_future 对象  
    std::thread myThread2(threadFunction, future);
    std::thread myThread3(threadFunction, future);
    
    myThread1.join();
    myThread2.join();
    myThread3.join();
}

```

`promise.get_future()`返回的是一个`std::future`类型的对象，但是被隐式转换为了`std::shared_future`类型的对象。所以我们才可以将`std::shared_future`传递给不同的线程，因为`std::shared_future`是可以拷贝的，而`future`不可以被拷贝。

然后，我们创建一个线程`myThread1`对`promise`设置值，并创建两个独立的线程`myThread2`和`myThread3`分别执行`threadFunction`函数获取`future`中存储的值。可以发现，两个线程是独立的，并且可以多次调用`get()`函数而不会将`future`的状态取消。

代码输出为：

```
Result: 42
Result: 42
```

当然，我们也可以在主线程中再次调用get函数，同样会得到相同的结果：

```c++
    std::thread myThread2(threadFunction, future);
    std::thread myThread3(threadFunction, future);
   
    myThread1.join();
    
    threadFunction(future);
    myThread2.join();
    myThread3.join();
```

输出结果为：

```
Result: 42
Result: 42
Result: 42
```

> 还有一点需要注意，`future`既不可以拷贝给多个线程，也不能移动给多个线程；前者是因为`future`不支持拷贝操作，只支持移动操作；后者是因为`future`的状态是**唯一**的，不能被多个线程共享。而`std::shared_future` 可以**拷贝**给多个线程。

# 5.packaged_task和promise构建线程池

**线程池**是一种多线程处理形式，它处理过程中将任务添加到队列，然后在创建线程后**自动启动**这些任务。线程池线程一般**后台线程**。

- **后台线程**：线程在主程序（前台线程）结束时不会阻止程序的终止。当主程序结束时，后台线程会被强制终止。
- **前台线程**：线程在主程序结束之前必须完成。主程序会等待这些线程完成，然后再退出。

每个线程都使用**默认的堆栈大小**，以**默认的优先级**运行，并处于多线程单元中。如果某个线程在托管代码中空闲（例如等待 I/O 操作完成、等待锁、或者等待其他线程的通知），则线程池将插入另一个辅助线程来使所有处理器保持繁忙。如果所有线程池线程都始终保持繁忙，但队列中包含挂起的工作，则线程池将在一段时间后创建另一个辅助线程但线程的数目永远不会超过**最大值**。超过最大值的线程可以排队，但他们要等到其他线程完成后才启动。

**举例：**

假设我们有一个线程池，它有 4 个线程在工作。现在其中一个线程正在等待文件 I/O 完成，这个操作可能需要几毫秒。在这段等待时间里，如果没有其他线程执行任务，CPU 会处于空闲状态。为了避免这个情况，线程池可以创建一个**新的线程**（辅助线程），来处理任务队列中的其他任务。这样，即使有一个线程在等待，其他线程仍然可以继续执行任务，从而保持 CPU 的高利用率。

> 假设我们有一个线程池，包含 4 个线程，编号为 T1、T2、T3 和 T4。现在我们来看看它们的工作情况：
>
> 1. **线程池中的线程数**：4 个线程（T1、T2、T3、T4）。
> 2. **任务队列中的任务**：假设有 6 个任务需要执行，任务编号为 A、B、C、D、E 和 F。
>
> **初始状态：**
>
> 线程 T1、T2、T3 和 T4 正在处理任务：
>
> - T1 正在处理任务 A。
> - T2 正在处理任务 B。
> - T3 正在处理任务 C。
> - T4 正在处理任务 D。
>
> 假设 T1 处理任务 A 时需要执行文件 I/O 操作，这个操作需要 3 毫秒。在这个操作过程中，T1 将会进入等待状态，而不再执行其他工作。在这种情况下，尽管线程池有 4 个线程，但因为 T1 正在等待 I/O，CPU 可能在这段时间里闲置，导致资源浪费。
>
> 为了避免 CPU 空闲，线程池会采取以下措施：
>
> - **创建辅助线程**：在 T1 进入等待状态时，线程池会检查任务队列，发现还有任务（如 E 和 F）待处理。于是，线程池可以创建一个辅助线程 T5，来处理任务 E 和 F。
>
> **流程：**
>
> - **0 毫秒**：T1 处理任务 A。
> - **3 毫秒**：T1 进入等待状态，T5 开始处理任务 E。
> - **4 毫秒**：T5 完成任务 E。
> - **5 毫秒**：T5 开始处理任务 F。
> - **6 毫秒**：T1 完成 I/O 操作，继续处理任务 A。

线程池可以避免在处理短时间任务时创建与销毁线程的代价，它维护着多个线程，等待着监督管理者分配可并发执行的任务，从而提高了整体性能。

以下是参考博主[恋恋风辰]([恋恋风辰官方博客](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/2VWIJgH3zKEww0BpLnYQX0NMpQ9))博客的线程池源码：

```C++
#ifndef __THREAD_POOL_H__
#define __THREAD_POOL_H__
#include <atomic>
#include <condition_variable>
#include <future>
#include <iostream>
#include <mutex>
#include <queue>
#include <thread>
#include <vector>
class ThreadPool  {
public:
    // delte拷贝构造和拷贝赋值
    ThreadPool(const ThreadPool&) = delete;
    ThreadPool&        operator=(const ThreadPool&) = delete;
    // 简单单例模式的实现
    static ThreadPool& instance() {
        static ThreadPool ins;
        return ins;
    }
    // 对任务(不接受参数并返回 void 的可调用对象)进行包装的包装器，用Task作为别名
    using Task = std::packaged_task<void()>;
    // 析构函数
    ~ThreadPool() {
        stop();
    }
	// 该函数用于插入新任务至队列，并返回新任务的future
    template <class F, class... Args>
    auto commit(F&& f, Args&&... args) -> std::future<decltype(f(args...))> {
        using RetType = decltype(f(args...));
        if (stop_.load())
            return std::future<RetType>{};
        auto task = std::make_shared<std::packaged_task<RetType()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...));
        std::future<RetType> ret = task->get_future();
        {
            std::lock_guard<std::mutex> cv_mt(cv_mt_);
            tasks_.emplace([task] { (*task)(); });
        }
        cv_lock_.notify_one();
        return ret;
    }
    // 返回线程数量
    int idleThreadCount() {
        return thread_num_;
    }
private:
    // 默认构造函数。定义线程池的容量大小
    ThreadPool(unsigned int num = 5)
        : stop_(false) {
            {
                if (num < 1)
                    thread_num_ = 1;
                else
                    thread_num_ = num;
            }
            start();
    }
    // 启动线程池
    void start() {
        for (int i = 0; i < thread_num_; ++i) {
            // 使用容器存储线程对象
            pool_.emplace_back([this]() { // 每个线程执行的lamda函数（线程池执行的任务相同）
                while (!this->stop_.load()) {
                    Task task;
                    {
                        std::unique_lock<std::mutex> cv_mt(cv_mt_);
                        this->cv_lock_.wait(cv_mt, [this] {
                            return this->stop_.load() || !this->tasks_.empty();
                        });
                        if (this->tasks_.empty())
                            return;
                        task = std::move(this->tasks_.front());
                        this->tasks_.pop();
                    }
                    this->thread_num_--;
                    task();
                    this->thread_num_++;
                }
            });
        }
    }
    // 将线程池关闭
    void stop() {
        stop_.store(true); // 将判断变量置为true
        cv_lock_.notify_all(); // 将所有线程唤醒
        for (auto& td : pool_) {
            if (td.joinable()) { // 打印线程池中所有的线程id
                std::cout << "join thread " << td.get_id() << std::endl;
                td.join();
            }
        }
    }
private:
    std::mutex               cv_mt_; // 互斥量
    std::condition_variable  cv_lock_; // 条件变量
    std::atomic_bool         stop_; // 布尔类型，配合条件变量使用
    std::atomic_int          thread_num_; // 线程数量
    std::queue<Task>         tasks_; // 任务队列，每个任务使用packaged_task进行包装
    std::vector<std::thread> pool_; // 使用容器存储线程
};
#endif  // !__THREAD_POOL_H__
```

一些简单的函数和变量我加了注释就不多做解释，这里注意解析下面几个函数。

## 5.1 commit函数

```c++
template <class F, class... Args>
    auto commit(F&& f, Args&&... args) -> std::future<decltype(f(args...))> {
        using RetType = decltype(f(args...)); // 使用RetType作为可调用对象返回类型的别名
        if (stop_.load()) // 线程池是否处于关闭状态
            return std::future<RetType>{};
    
    	// 将可调用对象和参数用装饰器packaged_task进行包装
        auto task = std::make_shared<std::packaged_task<RetType()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...));
        // 从包装器获取future
        std::future<RetType> ret = task->get_future();
        // 在{}内使用lock_guard锁定互斥量，并将task使用emplace插入至任务队列
        {
            std::lock_guard<std::mutex> cv_mt(cv_mt_);
            tasks_.emplace([task] { (*task)(); });
        }
        cv_lock_.notify_one();
        return ret;
    }
```

- 模板类型：

  - `F`：回调函数，线程执行的任务
  - `... Args`：任务的参数列表

- `F&&`和 `Args&&`：**转发引用**，根据传入参数的类型自动推断为左值引用或右值引用

  - 传入的参数是右值（比如使用 `std::move`），被推导为右值引用
  - 传入的参数是左值，被推导为左值引用

- 返回值类型：

  - `std::future<decltype(f(args...))> `：使用可调用对象的返回类型作为`std::future`的模板类型，并将`std::future<推断的类型>`返回
  - `decltype(f(args...))`：自动推断可调用对象f的返回值类型，args...是传给f的实参

- `stop_.load()`：读取 `stop_` 的当前值

  - 如果是`True`，表示线程池关闭状态，直接返回一个使用默认构造函数构造的`std::future`对象
  - 如果是`False`，表示线程池开启状态，继续执行代码

- ```C++
  auto task = std::make_shared<std::packaged_task<RetType()>>(
       std::bind(std::forward<F>(f), std::forward<Args>(args)...));
  ```

  - `std::packaged_task<RetType()>`：包装任务，任务的返回类型是`RetType()`；
  - 需要包装的任务通过`std::bind`绑定起来，并传给包装器`std::packaged_task<RetType()>`	
  - 包装器传递给智能指针`std::make_shared`进行管理

- `ret`：从包装器packaged_task获取future，future存储了可调用对象的返回值

- `[task] { (*task)(); })`：按值捕获上面定义的 task（**伪闭包**），task是一个存储`packaged_task`对象的智能指针，这里插入任务队列的是一个回调函数，该回调函数将会执行 `packaged_task` 包含的任务（函数）

  - packaged_task 重载了 () 运算符，这里其实就相当于再调用可调用对象（使用packaged_task包装的任务）

- 只要队列中插入新任务，就唤醒线程

- 最后，返回新任务的 future 对象

------

```C++
using Task = std::packaged_task<void()>; 
std::queue<Task>         tasks_; 
tasks_.emplace([task] { (*task)(); });
```

如上段代码所示，队列tasks_的元素类型是std::packaged_task<void()>，但为什么插入的是一个lambda函数？

1. **`std::packaged_task<void()>`：**

- `std::packaged_task<void()>` 表示packaged_task接受任何无传递参数并返回 `void` 的可调用对象。它可以封装任何符合这个签名的可调用对象，包括普通函数、函数对象、和 lambda 表达式。

2. **lambda 表达式**：

-  `[task] { (*task)(); }`，这个表达式本身是一个可调用对象。它捕获了外部变量 `task`，并在调用时执行 `(*task)()`，这实际上是调用了 `packaged_task` 封装的函数。

3. **插入到队列**：

- 在使用 `tasks_.emplace(...)` 时，实际上在构造一个 `std::function<void()>` 类型的对象，而这个对象可以被 `std::packaged_task` 接受。
- 由于 lambda 表达式符合 `void()` 的函数签名，因此 `std::packaged_task` 能够接受它并正确地存储。

## 5.2 start函数

```c++
void start() {
        for (int i = 0; i < thread_num_; ++i) {
            // 使用容器存储线程对象
            pool_.emplace_back([this]() { // 每个线程执行的lamda函数（线程池执行的任务相同）
                while (!this->stop_.load()) { // 判断线程池是否停止
                    Task task; // 初始化一个接受无参并无返回值可调用对象的packaged_task
                    {
                        std::unique_lock<std::mutex> cv_mt(cv_mt_);
                        // 条件变量判断，当满足任务队列不为空或者stop_为true，并且当前线程被唤醒时，退出挂起状态
                        this->cv_lock_.wait(cv_mt, [this] {
                            return this->stop_.load() || !this->tasks_.empty();
                        });
                        // 队列为空，那么就只有stop_为true一种情况，此时无任务需要处理，直接退出
                        if (this->tasks_.empty())
                            return;
                        // 处理队列剩余任务
                        task = std::move(this->tasks_.front());
                        this->tasks_.pop();
                    }
                    this->thread_num_--; // 减少一个线程数用来执行task任务
                    task(); // 执行任务
                    this->thread_num_++; // task任务在线程内是同步执行的，所以当task任务执行完后，可用的线程数加一
                }
            });
        }
    }
```

线程池会启动数量为`thread_num_`的线程，每个线程执行以下程序：

- 将当前线程插入至`pool_`容器内进行管理
- 循环判断线程池是否关闭，如果不关闭，那么该线程将会无止境的进行工作
- 为了避免while循环占用CPU资源，使用条件变量挂起当前当前，直至满足（任务队列不为空**或者**stop_为true），**并且**当前线程被唤醒时，退出挂起状态。挂起状态时，当前线程会释放锁让其他线程可以访问共享资源；线程被唤醒时，当前线程会重新拿取锁
  - 如果队列为空，且当前线程被唤醒，那就只有一种可能：线程池关闭。此时无任务需要处理，直接退出
  - 如果队列不为空，取出任务队列的第一个元素
- 因为在当前线程内的任务执行是同步的，所以在执行任务前需要将可用线程数减一，待执行完任务后，可用线程数加一。
- 最后，循环第二步~第四步

## 5.3 如何使用线程池

```C++
int main(){
    int m = 0;
    // 调用线程池执行
    ThreadPool::instance().commit([](int& m) {
        m = 1024;
        std::cout << "inner set m is " << m << std::endl;
        std::cout << "m address is " << &m << std::endl;
        }, m);
    // 主线程执行
    std::this_thread::sleep_for(std::chrono::seconds(3));
    std::cout << "m is " << m << std::endl;
    std::cout << "m address is " << &m << std::endl;
    
    return 0;
}
```

首先，往线程池中的任务队列插入一个新任务，新任务没有返回类型，并且参数只有int；然后再主线程中执行相同的任务。

代码输出为：

```
inner set m is 1024
m address is 0000020BC8566A98
m is 0
m address is 00000027BB18F834
join thread 29392
join thread 6976
join thread 28224
join thread 30688
join thread 25912
```

线程池中的数量为5，然后我们启动一个线程执行我们传入的任务，此时，m被修改为1024。注意，线程池中的其余4个线程处于空闲状态，没有被调用。

随着线程池中的m被改变，但主线程中的m并没有被改变，并且地址也不同。其实和thread、async比较类似，但不完全相同，如果这里直接将m传入给thread，那么编译不会通过，因为lamda的参数是一个int&，但是thread传递给lambda的是一个int&&，左值引用不会接受右值引用。

在这里，lambda函数和参数传递给commit函数，在commit函数中，经过引用转发和原样转发后（m在模板类型中会被推断为int&，注意m不会被模板推断为int，int只能表示右值，可用参考我写的[文章](https://www.aichitudou.cn/2024/11/03/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%EF%BC%881%EF%BC%89%E2%80%94%E2%80%94%E7%BA%BF%E7%A8%8B/)），进入bind函数中的参数类型Args其实是**int&**。但在bind中，会将参数Args首先进行decay取出cv修饰符和引用，此时Args类型为**int**（右值），然后将decay处理后的调用对象和参数保存到pair类型中。如下：

```c++
    using _Seq    = index_sequence_for<_Types...>;
    using _First  = decay_t<_Fx>;
    using _Second = tuple<decay_t<_Types>...>;

    _Compressed_pair<_First, _Second> _Mypair;
```

_Mypair 保存了取出修饰的可调用对象和参数。其中Args的类型是一个int右值，传递给可调用对象的参数也是这个int右值，但是会调用拷贝构造，相当于传递给的是一个副本，所以在bind内部的修改不会影响到外面的m。所以主线程的m和线程池中的m的地址不同。



```
ThreadPool::instance().commit([](int& m) {
    m = 1024;
    std::cout << "inner set m is " << m << std::endl;
    std::cout << "m address is " << &m << std::endl;
    }, std::ref(m));
        
inner set m is 1024
m address is 00000002F28FF834
m is 1024
m address is 00000002F28FF834
join thread 7864
join thread 31712
join thread 11832
join thread 9924
join thread 26476
```

如果将m用std::ref进行封装，那么线程内的修改会影响到线程外的m。

------

> **注意：线程池不能被用于以下两种任务：**

1. 线程池做得任务是并发的，并且无序，不能保证连续性。比如在网络编程中，你的接受线程必须要保证收到的消息顺序是有序的，所以这里就不能用线程池；还比如在逻辑线程中，我需要先处理消息A，再处理消息B，同样是有顺序的不能使用线程池，只能使用单线程来完成。
2. 如果执行的任务互斥性很大，或者说是强关联，比如玩游戏：第一个任务是玩家A进入工会做任务增加工会贡献，第二个任务是工会会长使用这个贡献。而贡献是所有玩家共享的一个资源，有一个公共互斥量，这也不可以用线程池来用，这只能用一个线程来用。因为来回加锁会导致效率大幅度下降，还不如使用一个线程来完成。
