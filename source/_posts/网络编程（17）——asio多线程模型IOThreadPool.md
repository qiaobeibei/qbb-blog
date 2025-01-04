---
title: 网络编程（17）——asio多线程模型IOThreadPool
date: 2024-11-03 11:45:53
categories:
- C++
- 网络编程
tags: 
- IOThreadPool
- 多线程模式
- 线程池
typora-root-url: ./..
---

# 十七、day17

之前我们介绍了**IOServicePool**的方式，一个IOServicePool开启n个线程和n个iocontext，每个线程内独立运行iocontext, 各个iocontext监听各自绑定的socket是否就绪，如果就绪就在各自线程里触发回调函数。为避免线程安全问题，我们将网络数据封装为逻辑包投递给逻辑系统，逻辑系统有一个单独线程处理，这样将网络IO和逻辑处理解耦合，极大的提高了服务器IO层面的吞吐率。

今天给大家介绍asio多线程模式的第二种多线程模式**IOThreadPool**，我们只初始化一个iocontext用来监听服务器的读写事件，包括新连接到来的监听也用这个iocontext。只是我们让iocontext.run在多个线程中调用，启动一个 io_context，由多个线程共享，这样回调函数就会被不同的线程触发，从这个角度看回调函数被并发调用了。

![img](/images/$%7Bfiilename%7D/format,png-1730605565645-66.png)

# 1. IOThreadPool实现

## 1) IOThreadPool.h

```cpp
#pragma once
#include "Singleton.h"
#include <boost/asio.hpp>

class AsioThreadPool : public Singleton<AsioThreadPool>
{
public:
	friend class Singleton<AsioThreadPool>;
	~AsioThreadPool() {}
	AsioThreadPool& operator = (const AsioThreadPool&) = delete;
	AsioThreadPool(const AsioThreadPool&) = delete;

	boost::asio::io_context& GetIOService();
	void Stop();
private:
	AsioThreadPool(int threadNum = std::thread::hardware_concurrency());
	boost::asio::io_context _service;
	std::unique_ptr<boost::asio::io_context::work> _work;
	std::vector<std::thread> _threads;
};
```

- **AsioThreadPool**也是单例模式，需继承单例模板类Singleton<T>
- **_service**：AsioThreadPool多线程模式中，只有一个io_context进行不同线程的调度，所以仅初始化一个ioc
- **_work**：work的数量与ioc相同，有且仅有一个
- **_threads**：线程数，默认与cpu核数相同

AsioThreadPool继承 Singleton 模板类时，Singleton 类的拷贝构造函数和赋值运算符已经被删除，这意味着AsioThreadPool也会继承这些删除操作。因此，派生类默认不能被拷贝或赋值，也不需要再次delte拷贝或者赋值，但是可以写上增加代码阅读性。

## 2）AsioThreadPool构造函数

```cpp
AsioThreadPool::AsioThreadPool(int threadNum) : _work(new boost::asio::io_context::work(_service)){
	for (int i = 0; i < threadNum; ++i) {
		_threads.emplace_back([this]() {
			_service.run();
			});
	}
}
```

注意：work是通过std::unique_ptr进行管理的，所以work不能被拷贝，仅仅可以通过移动语义转移。

将AsioThreadPool池中唯一的ioc与work绑定，防止ioc返回，然后启动**threadNum**个线程，每个线程中都调用**_service.run()。**

**_service.run**函数内部就是从iocp或者epoll获取就绪描述符和绑定的**回调函数**，进而调用回调函数，因为回调函数是在不同的线程里调用的，所以会存在**不同的线程调用同一个socket的回调函数的情况。**

_service.run内部在Linux环境下调用的是epoll_wait返回所有就绪的描述符列表，在windows上会循环调用GetQueuedCompletionStatus函数返回就绪的描述符，二者原理类似，进而通过描述符找到对应的注册的回调函数，然后调用回调函数。

二者的流程如下：

***a. IOCP**

- IOCP的使用主要分为以下几步：
   1 创建完成端口(iocp)对象
   2 创建一个或多个工作线程，在完成端口上执行并处理投递到完成端口上的I/O请求
   3 Socket关联iocp对象，在Socket上投递网络事件
   4 工作线程调用GetQueuedCompletionStatus函数获取完成通知封包，取得事件信息并进行处理。该函数会阻塞，直到有I/O操作完成并返回相应的事件信息，线程接收到通知后就可以处理对应的I/O请求。

***b. epoll**

- 调用epoll_creat在内核中创建一张epoll表，这张 epoll 表用于管理要监视的文件描述符
- 开辟一片包含n个epoll_event大小的连续空间，这个空间的大小是预期要监听的事件数量 n，以便存放准备好的事件信息。
- 将要监听的socket（或其他文件描述符）注册到epoll表里，可以指定需要监听的事件类型（如可读、可写等），并将其与之前创建的 epoll 实例关联。
- 调用epoll_wait，传入之前我们开辟的连续空间，这一调用会阻塞，直到有事件发生。一旦有事件就绪，epoll_wait 会返回就绪的 epoll_event 列表，并将对应的 Socket 信息写入之前开辟的内存空间中，供后续处理。

## 3）其他函数

```cpp
boost::asio::io_context& AsioThreadPool::GetIOService() {
	return _service;
}

void AsioThreadPool::Stop() {
	_work.reset();
	for (auto& t : _threads) {
		t.join(); 
	}
}
```

- GetIOService()用于返回多线程池中唯一的上下文服务
- Stop()用于将work释放，以便ioc可以返回

# 2. 隐患及如何解决

IOThreadPool模式有一个**隐患**，同一个socket的就绪后，触发的回调函数可能在不同的线程里，比如第一次是在线程1，第二次是在线程3，如果这两次触发间隔时间不大，那么很可能出现不同线程并发访问数据的情况，比如在处理读事件时，第一次回调触发后我们从socket的接收缓冲区读数据出来，第二次回调触发,还是从socket的接收缓冲区读数据，就会造成两个线程同时从socket中读数据的情况，会造成数据混乱。因为多个线程都会执行io_context.run()，每次调用都会返回一些就绪的回调函数，比如，线程1调用io_context.run()返回ReadHandler1以及绑定的socket，线程2调用io_context.run()返回ReadHandler2以及绑定的socket，线程3调用io_context.run()返回ReadHandler3以及绑定的socket，但是ReadHandler1和ReadHandler3都是session1调用的，那么可能会导致多个线程同时处理相同的回调，导致线程安全问题

**注**：**但如果回调函数被派发到逻辑队列中或者进行加锁，那么就不会存在不同线程访问同一个socket数据的线程安全问题。这里的隐患是不通过队列或者加锁，处理逻辑问题会存在线程安全问题。**

## **1）通过strand改进**

在多线程环境中触发回调函数时，我们可以使用 asio 提供的串行类 strand 来进行封装，从而实现串行调用。其基本原理是：**每个线程在调用函数时，不直接执行该函数，而是将要调用的函数投递到由 strand 管理的队列中；随后，由一个统一的线程从队列中取出并调用这些回调函数。**通过这种方式，函数调用是串行进行的，从而解决由于线程并发带来的安全问题。



![img](/images/$%7Bfiilename%7D/format,png-1730605565646-67.png)

<center>使用strand进行封装</center>

图中当socket就绪后并不是由多个线程调用每个socket注册的回调函数，而是将回调函数投递给strand管理的队列，再由strand统一调度派发。为了让回调函数被派发到strand的队列，我们只需要在注册回调函数时加一层strand的包装即可。

strand 实际上是将回调函数放入**同一个队列（如果有逻辑层或者通过加锁处理回调，那么就不需要strand）**，以确保线程安全。因此，在描述符就绪后，相应的回调会被放入该队列中进行处理。虽然使用 strand 可以保证在多线程环境下的安全性，但在触发时仍然是单线程的，这限制了整体性能。因此，尽管多线程可以提高读写事件的派发效率，但整体性能通常不**如IOService_pool** 这种方式更优。在实际工作中，我们通常选择使用 **IOService_pool**，因为它能够更好地平衡并发和性能。

### ***a. CSession类中新增成员变量_strand**

```cpp
boost::asio::strand<boost::asio::io_context::executor_type> _strand;
```

### ***b. 修改CSession的构造函数**

```cpp
CSession::CSession(boost::asio::io_context& io_context, CServer* server):
    _socket(io_context), _server(server), _b_close(false),
    _b_head_parse(false), _strand(io_context.get_executor()){
    boost::uuids::uuid  a_uuid = boost::uuids::random_generator()();
    _uuid = boost::uuids::to_string(a_uuid);
    _recv_head_node = make_shared<MsgNode>(HEAD_TOTAL_LEN);
}
```

可以看到**_strand**的初始化是放在初始化列表里，利用**io_context.get_executor()**返回的执行器构造strand。因为在asio中无论iocontext还是strand，底层都是通过executor调度的，我们将他理解为调度器就可以。如果多个iocontext和strand的调度器是一个，那他们的消息派发统一由这个调度器执行。我们利用iocontext的调度器构造strand，这样他们统一由一个调度器管理。在绑定回调函数的调度器时，我们选择strand绑定即可。

比如我们在**Start**函数里添加绑定 ，将回调函数的调用者绑定为**_strand**

```cpp
void CSession::Start(){
    ::memset(_data, 0, MAX_LENGTH);
    _socket.async_read_some(boost::asio::buffer(_data, MAX_LENGTH),
        boost::asio::bind_executor(_strand, std::bind(&CSession::HandleRead, this,
            std::placeholders::_1, std::placeholders::_2, SharedSelf())));
}
```

修改前的代码为：

```cpp
void CSession::Start() {
	memset(_data, 0, MAX_LENGTH); // 缓冲区清零
		// 从套接字中读取数据，并绑定回调函数headle_read
	_socket.async_read_some(boost::asio::buffer(_data, MAX_LENGTH),
		std::bind(&CSession::headle_read, this, std::placeholders::_1, std::placeholders::_2,
			shared_from_this()));
	//_socket.async_read_some(boost::asio::buffer(_data, MAX_LENGTH),
	//	boost::asio::bind_executor(_strand, 
	//		std::bind(&CSession::headle_read, this, std::placeholders::_1, std::placeholders::_2,shared_from_this())));
}
```



同样的道理，在所有收发的地方，都将调度器绑定为**_strand**， 比如发送部分我们需要修改为如下

```cpp
    auto& msgnode = _send_que.front();
    boost::asio::async_write(_socket, boost::asio::buffer(msgnode->_data, msgnode->_total_len), 
    boost::asio::bind_executor(_strand, std::bind(&CSession::HandleWrite, this, std::placeholders::_1, SharedSelf()))
        );
```

# 3. 客户端

```cpp
int main()
{
    try {
        auto pool = AsioThreadPool::GetInstance();
        boost::asio::io_context ioc;
        boost::asio::signal_set signals(ioc, SIGINT, SIGTERM);
        // 必须异步等待，否则建立线程进行处理
        signals.async_wait([&ioc, pool](const boost::system::error_code& error, int signal_number) {
            if (!error) {
                std::cout << "Signal " << signal_number << " received." << std::endl;
                ioc.stop();  // 停止 io_context
                pool->Stop();
                std::unique_lock<std::mutex> lock(mutex_quit);
                bstop = true;
                cond_quit.notify_one();
            }
            });

        CServer s(pool->GetIOService(), 10086);
        ioc.run();
        {
            std::unique_lock<std::mutex> lock(mutex_quit);
            while (!bstop) {
                cond_quit.wait(lock);
            }
        }
    }
    catch (std::exception& e) {
        std::cerr << "Exception: " << e.what() << '\n';
    }
    boost::asio::io_context io_context;
}
```

之所以用条件变量是因为传入Server的ioc是从线程池中获得的，Server初始化之后并没有调用ioc.run()，线程池中的ioc是运行在自己的线程中的，下面这段是运行在主线程中的，是为了防止主线程结束了而子线程还在运行，所以必须设置条件变量防止主线程结束

```cpp
        {
            std::unique_lock<std::mutex> lock(mutex_quit);
            while (!bstop) {
                cond_quit.wait(lock);
            }
        }
```

如果不设置条件变量使主线程挂起，那么当主线程结束后，如果server还没有跑起来，server相当于子线程，那么会直接让子线程结束

```cpp
CServer s(pool->GetIOService(), 10086);
```

单独生成一个ioc运行监听信号，当收到两个退出信号时，服务器退出，而server的ioc是从线程池中获取的。当获取到退出信号时，首先将监听信号的ioc退出，然后将线程池的unique_ptr释放work，使得线程池中的ioc能退出，最后上锁然后唤醒主线程，主线程结束。

```cpp
                std::cout << "Signal " << signal_number << " received." << std::endl;
                ioc.stop();  // 停止 io_context
                pool->Stop();
                std::unique_lock<std::mutex> lock(mutex_quit);
                bstop = true;
                cond_quit.notify_one();
```

# 4. 总结

***1. 在IOServicePool示例中，使用main函数中定义的io_context来监听异常信号signal，并且用于初始化Server来监听接收信号；但在IOThreadPool示例中，main函数定义的ioc仅仅用于监听异常信号，而初始化Server启动异步接收async_accept是通过线程池中的ioc。**

这是因为

第一种方式调用了n+1个ioc.run()，第二种方式调用了2个ioc.run()

- **前者**在main函数中处理信号监听和Server任务，当main中的ioc接收到客户端连接之后，生成一个子session，并从IOServicePool线程池中取一个ioc用于管理子Session的收发。也就是说，main中的ioc用于监听信号和server任务处理，而线程池中的ioc用于处理子Session的收发。main中的ioc一旦停止，那么服务器就会停止客户端的连接服务，session任务不会在增加，而线程池中的ioc全部停止后，线程的收发就会停止。
- **后者**再main函数中处理监听信号任务，而server任务处理交给了线程池IOThreadPool中唯一的ioc进行处理。也就是说，让一个ioc单独监听信号，另一个ioc运行在多线程中做收发。main中的ioc停止后，仅仅只是服务器的信号监听停止了，而server的任务处理以及session的收发仍然会持续。

此外，他们的退出机制也有些不同。

- **第一种方式**：信号处理函数在 `ioc` 上注册，当收到信号时，会调用 `ioc.stop()` 停止 `io_context`，并调用线程池的 `Stop` 方法停止线程池的运行。这种方式不需要额外的同步机制，因为主线程就是 `ioc.run()` 的执行线程，一旦停止，就会退出。
- **第二种方式**：信号处理函数在主线程的 `ioc` 上注册，但实际处理的 I/O 事件由线程池中的 `io_context` 完成。为了协调多线程退出，需要使用 `mutex` 和条件变量（`cond_quit`）来确保所有线程安全地退出。信号处理函数在停止 `io_context` 后，会通知等待在条件变量上的线程退出，从而确保整个程序能够正确结束。

***2. 哪一种多线程模式的效率最高？**

直观上来说可能二者的效率都差不多？其实IOServicePool的效率略高于IOThreadPool。

**IOThreadPool** 通常会涉及到 **strand**，**strand** 的作用是将回调函数放入**同一个队列中**执行，以确保线程安全（**如果逻辑层或者通过加锁已经处理了回调的同步问题，那么就不需要使用 strand**）。在这种机制下，当描述符就绪时，相应的回调会被放入这个队列中依次处理。虽然 strand 可以确保多线程环境下的安全性，但由于回调函数的执行是串行的，所以在事件触发时仍然是单线程操作，这在一定程度上限制了整体性能的提升。

尽管多线程模式确实能提高读写事件的派发效率，但由于 strand 的串行限制，整体性能通常不如使用 **IOService_pool** 的方式。在实际工作中，我们更倾向于使用 **IOService_pool**，因为它在保证线程安全的同时，能够更好地利用并发性，提升程序性能和效率。
