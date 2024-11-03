---
title: 网络编程（22）——通过beast库快速实现websocket服务器
date: 2024-11-03 12:12:57
categories:
- 网络编程
tags: 
- websocker
- beast库
- http1.1
typora-root-url: ./..
---

##  二十二、day22

因为http受限于**请求-响应模式**，客户端发起请求，服务器响应后连接立即关闭，每次通信都要重新建立连接，如果我们想要服务器与客户端之间可以随时互相发送数据，那么http只有多次重新建立客户端与服务器的连接才能满足我们的需求，但开销太大。

websocket有两种实现方式，第一种是基于tcp长连接进行升级，第二种是基于http升级，今天主要学习第一种。

如果是在tcp长连接进行升级，实际就是在已有的TCP连接之上，**直接切换到WebSocket协议**。它的核心思路是：在已经建立的TCP连接上，通过WebSocket协议的握手过程，达成协议切换，从而实现全双工的长连接通信。

如果在http的基础上进行实现websocket，那么websocket其实就是基于http进行初始握手，然后升级为websocket协程，从短连接升级为长连接，WebSocket则成为一个独立的、全双工的长连接协议，不再受HTTP的请求-响应模式限制。

参考视频：

[C++ 网络编程(23) beast网络库实现websocket服务器_哔哩哔哩_bilibili![img](/images/$%7Bfiilename%7D/icon-default-1730607206171-219.png)https://www.bilibili.com/video/BV1Mu411b7qV/?spm_id_from=333.337.search-card.all.click](https://www.bilibili.com/video/BV1Mu411b7qV/?spm_id_from=333.337.search-card.all.click)

## 1. websocket简述

websocket有两种升级方式，第一种是基于tcp进行升级，第二种基于http进行升级，前者只能接受websocket请求，而后者既可以接受http请求，也可以接受websocket请求，今天主要学习第一种。

WebSocket协议需要先通过HTTP协议进行初始握手。客户端发送一个HTTP请求，其中包含了**Upgrade**头，表明希望将连接从HTTP升级为WebSocket。服务器同意后，会返回**HTTP 101 Switching Protocols**状态码，确认连接升级为WebSocket。

**1）HTTP握手请求**：客户端发送的HTTP请求头中包含以下内容：

```cpp
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

这里，协议由1.0升级为1.1，keep_alive置为true，升级为长连接。

请求头已经被beast库封装好了，我们只需要调用就行，不需要像上面这样构造一个请求头。步骤如下：

```cpp
stream<tcp_stream> ws(ioc);
net::ip::tcp::resolver resolver(ioc);
get_lowest_layer(ws).connect(resolver.resolve("www.example.com", "ws"));

// Do the websocket handshake in the client role, on the connected stream.
// The implementation only uses the Host parameter to set the HTTP "Host" field,
// it does not perform any DNS lookup. That must be done first, as shown above.

ws.handshake(
    "www.example.com",  // The Host field
    "/"                 // The request-target
);
```

首先，初始化一个websoket对象ws，然后定义一个解析器，解析对端的地址并连接。

连接成功后，websocket调用handshake函数进行一次握手，用于升级协议。如果想判断握手是否成功，可初始化一个response_type 对象

```cpp
// This variable will receive the HTTP response from the server
response_type res;
ws.handshake(
    res,                // Receives the HTTP response
    "www.example.com",  // The Host field
    "/"                 // The request-target
);
```

通过解析res的内容，即可判断握手是否成功

**2**）**服务器响应**：如果服务器支持WebSocket并同意升级，会返回：

```cpp
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

在此之后，连接便从HTTP切换为WebSocket协议，开始全双工通信。

而beast同样也已经将其封装好了，我们只需要调用ws.accept()或者async_accept()函数即可升级协程为websocket.

```cpp
ws.accept();
ws.async_accept()
```

该函数首先会接收客户端的握手协议，并解析校验，如果没有问题，就向对端回一个响应，内容就是上面那部分内容。

除此之外，如果服务器先从数据流中读取数据，之后再升级为websocket，beast同样提供了这样的函数：

```cpp
// This buffer will hold the HTTP request as raw characters
std::string s;

// Read into our buffer until we reach the end of the HTTP request.
// No parsing takes place here, we are just accumulating data.

net::read_until(sock, net::dynamic_buffer(s), "\r\n\r\n");

// Now accept the connection, using the buffered data.
ws.accept(net::buffer(s));
```

通过调用read_until函数读取对端请求，直至读取到‘\r\n\r\n’，也就是直至将整个请求头读取出来，请求头的末尾是‘\r\n\r\n’。然后再调用accept接收升级。

3）在http服务器的基础上使用websocket协议

```cpp
// This buffer is required for reading HTTP messages
flat_buffer buffer;

// Read the HTTP request ourselves
http::request<http::string_body> req;
http::read(sock, buffer, req);

// See if its a WebSocket upgrade request
if(websocket::is_upgrade(req))
{
    // Construct the stream, transferring ownership of the socket
    stream<tcp_stream> ws(std::move(sock));

    // Clients SHOULD NOT begin sending WebSocket
    // frames until the server has provided a response.
    BOOST_ASSERT(buffer.size() == 0);

    // Accept the upgrade request
    ws.accept(req);
}
else
{
    // Its not a WebSocket upgrade, so
    // handle it like a normal HTTP request.
}
```

首先，和昨天学的一样，先定义一个请求头req用于将读到的请求内容存放到请求头req中，然后调用websocket::is_upgrade(req)函数判断是否要升级为websocket，如果是就调用accept升级。

## 2. 基于TCP长连接实现sebsocket

在已经建立的TCP连接上，通过WebSocket协议的握手过程，达成协议切换，从而实现全双工的长连接通信。

实现步骤如下：

**1）建立TCP连接**：客户端和服务器首先通过TCP协议建立一个长连接（通常是在特定的端口上，比如80或443）。

**2）协议升级：**

- 在这个TCP连接上，客户端发起一个WebSocket握手请求，携带必要的信息（如Sec-WebSocket-Key等），告知服务器希望将TCP连接升级为WebSocket协议。
- 服务器收到请求后，同意并返回相应的信息（Sec-WebSocket-Accept），确认连接升级。

**3）通信切换**：升级完成后，TCP连接保持不变，但通信协议切换为WebSocket协议。这时，客户端和服务器可以在这个TCP长连接上，进行实时的全双工数据交换。

代码实现如下：

### a. Connection

该类用于管理**一个**Websocket连接（多个connection通过ConnectionMgr类进行管理），并负责客户端与服务器间的通信。

因为beast库为websocket单独封装了一个超时定时器，所以在这里不需要向上一节http服务器的搭建一样自定义一个超时器函数。

```cpp
#pragma once
#include <iostream>
#include <boost/beast.hpp>
#include <boost/asio.hpp>
#include <memory>
#include <boost/uuid/uuid.hpp>
#include <boost/uuid/uuid_io.hpp>
#include <queue>
#include <mutex>
#include <boost/uuid/uuid_generators.hpp>

namespace net = boost::asio;
namespace beast = boost::beast;
namespace websocket = boost::beast::websocket;
using namespace boost::beast;

class ConnectionMgr;

class Connection : public std::enable_shared_from_this<Connection> {
private:
	std::unique_ptr<websocket::stream<tcp_stream>> _ws_ptr;
	std::string _uuid;
	net::io_context& _ioc;
	flat_buffer _recv_buffer;
	std::queue<std::string> _send_que;
	std::mutex _send_mtx;
public:
	Connection(net::io_context& ioc);
	std::string GetUid();
	net::ip::tcp::socket& GetSocket();
	void AsyncAccept();
	void Start();
	void AsyncSend(std::string msg);
};
```

首先，为了实现伪闭包，Connection类需要继承std::enable_shared_from_this<T>，便于调用shared_from_this()函数。

成员变量的介绍如下：

- **std::unique_ptr<websocket::stream<tcp_stream>> _ws_ptr**：初始化一个WebSocket流指针，用于管理WebSocket连接。其中，websocket::stream<T>是Beast 库中用于表示一个 WebSocket 流的模板类，这里将一个底层的TCP连接（tcp_stream）包装为一个WebSocket，通过这个类，程序可以在一个 TCP 连接上实现 WebSocket 协议，进行双向的实时通信。

通过 **websocket::stream<tcp_stream>** 简单写一个**客户端**：

```cpp
        net::io_context ioc;
        tcp::resolver resolver(ioc);
        auto const results = resolver.resolve("example.com", "80");

        // 创建 TCP 流并连接到服务器
        tcp::socket socket(ioc);
        net::connect(socket, results.begin(), results.end());

        // 创建 WebSocket 流并进行 WebSocket 握手
        websocket::stream<tcp::socket> ws(std::move(socket));
        ws.handshake("example.com", "/");

        // 发送一条消息
        ws.write(net::buffer(std::string("Hello WebSocket!")));

        // 接收服务器返回的消息
        beast::flat_buffer buffer;
        ws.read(buffer);
        std::cout << beast::make_printable(buffer.data()) << std::endl;

        // 关闭 WebSocket 连接
        ws.close(websocket::close_code::normal);
```

- **_uuid**：用于存储每个connection的名称，每个名称都是唯一的
- **_ioc**：上下文，注意，这里是&而不是初始化一个io_context
- **flat_buffer _recv_buffer**：用于接收WebSocket消息的缓冲区，flat_buffer 是 Boost.Beast 库中用于管理内存缓冲区的一个类，它的主要作用是为网络通信中的数据接收和发送提供一个高效的缓冲区，一般用于WebSocket、HTTP 协议的数据收发。该缓存区可以和beast的解析器配合使用，以解析数据。
- **_send_que**：存储发送消息的队列
- **_send_mtx**：互斥锁

Connection类的函数实现如下：

#### *1）构造函数*

```cpp
Connection::Connection(net::io_context& ioc) : 
	_ioc(ioc), _ws_ptr(std::make_unique<websocket::stream<tcp_stream>>(make_strand(ioc))){
	// 生成uuid
	boost::uuids::random_generator generator;
	boost::uuids::uuid uuid = generator();
	_uuid = boost::uuids::to_string(uuid);
}
```

首先，构造函数接受一个io_context引用的ioc作为参数，并将ioc赋值给_ioc，用于管理异步操作。然后，创建一个 websocket::stream<tcp_stream> 对象，并赋值给_ws_ptr（这里要么在初始化列表中进行赋值，要么通过右值进行赋移动值，因为_ws_ptr是一个unique_ptr不允许被复制拷贝或者赋值）。

然后，通过boost自带的函数（雪花算法）生成一个唯一的uuid。每个Connection的uuid可以代表这个独立的连接。

#### *2）GetUid()*

```cpp
std::string Connection::GetUid() {
	return _uuid;
}
```

#### *3）GetSocket()*

```cpp
net::ip::tcp::socket& Connection::GetSocket() {
	// 获取最底层的socket
	return boost::beast::get_lowest_layer(*_ws_ptr).socket();
}
```

返回一个指向底层使用的 TCP socket，在 WebSocket 的实现中，它本质上是基于一个底层的 TCP 连接来进行数据传输的。

返回局部变量的引用会有问题吗？在这里不会有问题，因为socket是由_ws_ptr生成的，只有_ws_ptr不被释放，那么socket就一直存在。

#### *4）AsyncAccept()*

```cpp
void Connection::AsyncAccept() {
	auto self = shared_from_this();
	_ws_ptr->async_accept([self](boost::system::error_code err) {
		try {
			if (!err) {
				ConnectionMgr::GetInstance().AddConnection(self);
				self->Start();

			}
			else {
				std::cout << "websocket accept failed, err is " << err.what() << std::endl;
			}
		}
		catch (std::exception& e) {
			std::cerr << "websocket async accept exception is " << e.what() << std::endl;
		}
		});
}
```

该函数用于异步接收WebSocket 连接，通过 **async_accept()** 处理传入的 WebSocket 握手请求。

注意，这里WebSocket 对象的 async_accept 和 acceptor对象的 async_accept **不一样。**

1）WebSocket 对象的 async_accept主要用于处理WebSocket 协议的握手（**handshake**），即在建立 TCP 连接后，客户端与服务器需要通过 WebSocket 协议进行握手以升级该连接为 WebSocket 连接。

- 首先，客户端通过 HTTP 发出一个特殊的握手请求，要求升级到 WebSocket 协议。
- 服务端使用 WebSocket 的 async_accept() 方法来异步处理这个请求，并返回相应的 WebSocket 握手响应。
- 握手成功后，连接正式变为 WebSocket 连接，双方可以开始传输 WebSocket 帧

2）而acceptor对象的 async_accept 用于异步接受一个新的 TCP 连接（socket），即监听某个端口，并等待客户端的连接请求，这里的 async_accept() 是处理纯 TCP 连接的接受，并不涉及 WebSocket 协议。

Websocker的流程如下：

```bash
1. TCP 层：
   - 服务端：TCP Acceptor `async_accept()`
     (等待并接受 TCP 连接)
   - 客户端：发起 TCP 连接

2. WebSocket 层：
   - 服务端：WebSocket `async_accept()`
     (处理 WebSocket 握手请求)
   - 客户端：发送 WebSocket 协议升级请求

3. WebSocket 数据传输：
   - 服务端和客户端通过 WebSocket 协议进行数据传输
```

如果WebSocket 握手成功，调用lambda函数，将当前连接对象加入ConnectionMgr 的管理中，并调用该连接对象的Start()函数开始处理WebSocket 的数据传输，进入连接的业务逻辑。

#### *5）Start()*

```cpp
void Connection::Start() {
	auto self = shared_from_this();
	_ws_ptr->async_read(_recv_buffer, [self](boost::system::error_code ec, std::size_t buffer_bytes) {
		try {
			if (!ec) {
				self->_ws_ptr->text(self->_ws_ptr->got_text());
				std::string recv_data = boost::beast::buffers_to_string(self->_recv_buffer.data());
				self->_recv_buffer.consume(self->_recv_buffer.size());
				std::cout << "websocket receive msg is " << recv_data << std::endl;

				self->AsyncSend(std::move(recv_data));
				self->Start();
			}
			else {
				std::cout << "websocket async read error is " << ec.what() << std::endl;
				return;
			}
		}
		catch (std::exception& e) {
			std::cerr << "exception is " << e.what() << std::endl;
			ConnectionMgr::GetInstance().RmvConnection(self->GetUid());
		}
		});
}
```

该函数用于读取 WebSocket 消息，并在读取后进行处理和响应。最后，递归调用Start() 实现持续读取，因为是异步读，所以并不会超出栈的上限，除此之外还可以通过协程实现。

在lambda函数内部：

- 首先通过 got_text() 函数检查接收的 WebSocket 消息是否是文本数据，并将读取到的帧标记为文本类型；
- 将读取在缓存区的数据转换为string类型赋值给变量recv_data，然后清空缓存区_recv_buffer为下一次读取做准备；
- 将读取到的消息打印出来，然后将接受到的数据转发到 **AsyncSend()** 函数，处理发送响应；
- 递归调用Start()，等待下一条消息。

#### *6）AsyncSend*

```cpp
void Connection::AsyncSend(std::string msg) {
	std::unique_lock<std::mutex> lck_guard(_send_mtx);
	int que_len = _send_que.size();
	_send_que.push(msg);
	if (que_len > 0) {
		return;
	}
	lck_guard.unlock();

	auto self = shared_from_this();
	_ws_ptr->async_write(boost::asio::buffer(msg.c_str(), msg.size()),
		[self](boost::system::error_code ec, std::size_t msize) {
			try {
				if (!ec) {
					std::string send_msg;
					{
						std::lock_guard<std::mutex> lck_guard(self->_send_mtx);
						self->_send_que.pop();
						if (self->_send_que.empty()) {
							return;
						}
						
						send_msg = self->_send_que.front();
					}

					self->AsyncSend(std::move(send_msg));
				}
				else {
					std::cout << "async send err is " << ec.what() << std::endl;
					ConnectionMgr::GetInstance().RmvConnection(self->GetUid());
					return;
				}
			}
			catch (std::exception& e) {
				std::cerr << "async read exception is " << e.what() << std::endl;
				ConnectionMgr::GetInstance().RmvConnection(self->GetUid());
			}
		});
}
```

该函数用于服务器的回传，该函数通过队列和互斥锁来管理并发情况下的消息发送，确保不会出现多次并发写入的情况。如果消息正在发送中，则新消息会加入队列，待当前消息发送完毕后，队列中的下一个消息再被发送。

首先，加锁然后将消息插入队列，插入队列后立马解锁（**我们只需要在使用队列时加锁，不使用队列是解锁，避免不相干的逻辑占用锁**）。

然后，调用websocket的异步发送async_write()函数发送websocket消息，如果发送成功，调用lambda函数，释放已发送的上一条消息，然后加锁从队列中获取消息，获取完立刻解锁，最后递归调用 AsyncSend继续发送。

有没有发现，我这里使用了两种方法进行了解锁

第一种：

```cpp
	std::unique_lock<std::mutex> lck_guard(_send_mtx);
	int que_len = _send_que.size();
	_send_que.push(msg);
	if (que_len > 0) {
		return;
	}
	lck_guard.unlock();
```

第二种：

```cpp
	{
		std::lock_guard<std::mutex> lck_guard(_send_mtx);
		int que_len = _send_que.size();
		_send_que.push(msg);
		if (que_len > 0) {
			return;
		}
	}
```

这两种方法都可以，一种通过unique_lock手动加解锁，另一种通过‘{}’自动解锁。

### b. ConnectionMgr

该类用于管理Connection连接，通过将Connection连接添加到map中，手动的通过uuid删除或增加Connection连接。ConnectionMgr类是一个单例模式类，我这里使用了C++11新特性，用最简便的方法实现单例，还有另外一种方法可以实现单例模式，这种方法可以人为定义删除器，防止单例类的析构函数被意外主动调用。

另一种单例的实现可参考文章：

[爱吃土豆：网络编程（19）——C++使用asio协程实现并发服务器3 赞同 · 0 评论文章![img](/images/$%7Bfiilename%7D/icon-default-1730607206171-219.png)https://zhuanlan.zhihu.com/p/957175334](https://zhuanlan.zhihu.com/p/957175334)

```cpp
#pragma once
#include "Connection.h"
#include <boost/unordered_map.hpp>

class ConnectionMgr
{
private:
	ConnectionMgr(const ConnectionMgr&) = delete;
	ConnectionMgr& operator=(const ConnectionMgr&) = delete;
	ConnectionMgr();
	boost::unordered_map<std::string, std::shared_ptr<Connection>> _map_cons;
public:
	static ConnectionMgr& GetInstance();
	void AddConnection(std::shared_ptr<Connection> conptr);
	void RmvConnection(std::string);
};
```

单例的实现就不再叙述了，直接介绍成员变量：

- _map_cons：使用boost库的无序关联容器，类似于STL 的 std::unordered_map ，该容器通过哈希表实现，比有序容器的查找效率更快，但是不会主动排序。键是string类型的值，也就是connection的uuid；值是connection的智能指针。

成员函数的实现：

```cpp
ConnectionMgr::ConnectionMgr() {
	
}

void ConnectionMgr::AddConnection(std::shared_ptr<Connection> conptr) {
	_map_cons[conptr->GetUid()] = conptr;
}

void ConnectionMgr::RmvConnection(std::string uuid) {
	_map_cons.erase(uuid);
}

ConnectionMgr& ConnectionMgr::GetInstance() {
	static ConnectionMgr instance;
	return instance;
}
```

无需显示定义构造函数，AddConnection函数用于加入一个新的connection连接，RmvConnection用于移除一个connection连接，二者都是根据键进行操作。GetInstance静态成员函数用于返回ConnectionMgr类的唯一实例，因为从C++11及之后开始，**返回局部静态变量是线程安全的。**

### c. WebServer

该类用于发起TCP层客户端与服务器的TCP连接。

```cpp
#pragma once
#include "ConnectionMgr.h"

class WebServer
{
private:
	net::ip::tcp::acceptor _acceptor;
	net::io_context& _ioc;
public:
	WebServer(const WebServer&) = delete;
	WebServer& operator=(const WebServer&) = delete;
	WebServer(net::io_context& ioc, unsigned short port);
	void StartAccept();

};
```

WebServer不做成一个单例类，但我们同样不允许它被拷贝被赋值，所以将赋值运算符和复制构造函数均delete。然后定义StartAccept()函数用于服务器接收客户端的连接。

```cpp
WebServer::WebServer(net::io_context& ioc, unsigned short port) : 
	_ioc(ioc), _acceptor(ioc, net::ip::tcp::endpoint(net::ip::tcp::v4(), port)){
	std::cout << "Server start on port: " << port << std::endl;
}

void WebServer::StartAccept() {
	auto con_ptr = std::make_shared<Connection>(_ioc);
	_acceptor.async_accept(con_ptr->GetSocket(), [this, con_ptr](boost::system::error_code ec) {
		try {
			if (!ec) {
				con_ptr->AsyncAccept();
			}
			else {
				std::cerr << "acceptor async accept error is " << ec.what() << std::endl;
			}

			StartAccept();
		}
		catch (std::exception& e) {
			std::cerr << "async accept error is " << e.what() << std::endl;
		}
		});
}
```

WebServer类的构造函数和之前将的一样，将io_context的引用赋值给_ioc，将acceptor与ioc和服务器端点绑定，acceptor将从该端点监听客户端的连接。

StartAccept()于异步接受传入的连接，并启动connection连接的start函数。

**再提一遍，WebSocket 对象的 async_accept 和 acceptor对象的 async_accept 不一样。**

### **d. 编译的小问题**

如果直接编译的话会报错，显示生成的对象文件过大，超出默认限制。我们这里这里打开项目的属性->C/C++->命令行->输入‘/bigobj’。



![img](/images/$%7Bfiilename%7D/format,png-1730607206171-214.png)

## 3. 测试

通过该网站进行在线测试：

[websocket在线测试www.websocket-test.com/编辑![img](/images/$%7Bfiilename%7D/icon-default-1730607206171-219.png)https://link.zhihu.com/?target=http%3A//www.websocket-test.com/](https://link.zhihu.com/?target=http%3A//www.websocket-test.com/)

启动服务器，



![img](/images/$%7Bfiilename%7D/format,png-1730607206171-215.png)

编辑

打开测试网站，输入服务器域名，点击连接并发送“hello world”

```
ws://127.0.0.1:10086
```



![img](/images/$%7Bfiilename%7D/format,png-1730607206171-216.png)

网站显示：



![img](/images/$%7Bfiilename%7D/format,png-1730607206171-217.png)

程序显示：



![img](/images/$%7Bfiilename%7D/format,png-1730607206171-218.png)

一个简单的websocket服务器搭建成功。

## 4. 基于http实现的websocket

除了通过TCP长连接可以实现websocket服务器外，也可以通过http服务器升级为websocket，代码可参考博主恋恋风辰的代码仓库：

[恋恋风辰官方博客](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/2VWIJgH3zKEww0BpLnYQX0NMpQ9)
