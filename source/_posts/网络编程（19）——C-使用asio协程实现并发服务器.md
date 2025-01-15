---
title: 网络编程（19）——C++使用asio协程实现并发服务器
date: 2024-11-03 11:49:29
categories:
- C++
- 网络编程
tags: 
- 协程
- 单例模式
- 线程池
- 逻辑层
- 字节序
typora-root-url: ./..
---

##  十九、day19

上一节学习了如果通过asio协程实现一个简单的并发服务器demo（官方案例），今天学习如何通过asio协程搭建一个比较完整的并发服务器。

主要实现了AsioIOServicePool线程池、逻辑层LogicSystem、粘包处理、接收协程、发送队列、网络序列化、json序列化、通过两种方式实现单例模式（第一种方式实现线程池AsioIOServicePool，第二种方式实现逻辑层LogicSystem）。

- Const文件主要用于定义各种宏
- CServer主要用于监听客户端的连接，并生成并分配Session会话
- AsioIOServicePool线程池用于多线程收发
- LogicSystem逻辑层用于处理逻辑队列中的数据
- CSession用于服务器与客户端的收发操作

视频资料参考up恋恋风辰：

[C++ 网络编程(19) 利用协程实现并发服务器(上)_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV18k4y1M7Nx/?vd_source=cb95e3058c2624d2641da6f4eeb7e3a1)

------

## 1. Const.h

```cpp
#pragma once
#define MAX_LENGTH 1024*2
#define HEAD_TOTAL_LEN 4
#define HEAD_ID_LEN 2
#define HEAD_DATA_LEN 2
#define MAX_RECVQUE 1000
#define MAX_SENDQUE 1000

enum MSG_ID {
	MSG_HELLO_WORD = 1001
};
```

该头文件用于定义消息收发中需要用到的宏

- **MAX_LENGTH**：一次最多接收2048字节的内容
- **HEAD_TOTAL_LEN**：消息头结点总长度为4字节，包括消息id和消息体长度
- **HEAD_ID_LEN**：消息id，2字节
- **HEAD_DATA_LEN**：消息长度，2字节
- **MAX_RECVQUE**：接收队列最多容纳1000个处理请求
- **MAX_SENDQUE**：发送队列最多容纳1000个处理请求
- **MSG_ID**：用于存储消息id，通过id在逻辑队列中查找对应的回调函数

## 2. 消息存储节点MsgNode

### a. MsgNode.h

```cpp
#pragma once
#include <string>
#include "Const.h"
#include <iostream>
#include <boost/asio.hpp>

class MsgNode
{
public:
	short _cur_len;
	short _total_len;
	char* _data;

	MsgNode(short max_len) : _total_len(max_len), _cur_len(0) {
		_data = new char[_total_len + 1]();
		_data[_total_len] = '\0';
	}
	~MsgNode() {
		std::cout << "destruct MsgNode" << std::endl;
		delete[] _data;
	}
	void Clear() {
		::memset(_data, 0, _total_len);
		_cur_len = 0;
	}
};

class RecvNode : public MsgNode {
private:
	short _msg_id;
public:
	RecvNode(short max_len, short msg_id);
	const short& GetMsgID() { return _msg_id; }
};

class SendNode : public MsgNode {
private:
	short _msg_id;
public:
	SendNode(const char* msg, short max_len, short msg_id);
};
```

MsgNode类主要用于构建消息接收节点和消息发送节点

- RecvNode节点作为接收节点，构造函数仅需最大长度和消息id，同时定义一个GetMsgID()函数用于返回消息id（在逻辑层根据消息id寻找对应已注册回调函数时需要用到）
- SendNode节点作为发送节点，不仅需要知道最大长度和消息id，还需传入消息首地址msg

### b. 成员函数实现

```cpp
#include "MsgNode.h"

RecvNode::RecvNode(short max_len, short msg_id) : MsgNode(max_len), _msg_id(msg_id){
}

SendNode::SendNode(const char* msg, short max_len, short msg_id) :
	MsgNode(max_len + HEAD_TOTAL_LEN), _msg_id(msg_id) {
	// 先发送id，并转换为网络字节序
	short msg_id_net = boost::asio::detail::socket_ops::host_to_network_short(msg_id);
	memcpy(_data, &msg_id_net, HEAD_ID_LEN);
	// 发送消息长度，转换为网络字节序
	short max_len_net = boost::asio::detail::socket_ops::host_to_network_short(max_len);
	memcpy(_data + HEAD_ID_LEN, &max_len_net, HEAD_DATA_LEN);
	memcpy(_data + HEAD_TOTAL_LEN, msg, max_len);
}
```

- RecvNode节点的构造函数比较简单，只需要调用基类MsgNode的构造函数并将消息id传入_msg_id
- SendNode不仅需要上述两步（基类MsgNode构造函数传入的最大长度不是传入的值，还需要再传入值基础上加4字节，4字节用于存储消息id和消息体长度），还需进行以下操作： 
  - 将传入的消息id转换为网络字节序并存储至_data（MsgNode类成员变量），长度2字节
  - 将传入的消息长度转换为网络字节序并存储至_data（MsgNode类成员变量），长度2字节
  - _data偏移4字节后存储消息体内容，长度为传入值max_len

因为不同设备的字节序存在大端序和小端序的问题，必须将消息id和消息长度转换为网络序，否则消息内容可能会紊乱，网络字节序的内容可参考

[爱吃土豆：网络编程（8）+字节序处理0 赞同 · 0 评论文章](https://zhuanlan.zhihu.com/p/721185318)

## 3. CServer

### a. CServer.h

```cpp
#pragma once
#include <memory>
#include <map>
#include <mutex>
#include "boost/asio.hpp"
#include "CSession.h"
#include "AsioIOServicePool.h"
#include <boost/uuid/uuid_generators.hpp>
#include <boost/uuid/uuid_io.hpp>
#include <boost/asio/co_composed.hpp>
#include <boost/asio/detached.hpp>
#include "Const.h"
#include <queue>
#include <mutex>
#include <memory>
#include "MsgNode.h"

using boost::asio::ip::tcp;
using boost::asio::awaitable;
using boost::asio::co_spawn;
using boost::asio::detached;
using boost::asio::use_awaitable;
namespace this_coro = boost::asio::this_coro;

class CServer
{
private:
	void HandleAccept(std::shared_ptr<CSession>, const boost::system::error_code& error);
	void StartAccept();
	boost::asio::io_context& _io_context;
	short _port;
	boost::asio::ip::tcp::acceptor _acceptor;
	std::map<std::string, std::shared_ptr<CSession>> _sessions;
	std::mutex _mutex;
public:
	CServer(boost::asio::io_context& io_context, short port);
	~CServer();
	void ClearSession(std::string);
};
```

包含要求的头文件并using需要使用的命名空间和函数（**协程**）

成员函数在定义的时候详细解释，这里介绍以下成员变量：

- **io_context& _io_context**：声明一个_io_context的引用，注意，和 **boost::asio::io_context _io_context**不同，io_context**&** _io_context表示对现有io_context对象（其他地方创建的ioc对象）的引用，这里并不是声明一个ioc实例，而是表示对外界传入ioc的引用（主函数需要将监听错误信号signal的ioc传入server用于监听客户端的连接请求accept）
- **_port**：服务器端口，客户端与服务器的指定端口进行连接
- **_acceptor**：用于绑定服务器指定ip和端口号，并进行监听客户端连接信号
- **_sessions**：存储会话任务的容器，类似python的字典，键是一个字符串（不同会话有不同的**uuid**），值是session，为每个session会话映射对应的**uuid**，通过uuid可实现会话的**擦除、插入和寻找**
- **_mutex**：锁，保证线程安全

### b. 成员函数实现

```cpp
CServer::CServer(boost::asio::io_context& io_context, short port) :
	_io_context(io_context), _port(port), _acceptor(io_context, boost::asio::ip::tcp::endpoint(boost::asio::ip::tcp::v4(), port)) {
	std::cout << "Server start success, on port: " << port << std::endl;
	StartAccept();
}
```

CServer的构造函数接收指定的上下文服务io_context和端口号，并将acceptor绑定服务器指定的ip和端口号，最后调用StartAccept()启动监听服务

```cpp
void CServer::StartAccept() {
	auto& io_context = AsioIOServicePool::GetInstance()->GetIOService();
	std::shared_ptr<CSession> new_session = std::make_shared<CSession>(io_context, this);
	_acceptor.async_accept(new_session->GetSocket(),
		std::bind(&CServer::HandleAccept, this, new_session, std::placeholders::_1));
}
```

主函数（主线程）的io_context主要用于监听错误信号和监听客户端连接请求，但在每个线程中的io_context都是从线程池AsioIOServicePool中取出独立运行的。总共启动n+1个io_context，n是线程池中存储的ioc，1是主函数中的ioc。函数流程如下：

- 首先，从线程池中取出一个ioc，并将线程池中ioc的索引+1，以便轮询取出下一个ioc；
- 然后，定义一个新的session会话，传入从线程池中取出的ioc，并将server指针传入（如果该session会话错误，通过server的Clear函数擦除该session会话）。通过智能指针shared_ptr管理session声明周期，通过智能指针在C++中实现伪闭包，以防session在线程结束前被提前释放导致线程安全；
- 最后，使用绑定主函数传入ioc和指定端点的acceptor监听客户端（已经为该客户端建立了会话session，只需客户端进行连接即可进行收发操作）连接。如果监听成功，调用回调函数**HandleAccept。**

```cpp
void CServer::HandleAccept(std::shared_ptr<CSession> new_session, const boost::system::error_code& error) {
	if (!error) {
		new_session->Start();
		std::lock_guard<std::mutex> lock(_mutex);
		_sessions.insert(make_pair(new_session->GetUuid(), new_session));
	}
	else {
		std::cerr << "session accept failed, error is " << error.what() << std::endl;
	}

	StartAccept();
}
```

该回调函数主要处理客户端的连接请求，如果**客户端连接成功**（没有错误），那么启动会话`（new_session->Start()）`，并加锁将该会话以及对应的uuid插入至**_sessions**，以便后续管理此session，`‘}’`结束后自动释放锁，无需手动释放。

最后调用**StartAccept()**，**重新从线程池中取出ioc并创建一个新的会话session**，只待主线程的ioc监听到客户端连接申请之后，便将新会话交给新连接请求进行收发。

```cpp
void CServer::ClearSession(std::string uuid) {
	std::lock_guard<std::mutex> lock(_mutex);
	_sessions.erase(uuid);
}
```

该函数用于将指定**uuid**的会话从会话存储容器**_sessions**中擦除，每个uuid对应一个session，uuid通过雪花算法实现。

## 4. 线程池AsioIOServicePool

### a. AsioIOServicePool.h

```cpp
#pragma once
#include "boost/asio.hpp"
#include <iostream>
#include <vector>
#include <memory>

class AsioIOServicePool;

class SafeDeletor{
public:
	void operator()(AsioIOServicePool* st);
};

class AsioIOServicePool
{
public:
	friend class SafeDeletor;
	using IOService = boost::asio::io_context;
	using Work = boost::asio::io_context::work;
	using WorkPtr = std::unique_ptr<Work>;

	static std::shared_ptr<AsioIOServicePool> GetInstance() {
		static std::once_flag s_flag; 
		std::call_once(s_flag, [&]() { // call_once内部有锁
			_instance = std::shared_ptr<AsioIOServicePool>(new AsioIOServicePool, SafeDeletor());
			});
		return _instance;
	}
	boost::asio::io_context& GetIOService();
	void Stop();
private:
	AsioIOServicePool(std::size_t size = std::thread::hardware_concurrency());
	~AsioIOServicePool();
	AsioIOServicePool(const AsioIOServicePool&) = delete;
	AsioIOServicePool& operator = (const AsioIOServicePool&) = delete;

	std::vector<IOService> _ioServices;
	std::vector<WorkPtr> _works;
	std::vector<std::thread> _threads;
	std::size_t _nextIOService;
	static std::shared_ptr<AsioIOServicePool> _instance;
};
```

建立线程池**AsioIOServicePool**，用于多线程收发。该服务器有两个单例类，分别是AsioIOServicePool（线程池）和LogicSystem（逻辑层），我使用了两种方式实现单例模式，AsioIOServicePool的单例实现比较详细，LogicSystem的实现比较简单，但代码更少更清晰。

```cpp
class SafeDeletor{
public:
	void operator()(AsioIOServicePool* st);
};
```

**SafeDeletor**是指定的删除器，我们将析构函数设置为私有，防止析构函数被主动调用，析构函数只能通过删除器SafeDeletor被调用，这里重载了()，将SafeDeletor类作为仿函数使用。

**AsioIOServicePool**的成员函数在后面定义的时候详细解释，这里介绍AsioIOServicePool以下成员变量：

- **IOService**：将`boost::asio::io_context`重命名为IOService，以便使用
- **Work**：将`boost::asio::io_context::work`重命名为Work，Work用来绑定ioc，以防ioc.run()提前返回，详细解释可参考

[爱吃土豆：网络编程（16）——asio多线程模型IOServicePool5 赞同 · 0 评论文章](https://zhuanlan.zhihu.com/p/890395457)

- **WorkPtr**：使用unique_ptr管理work，希望该work不会被拷贝，只能移动或者从头用到尾不被改变
- **_ioServices**：io_context存储容器，用于存储指定数量的io_context
- **_works**：存储与ioc数量对应的work
- **_threads**：存储指定数量的线程
- **_nextIOService**：记录ioc在vector的下标，通过轮询返回ioc时，需要记录当前ioc的下标，累加，当超过vector的size时就归零，然后继续按轮询的方式返回
- **_instance**：声明一个静态成员变量，用于存储单例类AsioIOServicePool唯一实例，通过懒汉方式实现，单例模式的详细介绍可参考：

[爱吃土豆：网络编程（13）——单例模式0 赞同 · 0 评论文章](https://zhuanlan.zhihu.com/p/774652701)

### b. 成员函数实现

```cpp
void SafeDeletor::operator()(AsioIOServicePool* st) {
	std::cout << "this is safe deleter operator() of AsioIOServicePool" << std::endl;
	delete st;
}
```

不能直接调用单例类的析构函数，只能通过删除器间接调用，单例类的构造函数和析构函数都私有化，并将复制构造函数和赋值运算符delete，防止单例类被拷贝和赋值。

```cpp
AsioIOServicePool::AsioIOServicePool(std::size_t size) :
	_ioServices(size), _works(size), _nextIOService(0) {
	for (std::size_t i = 0; i < size; i++) {
		_works[i] = std::unique_ptr<Work>(new Work(_ioServices[i]));
	}

	// 遍历多个ioservice,创建多个线程，每个线程内部启动ioservice
	for (std::size_t i = 0; i < _ioServices.size(); i++) {
		_threads.emplace_back([this, i]() {
			_ioServices[i].run();
			});
	}
}
```

线程池AsioIOServicePool的构造函数需接收ioc的指定数量**size**，如果不显式指定，那么**size**会使用默认值 **std::thread::hardware_concurrency()**，该函数用于获取CPU核数。

work的数量和ioc的数量相同，索引_nextIOService默认值为0，上限为ioc的数量**size**

还有两个for循环，**第一个for循环**使用Work()函数将指定ioc与work绑定并通过右值移动给对应位置的_works容器，此时_works容器中对应索引便存储了与对应ioc绑定的work，只有将work给reset之后，ioc.run()才会返回。**第二个for循环**启动指定ioc数量_size的**线程**，每个线程中有独立的ioc，并启动**ioc.run()**

```cpp
boost::asio::io_context& AsioIOServicePool::GetIOService() {
	auto& service = _ioServices[_nextIOService++];
	if (_nextIOService == _ioServices.size())
		_nextIOService = 0;

	return service;
}
```

以_nextIOService为索引轮询取出线程池中的io_context

```cpp
void AsioIOServicePool::Stop() {
	for (auto& work : _works) {
		work.reset();
	}

	for (auto& t : _threads) {
		t.join();
	}
}
```

不同线程的io_context如果想要停止返回，那么必须将已绑定的work释放，该函数用于释放所有ioc的work，因为不同线程独立运行，所以必须要等待每个线程结束后才结束程序。

```cpp
	static std::shared_ptr<AsioIOServicePool> GetInstance() {
		static std::once_flag s_flag; 
		std::call_once(s_flag, [&]() { // call_once内部有锁
			_instance = std::shared_ptr<AsioIOServicePool>(new AsioIOServicePool, SafeDeletor());
			});
		return _instance;
	}
```

定义静态成员函数**GetInstance()**，通过C++11新特性once_flag 和call_once实现线程安全，保证多线程中AsioIOServicePool只会被创建一次，有且仅有唯一实例。

once_flag 和call_once的详细介绍可参考：

[爱吃土豆：C++11新特性0 赞同 · 0 评论文章](https://zhuanlan.zhihu.com/p/781350340)

最后注意，**非模板类**的静态成员变量和函数需要在实现文件cpp中初始化，不能在头文件中初始化。**模板类**的静态成员变量必须在头文件中定义，模板类中的静态变量可以在头文件中定义，是因为模板类在编译时不会像普通类那样生成具体的类型和符号。相反，模板类的代码只会在模板被实例化时才生成具体的类型。因此，模板类的静态成员变量不会像普通类的静态成员那样在多个编译单元中重复定义。

```cpp
AsioIOServicePool.cpp
// 初始化
std::shared_ptr<AsioIOServicePool> AsioIOServicePool::_instance = nullptr;
```

## 5. 逻辑层LogicSystem

一般在解析完对端发送的数据之后，还要对该请求做更进一步地处理，比如根据不同的消息id执行不同的逻辑层函数或不同的操作，比如读数据库、写数据库，还比如游戏中可能需要给玩家叠加不同的buff、增加积分等等，这些都需要交给**逻辑层**处理，而不仅仅是把消息发给对端。

![img](/images/$%7Bfiilename%7D/9bdf7fe5ff534c6a97d1b480313e7342.png)编辑

<center>服务器架构</center>

上图所示的是一个完成的服务器架构，一般需要将逻辑层独立出来，因为如果在解析完对端数据后需要执行一些复杂的操作，比如玩家需要叠加各自buff或者技能，此时可能会耗时1s甚至更多，如果没有独立的逻辑层进行操作，那么系统会一直停留在执行回调函数那一步，造成**阻塞**，直至操作结束。

而逻辑层是独立的，回调函数只需将数据投递给逻辑**队列**（回调函数将数据放入队列中之后系统会运行下一步，便不会被阻塞），逻辑系统会自动从队列中取数据并做相应操作，如果需要在执行完操作之后做相应回复，那么逻辑系统会调用写事件并注册写回调给asio网络层，网络层就是asio底层通信的网络层步骤。

### a. LogicSystem.h

```cpp
#pragma once
#include <queue>
#include <thread>
#include "CSession.h"
#include <map>
#include <functional>
#include "Const.h"
#include <json/json.h>
#include <json/value.h>
#include <json/reader.h>

class LogicNode;
class CSession;

typedef std::function<void(std::shared_ptr<CSession>, const short& msg_id, const std::string& msg_data)> FunCallBack;

class LogicSystem
{
private:
	LogicSystem();
	~LogicSystem();
	LogicSystem(const LogicSystem&) = delete;
	LogicSystem& operator=(const LogicSystem&) = delete;

	void RegisterCallBacks();
	void HelloWordCallBack(std::shared_ptr<CSession>, const short& msg_id, const std::string& msg_data);
	void DealMsg();

	std::queue<std::shared_ptr<LogicNode>> _msg_que;
	std::mutex _mutex;
	std::condition_variable _consume;
	std::thread _worker_thread;
	bool _b_stop;
	std::map<short, FunCallBack> _fun_callback;
public:
	void PostMsgToQue(std::shared_ptr<LogicNode> msg);
	static LogicSystem& GetInstance();
};
```

定义逻辑类的头文件，成员函数在后面定义时详细介绍，这里介绍一下逻辑类的成员变量：

- **FunCallBack**为要注册的回调函数类型，其参数为会话Session类智能指针，消息id，以及消息内容。
- **_msg_que**为逻辑队列，其中的元素相当于**RecvNode，**只不过为了实现**伪闭包**，所以创建一个LogicNode类，包含Session的智能指针防止被提前释放
- **_mutex** 为保证逻辑队列安全的互斥量
- **_consume**表示消费者条件变量，用来控制当逻辑队列为空时保证线程暂时挂起等待，不要干扰其他线程。
- **_fun_callbacks**用于存储消息id和对应消息id的回调函数，根据id查找对应的逻辑处理函数。
- **_worker_thread**表示工作线程，用来从逻辑队列中取数据并执行回调函数。
- **_b_stop**表示收到外部的停止信号，逻辑类要中止工作线程并优雅退出。

_fun_callback的定义如下：

```cpp
typedef std::function<void(std::shared_ptr<CSession>, const short& msg_id, const std::string& msg_data)> FunCallBack;

std::map<short, FunCallBack> _fun_callback;
```

_fun_callback是一个map，值和键的类型分别为short和FunCallBack，前者是消息id的类型，后者是回调函数。

FunCallBack使用了C++11新特性**std::function**来声明了一个函数签名，表示接受一个 **shared_ptr** 指向 **CSession**、一个 **short** 类型的消息 ID 和一个 **std::string** 类型的消息数据的可调用对象

**std::function**的详细介绍可参考

[爱吃土豆：C++11新特性0 赞同 · 0 评论文章](https://zhuanlan.zhihu.com/p/781350340)

因为逻辑队列存储的成员需要包含对应session和消息内容，所以需要重新定义一个逻辑节点，用于存储会话session中将对应消息投递至逻辑队列的内容，包括该session和消息内容。

### b. LogicNode

```cpp
class LogicNode {
	friend class LogicSystem;
public:
	LogicNode(std::shared_ptr<CSession>, std::shared_ptr<RecvNode>);
private:
	std::shared_ptr<CSession> _session;
	std::shared_ptr<RecvNode> _recvnode;
};
```

**LogicNode**类定义在**CSession.h**中，**CSession.h**在文章后面介绍

- **_session：**声明Session的智能指针，实现伪闭包，防止Session被提前释放
- **_recvnode**：接收消息节点的智能指针

构造函数如下：

```cpp
LogicNode::LogicNode(std::shared_ptr<CSession> session, std::shared_ptr<RecvNode> recvnode) :
_session(session), _recvnode(recvnode){}
```

用于接收一个session会话和该会话投递的消息内容

### c. 成员函数实现

```cpp
LogicSystem::LogicSystem() :_b_stop(false) {
	RegisterCallBacks();
	_worker_thread = std::thread(&LogicSystem::DealMsg, this);
}
```

- **_b_stop**初始化为false，当前工作线程不会停止
- **RegisterCallBacks()：**调用回调注册函数：这个函数用于**注册消息处理的回调函数**。将不同的消息 ID 和对应的处理函数关联起来，以便在处理消息时能够找到正确的函数。
- `_worker_thread = std::thread(&LogicSystem::DealMsg, this);`
  - 创建工作线程：使用 std::thread 创建一个新的线程。
  - `&LogicSystem::DealMsg`：指定要在新线程中执行的成员函数，即 DealMsg，该函数将负责从消息队列中提取消息并处理它。
  - this：传递当前对象的指针，以便 DealMsg 可以访问 LogicSystem 的成员变量和函数。

```cpp
void LogicSystem::RegisterCallBacks() {
	_fun_callback[MSG_HELLO_WORD] = std::bind(&LogicSystem::HelloWordCallBack,
		this, std::placeholders::_1, std::placeholders::_2, std::placeholders::_3);
}
```

将**消息ID**：**MSG_HELLO_WORD**映射至**回调函数**HelloWordCallBack，并存储至_fun_callback。当接收到消息MSG_HELLO_WORD时，系统会调用 HelloWordCallBack 方法来处理消息。

```cpp
void LogicSystem::HelloWordCallBack(std::shared_ptr<CSession> session, const short& msg_id, const std::string& msg_data) {
	Json::Reader reader;
	Json::Value root;
	reader.parse(msg_data, root);
	std::cout << "receive msg id is " << root["id"].asInt() << " msg data is "
		<< root["data"].asString() << std::endl;
	root["data"] = "server has receive msg, msg data is " + root["data"].asString();

	std::string return_str = root.toStyledString();
	session->Send(return_str, root["id"].asInt());
}
```

- HelloWordCallBack 是对应消息id：MSG_HELLO_WORD 的回调函数 	

  - 使用jsoncpp库包装或者解析数据
  - 调用 parse 方法将 **msg_data** 字符串解析为 JSON 格式，结果存储在 root 中
  - 通过session类的send回传消息

```cpp
void LogicSystem::DealMsg() {
	for (;;) {
		std::unique_lock<std::mutex> unique_lk(_mutex);
		// 循环会耗费资源，这里使用条件变量挂起线程，并释放锁，直至被唤醒
		while (_msg_que.empty() and !_b_stop) {
			_consume.wait(unique_lk);
		}

		// 判断如果为关闭状态，取出逻辑队列所有数据处理并退出循环
		if (_b_stop) {
			while (!_msg_que.empty()) {
				auto msg_node = _msg_que.front();
				std::cout << "recv msg id is " << msg_node->_recvnode->GetMsgID() << std::endl;
				auto call_back_iter = _fun_callback.find(msg_node->_recvnode->GetMsgID());
				if (call_back_iter == _fun_callback.end()) { // 如果未注册对应消息id的回调函数
					_msg_que.pop(); // 释放该消息
					continue; // 处理下一条消息
				}
				// 取出对应消息id的回调函数
				call_back_iter->second(msg_node->_session, msg_node->_recvnode->GetMsgID(),
					std::string(msg_node->_recvnode->_data, msg_node->_recvnode->_total_len));
				_msg_que.pop();
			}
			break;
		}

		// 如果没有停服，且队列不为空
		auto msg_node = _msg_que.front();
		std::cout << "recv msg id is " << msg_node->_recvnode->GetMsgID() << std::endl;

		auto call_back_iter = _fun_callback.find(msg_node->_recvnode->GetMsgID());
		// 若队列中没有该消息对应的回调函数，则释放该消息并处理下一条
		// 注册_fun_callback时，将消息id以及匹配的回调函数绑定在一起
		if (call_back_iter == _fun_callback.end()) {
			_msg_que.pop();
			continue;
		}
		// 执行消息类型对应的回调函数
		call_back_iter->second(msg_node->_session, msg_node->_recvnode->GetMsgID(),
			std::string(msg_node->_recvnode->_data, msg_node->_recvnode->_total_len));
		_msg_que.pop();
	}
}
```

当逻辑类实例化后，开始循环运行消息处理函数**DealMsg()**。为了实现自由加解锁，定义一个unique_lock类的锁unique_lk。当逻辑队列为空并且服务器未停止时，进行等待while循环中，因为while循环也会占用资源，所以使用C++11特性条件变量 **std::condition_variable**将线程挂起，该线程不会占用cpu资源，并释放锁使得其他线程可以获取共享资源，线程挂起持续至被唤醒（notify_one()或者notify_all()）。条件变量的详细介绍可参考

[爱吃土豆：C++11新特性0 赞同 · 0 评论文章](https://zhuanlan.zhihu.com/p/781350340)

当线程被唤醒后，如果服务器关闭，则取出逻辑队列所有数据处理并退出循环

当线程被唤醒后，如果队列不为空且服务器未关闭，那么取出队列第一个数据，并在注册map中查找对应数据id的回调函数，如果没有找到，那么释放该数据并处理下一条消息；反之，调用该回调函数。这里我们只会传入消息id：MSG_HELLO_WORD，回调函数HelloWordCallBack会被调用

```cpp
LogicSystem::~LogicSystem() {
	std::cout << "逻辑层成功析构" << std::endl;
	_b_stop = true;
	_consume.notify_one();
	_worker_thread.join();
}
```

逻辑类的析构函数在所有工作线程运行结束后被执行，但工作线程可能会处于挂起状态，此时需要一个激活信号打断**_consume**的**wait**状态（在该命令前一步将_b_stop置为true）

```cpp
void LogicSystem::PostMsgToQue(std::shared_ptr<LogicNode> msg) {
	std::unique_lock<std::mutex> unique_lk(_mutex); // 锁定互斥量以保护共享数据
	_msg_que.push(msg); // 将消息推入队列

	if (_msg_que.size() == 1) { // 如果这是队列中的第一个消息
		_consume.notify_one(); // 通知消费者线程，结束wait状态
	}
}
```

该函数在Session会话中将对应session指针和消息内容投递至逻辑队列处理。

```cpp
LogicSystem& LogicSystem::GetInstance() {
	// 返回局部静态变量在C++11及以上是线程安全的
	static LogicSystem instance;
	return instance;
}
```

LogicSystem类也是一个**单例模式**，我这里使用了第二种方式来定义单例模式。我们只需要定义一个静态成员函数GetInstance()，在该函数中返回局部静态变量instance，instance是LogicSystem 的实例。该方法只能在C++11及以后的平台才可以使用，**因为返回局部静态变量只有在C++11及以上是线程安全的。**

逻辑类的详细介绍可参考：

[爱吃土豆：网络编程（12）——完善粘包处理操作+asio底层通信过程2 赞同 · 0 评论文章](https://zhuanlan.zhihu.com/p/745782212)

[爱吃土豆：网络编程（14）——基于单例模板实现的逻辑层1 赞同 · 0 评论文章](https://zhuanlan.zhihu.com/p/779220373)

## 6. CSession

CSession用于接收客户端的连接，并处理异步收发，在该类中，构造一个**收协程**用于接收数据，但发仍然通过Send函数进行队列发送，不在构造一个**发协程。**理由如下：

- 可以构建发送协程用于发送数据，但考虑到协程很大程度上在使用层面简化了代码，但实际上也耗费资源，如果服务器进行频繁发送操作，且处于任何时刻都可能调用的情况下，使用协程可能并不方便 ，因为可能在一个线程中，随着发送次数的上升，启动很多协程
- 还有一个原因是：如果发送的时候处于另一个线程，如果用发送协程的话，可能会导致其他线程声明周期的改变

所以我们这里仍然使用之前代码中用到的发送队列的方式。

### a. CSession.h

```cpp
#pragma once
#include <iostream>
#include <boost/asio.hpp>
#include <queue>
#include "MsgNode.h"
#include "LogicSystem.h"

class CServer;

class CSession :public std::enable_shared_from_this<CSession>
{
private:
	void HandleWrite(const boost::system::error_code& error, std::shared_ptr<CSession> shared_self);
	boost::asio::io_context& _io_context;
	CServer* _server;
	boost::asio::ip::tcp::socket _socket;
	std::string _uuid;
	bool _b_close;
	std::mutex _send_lock;
	std::queue<std::shared_ptr<SendNode>> _send_que;
	std::shared_ptr<RecvNode> _recv_msg_node;
	std::shared_ptr<MsgNode> _recv_head_node;
public:
	CSession(boost::asio::io_context& io_context, CServer* server);
	~CSession();
	boost::asio::ip::tcp::socket& GetSocket();
	void Start();
	std::string& GetUuid();
	void Close();
	void Send(const char* msg, short max_length, short msg_id);
	void Send(std::string msg, short msg_id);
};

class LogicNode {
	friend class LogicSystem;
public:
	LogicNode(std::shared_ptr<CSession>, std::shared_ptr<RecvNode>);
private:
	std::shared_ptr<CSession> _session;
	std::shared_ptr<RecvNode> _recvnode;
};
```

同样，CSession类的成员函数在后面定义时详细介绍，这里仅介绍成员变量：

- `boost::asio::io_context& _io_context`：用于接收从线程池**AsioIOServicePool**中提取的ioc，注意，这里是引用而不是重新声明一个io_context新实例
- **CServer _server**：接收CServer指针，当Session收发操作出错时，方便调用Server的Clear函数擦除该session会话任务
- **_socket**：每个session都有一个独立的套接字，负责异步收发
- **_b_close**：session会话任务是否结束的标志位
- **_send_que**：发送队列，逻辑层会调用session的Send发送函数，Send函数会将需要发送的消息添加到发送队列，保证发送的顺序性
- **_recv_msg_node**：接收节点，将接收的消息存储至该节点，包括消息id和消息内容
- **_recv_head_node**：消息头节点，将收到消息的id和长度存储至该节点
- **LogicNode**：逻辑节点，负责将session和消息内容投递至逻辑队列，逻辑队列元素的格式为LogicNode

### b. 成员函数实现

```cpp
CSession::CSession(boost::asio::io_context& io_context, CServer* server) :
	_io_context(io_context), _server(server), _socket(io_context), _b_close(false) {
	// random_generator是函数对象，加()就是函数，再加一个()就是调用该函数
	boost::uuids::uuid a_uuid = boost::uuids::random_generator()();
	_uuid = boost::uuids::to_string(a_uuid);
	_recv_head_node = std::make_shared<MsgNode>(HEAD_TOTAL_LEN);
}
```

- CSession的构造函数需要接收线程池中的ioc、server指针，并将_b_close置为false，表示该会话未关闭
- 通过boost的random_generator()()函数获取一个uuid，每个session都有自己的uuid，相当于一个名称，通过该名称可找出对应的session会话，该函数通过雪花算法实现。
- 定义消息接收节点_recv_head_node，总长度4字节，包括消息id和消息长度

```cpp
void CSession::Start() {
	// 防止协程处理过程中，智能指针被意外释放，通过智能指针实现伪闭包
	auto shared_this = shared_from_this();
	// 开启协程接收
	boost::asio::co_spawn(_io_context, [self = shared_from_this(), this]()->boost::asio::awaitable<void> {
		try {
			for (; !_b_close;) {
				_recv_head_node->Clear();
				// 以同步的方式调用异步读，并将读到的字节数（4字节）返回（使用简易粘包处理方式）
				std::size_t n = co_await boost::asio::async_read(_socket, boost::asio::buffer(
					_recv_head_node->_data, HEAD_TOTAL_LEN), use_awaitable);

				if (n == 0) { // 如果没读到数据
					std::cout << "receive peer closed" << std::endl;
					Close();
					_server->ClearSession(_uuid);
					co_return; // 类似return，但co_return是非阻塞的
				}

				// 获取头部消息id
				short msg_id = 0;
				memcpy(&msg_id, _recv_head_node->_data, HEAD_ID_LEN);
				// 转换网络字节序为本地字节序
				msg_id = boost::asio::detail::socket_ops::network_to_host_short(msg_id);
				std::cout << "msg id is " << msg_id << std::endl;
				if (msg_id > MAX_LENGTH) {
					std::cout << "invalid msg id is " << msg_id << std::endl;
					Close();
					_server->ClearSession(_uuid);
					co_return;
				}

				// 获取消息长度
				short msg_len = 0;
				memcpy(&msg_len, _recv_head_node->_data + HEAD_ID_LEN, HEAD_DATA_LEN);
				msg_len = boost::asio::detail::socket_ops::network_to_host_short(msg_len);
				std::cout << "msg len is " << msg_len << std::endl;
				if (msg_len > MAX_LENGTH) {
					std::cout << "invalid msg len is " << msg_len << std::endl;
					Close();
					_server->ClearSession(_uuid);
					co_return;
				}

				// 获取消息内容
				_recv_msg_node = std::make_shared<RecvNode>(msg_len, msg_id); // 构造消息节点
				// 将异步读以同步方式调用，指定读取字节数
				n = co_await boost::asio::async_read(_socket, boost::asio::buffer(_recv_msg_node->_data,
					_recv_msg_node->_total_len),use_awaitable);
				if (n == 0) {
					std::cout << "receive peer closed " << std::endl;
					Close();
					_server->ClearSession(_uuid);
					co_return;
				}

				_recv_msg_node->_data[_recv_msg_node->_total_len] = '\0';
				std::cout << "receive data is " << _recv_msg_node->_data << std::endl;
				// 投递至逻辑线程
				LogicSystem::GetInstance().PostMsgToQue(std::make_shared<LogicNode>(shared_from_this(), _recv_msg_node));
			}
		}
		catch (std::exception& e) {
			std::cerr << "exception is " << e.what() << std::endl;
			Close();
			_server->ClearSession(_uuid);
		}
		}, detached);
}
```

**Start()**函数用于接收消息内容，并通过另一种简便的粘包处理方式获得消息id、消息长度，可参考：

[爱吃土豆：网络编程（11）——另一种简便的粘包处理方式1 赞同 · 0 评论文章](https://zhuanlan.zhihu.com/p/730535859)

该函数在Server中收到客户端连接请求后，Server将一个新的session分配给该客户端，并调用**Start()**函数。

首先，构造一个**收协程：**

- 这里通过**boost::asio::co_spawn**构造一个收协程，传入ioc和lambda函数，以及**detached**。注意，**如果用 `=` 捕获所有变量，但是lambda表达式并未使用智能指针则不会增加引用计数，除非在lambda内部使用这个指针或者在 `[]` 中显式捕获该指针，我们这里通过`self = shared_from_this()`, this显式增加session的引用计数，防止协程处理过程中，智能指针被意外释放，通过智能指针实现伪闭包。**
- 显式指定返回类型为`boost::asio::awaitable<void>`，`awaitable<void>`类型允许函数在执行时可以被暂停和恢复，这使得它能够与 **co_await** 一起使用，所以函数返回类型必须是`awaitable<void>`。
- session会在Start()函数中循环执行收协程，直至该会话被关闭，每一次执行都需要将消息头结点清空，接收下一个消息。通过**co_await** 关键字和**use_awaitable** 以同步的方式调用异步读操作async_read，并将读到的字节数（4字节）返回，只有读到4字节才会返回，否则已知挂起。这里和asynv_read_some有一些区别然后分别读取消息id、消息长度和消息内容，且需要将消息id和消息长度从网络序列转换为本地序列。这里使用**async_read**指定读取字节数，而不是像之前粘包处理使用**async_read_some**函数，通过调用回调函数处理粘包问题。
- 最后，在接收完所有消息后，将消息投递至逻辑队列进行相应的处理。

在协程中，co_return用来返回一个值或表示**协程结束**，它将把值传递给协程的返回对象（如果返回类型不为`boost::asio::awaitable<void>`，对象是`boost::asio::awaitable`）。

- co_return 只能在协程中使用，而普通函数中使用的是 return。
- co_return 将结果传递给协程的承诺（promise）对象，这个对象会将值交给协程的返回类型

协程关键字和函数**use_awaitable、co_spawn、detached**如何使用请参考文章：

[爱吃土豆：网络编程（18）——使用asio协程实现并发服务器demo（官方案例）4 赞同 · 0 评论文章](https://zhuanlan.zhihu.com/p/925377617)

**我们已经构建了一个接收协程用于接收数据，那么我们需要构建发送协程吗？**

- 可以构建发送协程用于发送数据，但考虑到协程很大程度上在使用层面简化了代码，但实际上也耗费资源，如果服务器进行频繁发送操作，且处于任何时刻都可能调用的情况下，使用协程可能并不方便 ，因为可能在一个线程中，随着发送次数的上升，启动很多协程
- 还有一个原因是：如果发送的时候处于另一个线程，如果用发送协程的话，可能会导致其他线程声明周期的改变

所以我们这里仍然使用之前代码中用到的发送队列的方式，Send函数两种重载定义如下：

```cpp
void CSession::Send(const char* msg, short max_length, short msg_id) {
	bool pending = false; // 发送标志，true时有未完成的发送操作，false为空
	std::unique_lock<std::mutex> lock(_send_lock);
	int send_que_size = _send_que.size();
	if (send_que_size > MAX_SENDQUE) {
		std::cout << "session: " << _uuid << " send que fulled, size is " << MAX_SENDQUE << std::endl;
		return;
	}

	// 判断队列是否有未完成的发送操作
	if (_send_que.size() > 0) {
		pending = true;
	}
	_send_que.push(std::make_shared<SendNode>(msg, max_length, msg_id));
	if (pending) { // 如果有未完成的发送，直接返回
		return;
	}

	auto msgnode = _send_que.front();
	// 数据取出来之后就解锁，仅在使用队列时加锁，避免不相干的逻辑占用锁
	lock.unlock();
	boost::asio::async_write(_socket, boost::asio::buffer(msgnode->_data, msgnode->_total_len),
		std::bind(&CSession::HandleWrite, this, std::placeholders::_1, shared_from_this()));
}

void CSession::Send(std::string msg, short msg_id) {
	bool pending = false; // 发送标志，true时有未完成的发送操作，false为空
	std::unique_lock<std::mutex> lock(_send_lock);
	int send_que_size = _send_que.size();
	if (send_que_size > MAX_SENDQUE) {
		std::cout << "session: " << _uuid << " send que fulled, size is " << MAX_SENDQUE << std::endl;
		return;
	}

	// 判断队列是否有未完成的发送操作
	if (_send_que.size() > 0) {
		pending = true;
	}
	_send_que.push(std::make_shared<SendNode>(msg.c_str(), msg.length(), msg_id));
	if (pending) { // 如果有未完成的发送，直接返回
		return;
	}

	auto msgnode = _send_que.front();
	// 数据取出来之后就解锁，仅在使用队列时加锁，避免不相干的逻辑占用锁
	lock.unlock();
	boost::asio::async_write(_socket, boost::asio::buffer(msgnode->_data, msgnode->_total_len),
		std::bind(&CSession::HandleWrite, this, std::placeholders::_1, shared_from_this()));
}
```

Send()的两种重载，针对数据内容是string类型和`char*`类型。

首先设置unique_lock，方便自由加解锁；

获取发送队列元素个数，并判断元素数量是否合法，如果队列中已存在元素，那么将本条消息传入队列然后返回，等待上一条消息发送成功；如果队列为空，那么取出队列的第一个元素并**解锁（我们只需要在使用队列时加锁，不使用队列是解锁，避免不相干的逻辑占用锁）**，然后同步异步写async_write发送数据（指定字节数量），这里不需要构造写协程，而是直接通过send函数调用异步写进行发送，发送成功后调用回调函数**HandleWrite**。

```cpp
void CSession::HandleWrite(const boost::system::error_code& error, std::shared_ptr<CSession> shared_self) {
	try {
		if (!error) { 
			std::unique_lock<std::mutex> lock(_send_lock); // 加锁保护发送队列
			_send_que.pop(); // 移除上一个已发送的消息（send函数中的异步发）
			if (!_send_que.empty()) { // 若队列不为空，处理下一个消息
				auto& msgnode = _send_que.front();
				lock.unlock(); // 解锁，避免不相干逻辑占用锁
				boost::asio::async_write(_socket, boost::asio::buffer(msgnode->_data, msgnode->_total_len),
					std::bind(&CSession::HandleWrite, this, std::placeholders::_1, shared_self));
			}
		}
		else {
			std::cerr << "handle write failed, error is " << error.what() << std::endl;
			Close();
			_server->ClearSession(_uuid);
		}
	}
	catch (std::exception& e) {
		std::cout << "exception is " << e.what() << std::endl;
		Close();
		_server->ClearSession(_uuid);
	}
}
```

如果发送成功，那么先加锁保护发送队列，然后移除上一个已发送的消息，并判断队列是否为空，如果不为空，取出第一个数据并解锁，并调用异步发函数发送该数据；如果发送失败，那么关闭session，并调用Server的ClearSession函数擦除对应uuid的session。

```cpp
boost::asio::ip::tcp::socket& CSession::GetSocket() {
	return _socket;
}
```

返回session的独立套接字

```cpp
std::string& CSession::GetUuid() {
	return _uuid;
}
```

返回session的uuid

```cpp
void CSession::Close() {
	_b_close = true;
	_socket.close();
}
```

显式关闭Session会话，首先将标志位_b_close 置1，并关闭套接字

```cpp
CSession::~CSession() {
	try {
		std::cout << "~CSession destruct" << std::endl;
	}
	catch (std::exception& e) {
		std::cerr << "exception is " << e.what() << std::endl;
	}
}
```

## 7. 主函数

```cpp
#include <iostream>
#include "CServer.h"
#include <csignal>
#include <thread>
#include <mutex>
#include "AsioIOServicePool.h"

int main()
{
	try {
		auto pool = AsioIOServicePool::GetInstance();
		boost::asio::io_context io_context;
		boost::asio::signal_set signals(io_context, SIGINT, SIGTERM);
		signals.async_wait([&io_context, &pool](auto,auto) {
			io_context.stop();
			pool->Stop();
			});

		CServer s(io_context, 10086);
		io_context.run();
	}
	catch (std::exception& e) {
		std::cerr << "Exception: " << e.what() << std::endl;
   }
}
```

## 8. 客户端

```cpp
#include <boost/asio.hpp>
#include <iostream>
#include <json/json.h>
#include <json/value.h>
#include <json/reader.h>

using namespace boost::asio::ip;
using std::cout;
using std::endl;
const int MAX_LENGTH = 1024 * 2; // 发送和接收的长度为1024 * 2字节
const int HEAD_LENGTH = 2;
const int HEAD_TOTAL = 4;

int main()
{
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
        getchar();
    }
    catch (std::exception& e) {
        std::cerr << "Exception: " << e.what() << endl;
    }
    return 0;
}
```

## 9. 测试

![img](/images/$%7Bfiilename%7D/dd34a3fb637b4548b6d4f99def4a6be2.png)



代码：

[https://github.com/qiaobeibei/CoroutineServergithub.com/qiaobeibei/CoroutineServer](https://link.zhihu.com/?target=https%3A//github.com/qiaobeibei/CoroutineServer)
