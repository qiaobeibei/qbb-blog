---
title: 网络编程（16）——asio多线程模型IOServicePool
date: 2024-11-03 11:41:51
categories:
- 网络编程
tags: 
- IOServicePool
- 多线程
- 线程池
typora-root-url: ./..
---

# 十六、day16

在之前的设计中，我们对 ASIO 的使用都是采用单线程模式。为了提升网络 I/O 并发处理的效率，这次我们设计了在多线程模式下使用 ASIO 的方法。总体而言，ASIO 有**两种多线程模型**：

- 启动多个线程，每个线程管理一个独立的 io_context。
- 启动一个 io_context，由多个线程共享。

在后续文章中，我们会对比这两种模式的区别。这里我们先介绍第一种模式，即多个线程，**每个线程管理一个独立的 io_context 服务。**

## 1. 什么是多线程？

之前在完善消息节点的章节学习过asio服务器底层通信的流程，它是基于单线程运行的，可参考

[知乎用户www.zhihu.com/people/zhi-chi-tian-ya-10-23/posts](https://www.zhihu.com/people/zhi-chi-tian-ya-10-23/posts)

![img](/images/$%7Bfiilename%7D/9144ff5d10454a66a46c0e576f6db398.png)编辑

<center>单线程模式</center>

而今天将设计IOServicePool类型的多线程模型，如下图所示

![img](/images/$%7Bfiilename%7D/1eb8c8c7e0024a6a86f659ba77d54e78.png)编辑

<center>多线程模式</center>

IOServicePool 服务池中，IOServicePool 类会根据系统的 CPU 核数创建相应数量的 io_context 实例，并将每个 io_context 运行在一个独立的线程中。例如，如果系统有两个 CPU 核，就会有两个独立的线程分别运行各自的 io_context。io_context 是一个调度器，用于管理异步事件。例如，对于 Session1 会话，如果想在线程 1 上注册一个读事件，可以通过 async_read 将读事件注册到 io_context1 中，这样它的回调函数就会在线程 1 中执行。同样，线程 2 也是独立运行的，并处理它对应的 io_context 的事件。

**IOServicePool多线程模式特点：**

- 每个 `io_context` 都在独立的线程中运行，因此同一个 socket 会被注册在同一个 `io_context` 上，它的回调函数也会在同一个线程中执行。这样，对于同一个 socket 来说，每次回调函数触发都会在同一个线程中执行，从而避免了线程安全问题，确保网络 I/O 层面的并发是线程安全的。
- 但是，对于不同的 socket，回调函数的触发可能会在同一个线程中（如果两个 socket 被分配到同一个 `io_context`），也可能在不同的线程中（如果两个 socket 分配到不同的 `io_context`）。多个socket由同一个ioc调度的话，不会发生逻辑安全或线程问题，但如果不同的socket由不同的ioc调度，那么可能会发生安全问题。比如，两个 socket 对应的上层逻辑有交互或共享数据，就可能存在线程安全问题。如果 socket1 代表玩家1，socket2 代表玩家2，而这两个玩家在逻辑层面上有交互（如同属一个工会并且共同完成任务），则涉及的工会积分是共享的数据区域，需要保证线程安全。可以通过**加锁**或使用**逻辑队列**来解决这个问题，目前我们采用的是**逻辑队列**的方法。
- 与单线程相比，多线程显著提高了**并发能力**。在单线程模式下，只有一个 `io_context` 来监听读写事件，事件就绪后回调函数在同一个线程中串行执行，如果一个回调函数执行时间较长，会影响后续的回调函数。而在多线程模式下，可以在一定程度上减少一个逻辑调用对下一个调用的影响。例如，如果两个 socket 被分配到不同的 `io_context` 上，它们的回调就不会互相影响。但如果两个 socket 分配到同一个 `io_context`，仍然可能有调用时间的影响。不过，我们已经**通过逻辑队列的方式将网络线程和逻辑线程解耦，从而避免了前一个调用时间影响下一个回调触发的问题。**

## **2. IOServicePool实现**

IOServicePool本质上是一个线程池，基本功能就是**根据构造函数传入的数量创建n个线程和iocontext，然后每个线程跑一个iocontext**，这样就可以并发处理不同iocontext读写事件了

### ***a. IOServicePool.h\***

```cpp
#pragma once
#include "Singleton.h"
#include <boost/asio.hpp>
#include <vector>

class AsioIOServicePool : public Singleton<AsioIOServicePool>
{
	friend Singleton<AsioIOServicePool>;
public:
	using IOService = boost::asio::io_context;
	using Work = boost::asio::io_context::work; // work的作用？
	using WorkPtr = std::unique_ptr<Work>; // 希望该work不会被拷贝，只能移动或者从头用到尾不被改变
	~AsioIOServicePool();
	AsioIOServicePool(const AsioIOServicePool&) = delete;
	AsioIOServicePool& operator = (const AsioIOServicePool&) = delete;
	// 使用round-robin 的方式返回一个io_context
	boost::asio::io_context& GetIOService();
	void Stop();
private:
	AsioIOServicePool(std::size_t size = std::thread::hardware_concurrency()); // hardware_concurrency获取CPu核数
	std::vector<IOService> _ioServices;
	std::vector<WorkPtr> _works;
	std::vector<std::thread> _threads;
	// 通过轮询返回ioc时，需要记录当前ioc的下标，累加，当超过vector的size时就归零，然后继续按轮询的方式返回
	// 记录ioc在vector的下标
	std::size_t _nextIOService;
};
```

- IOServicePool也是单例模式，有且仅有唯一实例
- IOService ：io_context
- Work ：用于绑定ioc，避免ioc.run()提前返回, work的详细作用请看文章末的总结部分
- WorkPtr ：使用unique_ptr管理work，希望该work不会被拷贝，只能移动或者从头用到尾不被改变
- _ioServices：存储指定数量的ioc
- _works：存储与ioc数量对应的work
- _threads：存储指定数量的线程
- _nextIOService：记录ioc在vector的下标，通过轮询返回ioc时，需要记录当前ioc的下标，累加，当超过vector的size时就归零，然后继续按轮询的方式返回

### **b. \*IOServicePool构造函数\***

```cpp
AsioIOServicePool::AsioIOServicePool(std::size_t size) : _ioServices(size), _works(size), _nextIOService(0) {
	for (std::size_t i = 0; i < size; i++) {
		_works[i] = std::unique_ptr<Work>(new Work(_ioServices[i]));
	}

	// 遍历多个ioservice，创建多个线程，每个线程内部启动ioservice
	for (std::size_t i = 0; i < _ioServices.size(); i++) {
		_threads.emplace_back([this, i]() {
			_ioServices[i].run();
			});
	}
}
```

size的默认值是std::thread::hardware_concurrency()，该函数用于获取CPU核数。如果不主动更改size，那么**IOServicePool**会构造数量等于CPU核数的上下文服务、work和线程。

因为work通过std::unique_ptr进行管理，所以下面这段代码是错的，因为std::unique_ptr 不允许将一个普通指针直接赋值给另一个 std::unique_ptr， std::unique_ptr是独占有权的。

```cpp
auto unptr = std::unique_ptr<Work>(new Work(_ioServices[i]));
_works[i] = unptr;
```

但是，可以通过**移动语义**将自动将创建的 unique_ptr 的所有权转移到 _works[i] ，实际上是在 _works[i] 中创建或替换一个 unique_ptr。

std::unique_ptr 不允许复制（即同一对象不能被多个 unique_ptr 同时拥有），但支持移动操作。使用 std::unique_ptr 时，_works[i] 直接接收新创建的 unique_ptr，所有权被有效地转移。

可以将unique_ptr作为**右值**赋值给另一个unique_ptr

```cpp
_works[i] = std::unique_ptr<Work>(new Work(_ioServices[i]));
```

注意，**Work(ioc_context)**是asio库的函数，用于将work与ioc进行绑定，避免ioc.run()返回。原型为

```cpp
boost::asio::io_context::work::work(boost::asio::io_context& io_context)
```

最后，遍历多个ioservice，创建多个线程，每个线程内部启动ioservice。

***c. GetIOService()\***

```cpp
boost::asio::io_context& AsioIOServicePool::GetIOService() {
	auto& service = _ioServices[_nextIOService++];
	if (_nextIOService == _ioServices.size())
		_nextIOService = 0;

	return service;
}
```

该段代码用于从ioc存储容器_ioServices中获取io_context&，其中_nextIOService为索引，轮询获取io_context&

### ***d. Stop()\***

```cpp
void AsioIOServicePool::Stop(){
    for (auto& work : _works) {
        work.reset();
    }
    for (auto& t : _threads) {
        t.join();
    }
}
```



同样我们要实现Stop函数，控制AsioIOServicePool停止所有ioc的工作，并等待所有线程结束。因为我们要保证每个线程安全退出后再让AsioIOServicePool停止。

## 3. 服务器修改

### ***a. void CServer::start_accept()\***

```cpp
void CServer::start_accept() {
	auto& ioc = AsioIOServicePool::GetInstance()->GetIOService();
	// make_shared分配并构造一个 std::shared_ptr,_ioc, this是传给Session的参数
	std::shared_ptr<CSession> new_session = std::make_shared<CSession>(ioc, this);
	// 开始一个异步接受操作，当new_session的socket与客户端连接成功时，调用回调函数handle_accept
	_acceptor.async_accept(new_session->Socket(), std::bind(&CServer::handle_accept, this, new_session,
		std::placeholders::_1));
}
```

该段函数在CServer实例化的时候被CServer构造函数调用，使服务器启动异步接收（相当于异步读，之前的代码不需要work的原因是因为ioc.run()是在CServer实例化后运行的，start_accept()函数会执行异步接收操作，相当于异步读注册给ioc，ioc.run不会返回），等待客户端连接。

**之前的代码中，new_session使用的ioc是acceptor绑定的ioc，该ioc负责异步接收、异步读和写。但是在多线程模式中，该ioc只需要执行异步接收操作，而异步读写通过从AsioIOServicePool池中获取的ioc运行。**

```cpp
std::shared_ptr<CSession> new_session = std::make_shared<CSession>(_ioc, this); // 修改前
std::shared_ptr<CSession> new_session = std::make_shared<CSession>(ioc, this); // 修改后
```

### ***b. AsyncServer_MsgNode.cpp\***

主函数也需要修改，因为现在的ioc不止用于执行异步接受，还有线程池中的ioc，所以需要将二者均stop

```cpp
int main()
{
    try {
        auto pool = AsioIOServicePool::GetInstance();
        boost::asio::io_context ioc;
        boost::asio::signal_set signals(ioc, SIGINT, SIGTERM);
        // 必须异步等待，否则建立线程进行处理
        signals.async_wait([&ioc, pool](const boost::system::error_code& error, int signal_number) {
            if (!error) {
                std::cout << "Signal " << signal_number << " received." << std::endl;
                ioc.stop();  // 停止 io_context
                pool->Stop();
            }
            });

        CServer s(ioc, 10086);
        ioc.run();
    }
    catch (std::exception& e) {
        std::cerr << "Exception: " << e.what() << '\n';
    }
    boost::asio::io_context io_context;
}
```

## 4. 客户端修改

```cpp
#include <boost/asio.hpp>
#include <iostream>
#include <json/json.h>
#include <json/value.h>
#include <json/reader.h>
#include <chrono>
#include <thread>

using namespace boost::asio::ip;
using std::cout;
using std::endl;
const int MAX_LENGTH = 1024 * 2; // 发送和接收的长度为1024 * 2字节
const int HEAD_LENGTH = 2;
const int HEAD_TOTAL = 4;

std::vector<std::thread> vec_threads;

int main()
{
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < 50; i++) { //建立100个线程
        vec_threads.emplace_back([]() {
            try {
                boost::asio::io_context ioc; // 创建上下文服务
                // 127.0.0.1是本机的回路地址，也就是服务器和客户端在一个机器上
                tcp::endpoint remote_ep(address::from_string("127.0.0.1"), 10086); // 构造endpoint
                tcp::socket sock(ioc);
                boost::system::error_code error = boost::asio::error::host_not_found; // 错误：主机未找到
                sock.connect(remote_ep, error);
                if (error) {
                    cout << "connect failed, code is " << error.value() << " error msg is " << error.message();
                    return 0;
                }
                int i = 0;
                while(i++ < 200) {
                    Json::Value root;
                    root["id"] = 1001;
                    root["data"] = "hello world";
                    std::string request = root.toStyledString();
                    size_t request_length = request.length();
                    char send_data[MAX_LENGTH] = { 0 };
                    int msgid = 1001;
                    int msgid_host = boost::asio::detail::socket_ops::host_to_network_short(msgid);
                    memcpy(send_data, &msgid_host, 2);

                    //转为网络字节序
                    int request_host_length = boost::asio::detail::socket_ops::host_to_network_short(request_length);
                    memcpy(send_data + 2, &request_host_length, 2);
                    memcpy(send_data + 4, request.c_str(), request_length);
                    boost::asio::write(sock, boost::asio::buffer(send_data, request_length + 4));

                    char reply_head[HEAD_TOTAL]; // 首先读取对端发送消息的总长度
                    size_t reply_length = boost::asio::read(sock, boost::asio::buffer(reply_head, HEAD_TOTAL));
                    msgid = 0;
                    memcpy(&msgid, reply_head, HEAD_LENGTH);
                    short msglen = 0; // 消息总长度
                    memcpy(&msglen, reply_head + 2, HEAD_LENGTH); // 将消息总长度赋值给msglen
                    //转为本地字节序
                    msglen = boost::asio::detail::socket_ops::network_to_host_short(msglen);
                    char msg[MAX_LENGTH] = { 0 }; // 构建消息体（不含消息总长度）
                    size_t msg_length = boost::asio::read(sock, boost::asio::buffer(msg, msglen));

                    Json::Reader reader;
                    reader.parse(std::string(msg, msg_length), root);
                    std::cout << "msg id is " << root["id"] << " msg is " << root["data"] << endl;
                
                }
            }
            catch (std::exception& e) {
                std::cerr << "Exception: " << e.what() << endl;
            }
            });
        std::this_thread::sleep_for(std::chrono::seconds(1));

        for (auto& t : vec_threads) {
            t.join();
        }
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        cout << "Time spent: " << duration.count() << " microsencods" << endl;

    }
    return 0;
}
```

## 5. 总结

**1. boost::asio::io_context::work的作用？**

在实际使用中，我们通常会将一些异步操作提交给io_context进行处理，然后该操作会被异步执行，而不会立即返回结果。如果没有其他任务需要执行，那么io_context就会停止工作，导致所有正在进行的异步操作都被取消。这时，我们需要使用**boost::asio::io_context::work**对象来防止io_context停止工作。

**boost::asio::io_context::work**的作用是**持有一个指向io_context的引用，并通过创建一个“工作”项来保证io_context不会停止工作，直到work对象被销毁或者调用reset()方法为止**。当所有异步操作完成后，程序可以使用**work.reset()方法来释放io_context，从而让其正常退出。**

在之前的代码中，ioc不会被阻塞是因为我们已经提前给ioc注册了一个读事件（acceptor通过async_accept注册了一个读事件监听对端连接，而acceptor又绑定了此io_context），所以此时的ioc不会退出。

```cpp
        boost::asio::io_context ioc;
        boost::asio::signal_set signals(ioc, SIGINT, SIGTERM);
        // 必须异步等待，否则建立线程进行处理
        signals.async_wait([&ioc](const boost::system::error_code& error, int signal_number) {
            if (!error) {
                std::cout << "Signal " << signal_number << " received." << std::endl;
                ioc.stop();  // 停止 io_context
            }
            });

        CServer s(ioc, 10086);
        ioc.run();

CServer::CServer(boost::asio::io_context& ioc, short port) : _ioc(ioc),
_acceptor(ioc, tcp::endpoint(tcp::v4(), port)) {
	cout << "Server start success, on port: " << port << endl;
	// 开始异步地接受客户端连接请求。服务器启动后就进入等待客户端连接的状态
	start_accept();
}

void CServer::start_accept() {
	// make_shared分配并构造一个 std::shared_ptr,_ioc, this是传给Session的参数
	std::shared_ptr<CSession> new_session = std::make_shared<CSession>(_ioc, this);
	// 开始一个异步接受操作，当new_session的socket与客户端连接成功时，调用回调函数handle_accept
	_acceptor.async_accept(new_session->Socket(), std::bind(&CServer::handle_accept, this, new_session,
		std::placeholders::_1));
}
```

而我们实现的IOServicePool中，在它的构造函数中初始化了n个io_context，且ioc运行在独立的线程中调用ioc.run()，如果不写work，相当于ioc没有绑定任何事件，那么ioc就会退出
