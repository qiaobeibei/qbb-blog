---
title: 网络编程（18）——使用asio协程实现并发服务器（官方案例）
date: 2024-11-03 11:47:55
categories:
- 网络编程
tags: 
- 协程
typora-root-url: ./..
---

## 十八、day18

到目前为止，我们以及学习了单线程同步/异步服务器、多线程IOServicePool和多线程IOThreadPool模型，今天学习如何通过asio协程实现**并发服务器**。

并发服务器有以下几种好处：

- 协程比线程更轻量，创建和销毁协程的开销较小，适合高并发场景
- 协程通常在单线程中运行，避免了多线程带来的资源竞争和同步问题，从而减少了内存使用
- 将回调函数改写为顺序调用，**让异步的函数能够以同步的方式写出来的同时不降低性能**，提高开发效率
- 协程调度比线程调度更轻量化，因为协程是运行在用户空间的，线程切换需要在用户空间和内核空间切换

首先需将C++语言标准换为C++20标准，**协程是在C++20之后引入的新标准**

![img](/images/$%7Bfiilename%7D/4c8c6ab8d03942b8b8dbb1bbfb623baa.png)

## 1. 官方案例

asio官网提供了一个协程并发编程的案例，如下

```cpp
#include <iostream>
#include <boost/asio/co_spawn.hpp>
#include <boost/asio/detached.hpp>
#include <boost/asio/io_context.hpp>
#include <boost/asio/ip/tcp.hpp>
#include <boost/asio/signal_set.hpp>
#include <boost/asio/write.hpp>

using boost::asio::ip::tcp;
using boost::asio::awaitable;
using boost::asio::co_spawn;
using boost::asio::detached;
using boost::asio::use_awaitable;
namespace this_coro = boost::asio::this_coro;

awaitable<void> echo(tcp::socket socket) {
    try {
        char data[1024];
        for (;;) {
            std::size_t n = co_await socket.async_read_some(boost::asio::buffer(data), use_awaitable);
            co_await async_write(socket, boost::asio::buffer(data, n), use_awaitable);
        }
    }
    catch (std::exception& e) {
        std::cout << "Echo exception is " << e.what() << std::endl;
    }
}

awaitable<void> listener() {
    auto executor = co_await this_coro::executor;
    tcp::acceptor acceptor(executor, { tcp::v4(), 10086 });
    for (;;) {
        tcp::socket socket = co_await acceptor.async_accept(use_awaitable);
        co_spawn(executor, echo(std::move(socket)), detached);
    }
}

int main()
{
    try {
        boost::asio::io_context io_context(1); // 1被用于提供有关所需并发级别的提示
        boost::asio::signal_set signals(io_context, SIGINT, SIGTERM);
        signals.async_wait([&](auto, auto) { // 处理几个信号就传入几个参数，这里使用auto自动推断
            io_context.stop();
            });
        co_spawn(io_context, listener(), detached);
        io_context.run();

    }
    catch (std::exception& e) {
        std::cout << "Exception is " << e.what() << std::endl;
    }
}
```

### ***a. 声明***

```cpp
using boost::asio::ip::tcp;
using boost::asio::awaitable;
using boost::asio::co_spawn;
using boost::asio::detached;
using boost::asio::use_awaitable;
namespace this_coro = boost::asio::this_coro;
```

- **awaitable** ：用于定义可以在协程中使用的异步操作，可以通过 **co_await** 关键字等待异步任务的完成，使**异步的函数能够以同步的方式写出来的同时不降低性能**

- **co_spawn**：用于启动新的协程的函数，可以用它来创建新的异步任务并在指定的执行上下文中运行

- **detached**：指示器，表示创建的协程不需要等待其结果。使用 **detached** 后，协程会在后台独立运行

- **use_awaitable**：适配器，指示以协程的方式使用 Boost.Asio 的异步操作，它使得**异步操作**可以与 **co_await** 关键字结合使用。适配器允许将异步操作的结果直接与协程的执行流结合，使得异步调用能够以同步的方式写出，从而避免了手动管理回调函数

- co_await 

  关键字的作用： 

  - 当协程遇到 co_await 时，它会挂起执行，直到被等待的异步操作完成。这允许当前线程释放 CPU，去处理其他任务或协程。
  - 一旦等待的操作完成，协程会自动恢复执行，继续从挂起的地方运行。这样可以避免复杂的回调地狱，提供更直观的控制流。
  - co_await 会自动获取异步操作的结果并将其返回给调用者。例如，如果等待的是一个返回值的异步操作，结果会被赋给相应的变量。
  - 如果在 co_await 等待的异步操作中发生异常，协程可以捕获这些异常，方便进行错误处理。

### ***b. echo()***

```cpp
awaitable<void> echo(tcp::socket socket) {
    try {
        char data[1024];
        for (;;) {
            std::size_t n = co_await socket.async_read_some(boost::asio::buffer(data), use_awaitable);
            co_await async_write(socket, boost::asio::buffer(data, n), use_awaitable);
        }
    }
    catch (std::exception& e) {
        std::cout << "Echo exception is " << e.what() << std::endl;
    }
}
```

**awaitable<void>**类型允许函数在执行时可以被暂停和恢复，这使得它能够与 co_await 一起使用，所以函数返回类型必须是**awaitable<void>。**

**echo** 函数能够高效处理多个客户端连接而不阻塞线程，主要是因为：

- echo 函数使用 socket.async_read_some 和 async_write 方法进行**异步读写操作**。这意味着当函数执行这些操作时，它不会阻塞当前线程，而是可以在等待 I/O 完成时让出控制权。
- 使用协程和 co_await，当 I/O 操作挂起时，协程会被暂停并释放线程。这使得同一线程可以处理其他任务或更多的连接，而不需要为每个连接创建新的线程。
- 服务器的主循环（io_context.run()）会持续运行，处理所有已准备好的异步操作。这样一来，多个连接可以并发处理，而不需要多个线程同时活跃。
- 当协程通过 co_await 等待 I/O 操作时，它不会占用 CPU 资源。主线程可以继续接受新的连接或处理其他已完成的操作，从而提高并发能力。

其中

```cpp
std::size_t n = co_await socket.async_read_some(boost::asio::buffer(data), use_awaitable);
```

该段代码使用**co_await** 关键字等待异步读取操作完成，并将读取的字节数存储到n中。和之前异步服务器异步操作需要绑定回调函数不同，这里通过协程实现的并发服务器读写通过**co_await** 关键字和**use_awaitable**适配器组合使用，会自动处理异步操作的结果。当调用 **socket.async_read_some** 时，协程会**暂停执行**，并在操作完成时恢复。这个机制隐藏了回调的复杂性，使得代码更简洁和易读。当异步操作完成时，协程会自动继续执行，并将结果传递给 n 。

```cpp
co_await async_write(socket, boost::asio::buffer(data, n), use_awaitable);
```

同理，异步写函数也以同步的方式使用，不需要显示bind回调函数，**co_await** 关键字会等待异步读取操作完成，而**适配器use_awaitable**允许将异步操作的结果直接与协程的执行流结合

### ***c. listener()***

```cpp
awaitable<void> listener() {
    auto executor = co_await this_coro::executor;
    tcp::acceptor acceptor(executor, { tcp::v4(), 10086 });
    for (;;) {
        tcp::socket socket = co_await acceptor.async_accept(use_awaitable);
        co_spawn(executor, echo(std::move(socket)), detached);
    }
}
```

该函数不断监听 TCP 端口，接受来自客户端的连接。每当有新连接到达时，它会启动一个 echo 协程来处理该连接。这种设计使得服务器能够同时处理多个客户端连接而不会阻塞，提高了并发处理能力。

```cpp
auto executor = co_await this_coro::executor;
```

获取执行器：

- **this_coro::executor** 是特殊的上下文，用于获取当前协程的执行器（executor），它定义了协程将在哪个上下文（io_context）中运行
- **co_await** 关键字使得协程在获取执行器时可以暂停，并在获取到执行器后恢复执行。

```cpp
co_spawn(executor, echo(std::move(socket)), detached);
```

- 启动处理协程: 
  - **co_spawn** 启动一个新的协程
  - **executor** 指定了新的协程的执行上下文
  - **echo**(std::move(socket)) 创建一个新的 echo 协程来处理该连接。std::move(socket) 将 socket 移动到 echo 协程中，避免不必要的拷贝。移动socket之后，上面的socket便无法发挥作用，因为该socket已经被移动至echo中。
  - detached 表示新协程的执行不需要主协程等待其完成。

### ***d. main()***

```cpp
int main()
{
    try {
        boost::asio::io_context io_context(1); // 1被用于提供有关所需并发级别的提示
        boost::asio::signal_set signals(io_context, SIGINT, SIGTERM);
        signals.async_wait([&](auto, auto) { // 处理几个信号就传入几个参数，这里使用auto自动推断
            io_context.stop();
            });
        co_spawn(io_context, listener(), detached);
        io_context.run();

    }
    catch (std::exception& e) {
        std::cout << "Exception is " << e.what() << std::endl;
    }
}
```

io_context有多个重载，这里使用的重载原型为

```cpp
explicit io_context(int concurrency_hint);
```

**concurrency_hint**用来提示实现该类的系统，它应当允许多少个**线程**（不是协程）同时运行。

- concurrency_hint=0时，则I/O操作的实现将使用默认的并发级别，此时，io_context 将根据内部实现和系统资源自动决定使用多少线程；
- concurrency_hint=1时，则I/O操作的实现将尝试最小化线程的创建，并且不会创建额外的工作线程，常表示仅使用**一个线程**来处理所有 I/O 操作，适用于大多数简单的应用场景，避免不必要的线程开销；
- concurrency_hint>1时，则I/O操作的实现将允许同时运行多个工作线程，允许程序在多个线程中并行处理 I/O 操作，从而提高性能。

```cpp
        signals.async_wait([&](auto, auto) { // 处理几个信号就传入几个参数，这里使用auto自动推断
            io_context.stop();
            });
```

信号处理，当遇到退出信号（ctrl+c或强制终止信号）时，执行lambd函数，停止ioc的运行。

```cpp
co_spawn(io_context, listener(), detached);
```

启动一个listener协程，开始监听客户端连接，并且这个协程的执行不需要主协程等待其完成。

## 2. 客户端

```cpp
#include <iostream>
#include <boost/asio.hpp>

const int MAX_LENGTH = 1024;

int main()
{
    try {
        boost::asio::io_context ioc;
        boost::asio::ip::tcp::endpoint remote_ep(boost::asio::ip::address::from_string("127.0.0.1"), 10086);
        boost::asio::ip::tcp::socket sock(ioc);
        boost::system::error_code error = boost::asio::error::host_not_found;
        sock.connect(remote_ep, error);
        if (error) {
        std::cout << "connect failed, code is " << error.value() << " error msg is " << error.what() << std::endl;

        return 0;
        }

        std::cout << "Enter message: ";
        char request[MAX_LENGTH];
        std::cin.getline(request, MAX_LENGTH);
        size_t request_length = strlen(request);
        boost::asio::write(sock, boost::asio::buffer(request, request_length));

        char reply[MAX_LENGTH];
        size_t reply_length = boost::asio::read(sock, boost::asio::buffer(reply, request_length));
        std::cout << "reply is " << std::string(reply, reply_length) << std::endl;
        getchar();
    }
    catch (std::exception& e) {
        std::cerr << "Exception is " << e.what() << std::endl;
    }

    return 0;
}
```

和之前的客户端处理基本类似，只不过忽略了消息节点封装和序列号操作。

3. 修改之前的服务器函数

```cpp
void CSession::Start() {
    auto shared_this = shared_from_this();
    //开启接收协程
    co_spawn(_io_context, [=]()->awaitable<void> {
        try {
            for (;!_b_close;) {
                _recv_head_node->Clear();
                std::size_t n = co_await boost::asio::async_read(_socket,
                    boost::asio::buffer(_recv_head_node->_data, HEAD_TOTAL_LEN),
                    use_awaitable);
                if (n == 0) {
                    std::cout << "receive peer closed" << endl;
                    Close();
                    _server->ClearSession(_uuid);
                    co_return;
                }
                //获取头部MSGID数据
                short msg_id = 0;
                memcpy(&msg_id, _recv_head_node->_data, HEAD_ID_LEN);
                //网络字节序转化为本地字节序
                msg_id = boost::asio::detail::socket_ops::network_to_host_short(msg_id);
                std::cout << "msg_id is " << msg_id << endl;
                //id非法
                if (msg_id > MAX_LENGTH) {
                    std::cout << "invalid msg_id is " << msg_id << endl;
                    _server->ClearSession(_uuid);
                    co_return;
                }
                short msg_len = 0;
                memcpy(&msg_len, _recv_head_node->_data + HEAD_ID_LEN, HEAD_DATA_LEN);
                //网络字节序转化为本地字节序
                msg_len = boost::asio::detail::socket_ops::network_to_host_short(msg_len);
                std::cout << "msg_len is " << msg_len << endl;
                //长度非法
                if (msg_len > MAX_LENGTH) {
                    std::cout << "invalid data length is " << msg_len << endl;
                    _server->ClearSession(_uuid);
                    co_return;
                }
                _recv_msg_node = make_shared<RecvNode>(msg_len, msg_id);
                //读出包体
                n = co_await boost::asio::async_read(_socket,
                    boost::asio::buffer(_recv_msg_node->_data, _recv_msg_node->_total_len), use_awaitable);
                if (n == 0) {
                    std::cout << "receive peer closed" << endl;
                    Close();
                    _server->ClearSession(_uuid);
                    co_return;
                }
                _recv_msg_node->_data[_recv_msg_node->_total_len] = '\0';
                cout << "receive data is " << _recv_msg_node->_data << endl;
                //投递给逻辑线程
                LogicSystem::GetInstance().PostMsgToQue(make_shared<LogicNode>(shared_from_this(), _recv_msg_node));
            }
        }
        catch (std::exception& e) {
            std::cout << "exception is " << e.what() << endl;
            Close();
            _server->ClearSession(_uuid);
        }
        }, detached);
}
```

在新的Session中，不需要绑定回调函数进行处理，而是通过**关键字co_await 和适配器**use_awaitable，使异步函数通过同步方式写出来，在一个函数中进行数据的粘包处理、网络序列-本地序列转换、序列化处理，并将消息投递至逻辑队列。

通过协程实现并发服务器可大大减少代码量，相比异步编程更加直观，但受限于平台，目前C++20的协程说是协程库，实际上只是开放了无栈协程的协议，正儿八经的官方协程还未发布。
