---
title: 网络编程（1）——同步读写api
date: 2024-11-03 10:57:51
categories:
- 网络编程
tags: 
- 同步api
typora-root-url: ./..
---

## 一、day1

学习了服务器和客户端socket的建立、监听以及连接。

### （1）socket的监听和连接

服务端
 1）socket——创建socket对象。

2）bind——绑定本机ip+port。

3）listen——监听来电，若在监听到来电，则建立起连接。

4）accept——再创建一个socket对象给其收发消息。原因是现实中服务端都是面对多个客户端，那么为了区分各个客户端，则每个客户端都需再分配一个socket对象进行收发消息。

5）read、write——就是收发消息了。

对于客户端是这样的
 客户端
 1）socket——创建socket对象。

2）connect——根据服务端ip+port，发起连接请求。

3）write、read——建立连接后，就可发收消息了。

boost库网络编程的基本流程（阻塞）：

### 1）终端节点创建

终端节点代表一个[网络通信](https://zhida.zhihu.com/search?content_id=247993602&content_type=Article&match_order=1&q=网络通信&zhida_source=entity)的端点，由 IP 地址和端口号组成。该函数创建一个 TCP 客户端的终端节点。如果我们是客户端，我们可以通过对端的ip和端口构造一个endpoint，用这个endpoint和其通信。

客户端端点的建立：

```cpp
// 在客户端创建一个TCP/IP的终端（endpoint），将IP地址和端口号组合成一个用于通信的终端
int client_end_point() {
	std::string raw_ip_address = "127.4.8.1"; // 对端地址
	unsigned short port_num = 3333; // 对端端口号
	boost::system::error_code ec; // 错误码
	// 使用 boost::asio 库的 from_string 方法，将字符串形式的 IP 地址转换为 asio 支持的 IP 地址格式。
	// 如果转换失败，错误信息将存储在 ec 中。
	boost::asio::ip::address ip_address = boost::asio::ip::address::from_string(raw_ip_address, ec);
	if (ec.value() != 0) { // 转换失败
		std::cout << "Failed to parse the IP address. Error code = " << ec.value() << " .Message is " << ec.message();
		return ec.value();
	}

	// 创建一个 TCP 终端 ep，将 ip_address 和 port_num 绑定在一起。
	boost::asio::ip::tcp::endpoint ep(ip_address, port_num);

	return 0;
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

服务器端端点的建立：

```cpp
// 在服务器端创建一个TCP/IP的终端（endpoint），使用IPv6地址的通配符::绑定本地的所有地址
int server_end_point() {
	unsigned short port_num = 3333; // 定义服务器端使用的端口号 3333
	/*
	 定义一个ip_address，使用boost::asio库的address_v6::any()方法创建一个IPv6地址的通配符，
	 这个地址可以绑定到本地的任何地址上。通配符地址 (:: in IPv6) 允许服务器绑定到所有的可用地址。
	 换句话说，服务器可以监听来自任何接口(有线连接、无线连接、虚拟接口等)的传入连接。
	 服务器不需要指定一个具体的 IP 地址来监听，而是监听所有可用的 IP 地址。
	*/
	boost::asio::ip::address ip_address = boost::asio::ip::address_v6::any();
	// 创建一个 TCP/IP终端ep，将ip_address和port_num绑定在一起, 服务器将在所有的IPv6地址上监听端口3333的连接
	boost::asio::ip::tcp::endpoint ep(ip_address, port_num);
	return 0;
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 2）建立socket

客户端socket_v4的建立：

上下文iocontext->选择协议->生成socke->打开socket

```cpp
// 客户端socket_v4的建立
int create_tcp_socket() {
	boost::asio::io_context ioc; // 声明用于管理异步操作的I/O上下文
	// 声明socket，使用前面定义的io_context(ioc)来管理其I/O操作
	boost::asio::ip::tcp::socket sock(ioc);  

	boost::asio::ip::tcp protocol = boost::asio::ip::tcp::v4(); // 声明ipv4的协议
	boost::system::error_code ec; // 错误码
	sock.open(protocol, ec); // 判断是否创建成功
	if (ec.value() != 0) { // ec不为0时创建失败
		std::cout << "Failed to parse the IP address. Error code = " 
			<< ec.value() << " .Message is " << ec.message();
		return ec.value();
	}

	return 0;
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

上述socket只是通信的socket，需在服务端建立acceptor的socket，用于接收新的连接：

```cpp
// 创建一个服务器用于接受客户端连接的TCP监听器
int create_acceptor_socket() {
	boost::asio::io_context ios;
	boost::asio::ip::tcp::acceptor acceptor(ios);  // 新版本截止到此已经成功，老版本还需下面的操作

	boost::asio::ip::tcp protocol = boost::asio::ip::tcp::v4(); // 声明ipv4的协议
	boost::system::error_code ec; // 错误码
	acceptor.open(protocol, ec);
	if (ec.value() != 0) { // ec不为0时创建失败
		std::cout << "Failed to parse the IP address. Error code = "
			<< ec.value() << " .Message is " << ec.message();
		return ec.value();
	}

	return 0;

	// 生成一个acceptor，指定tcpv4协议，并接收所有发向3333端口的信息（新版本）
	// 创建 acceptor 并绑定它到一个特定的端点（IP 地址和端口）
	// boost::asio::ip::tcp::acceptor a(ios, asio::ip::tcp::endpoint(asio::ip::tcp::v4(), 3333));
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 3）绑定socket

对于acceptor类型的socket，服务器要将其绑定到指定的端点，所有连接这个端点的连接都可以被接收到

```cpp
// 创建一个服务器TCP连接接收器（acceptor），将其绑定到一个指定的端点（IP 地址和端口）
int bind_acceptor_socket() {
	unsigned short port_num = 3333;
	boost::asio::ip::tcp::endpoint ep(boost::asio::ip::address_v4::any(), port_num);
	boost::asio::io_context ios;
	boost::asio::ip::tcp::acceptor acceptor(ios, ep.protocol());
	boost::system::error_code ec; // 将接收器绑定到之前定义的端点 ep（包括通配符 IP 地址和端口 3333）
	acceptor.bind(ep, ec);
	if (ec.value() != 0) { // ec不为0时创建失败
		std::cout << "Failed to parse the IP address. Error code = "
			<< ec.value() << " .Message is " << ec.message();
		return ec.value();
	}
	return 0;
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 4）连接指定的端点

作为客户端可以连接服务器指定的端点

```cpp
// 创建TCP客户端socket，并尝试连接到指定的服务器端点（服务器的 IP 地址和端口）
int connect_to_end() {
	std::string raw_ip_address = "192.168.1.124"; // 服务器ip
	unsigned short port_num = 3333; // 服务器端口号
	try {
		// 生成一个端点，在客户端是连接，在服务器端是绑定
		boost::asio::ip::tcp::endpoint ep(boost::asio::ip::address::from_string(raw_ip_address), port_num);
		boost::asio::io_context ios;
		// socket对象连接到指定的服务器端点
		boost::asio::ip::tcp::socket sock(ios, ep.protocol());
		// 这里客户端是连接到指定端点，但在服务器中是绑定
		sock.connect(ep);
	}
	catch (boost::system::system_error& e) {
		std::cout << "Error occured! Error code = " << e.code()
			<< ". Message: " << e.what();
		return e.code().value();
	}

	return 0;
}

// 通过域名解析将主机名（服务器域名）转换为 IP 地址，然后尝试使用 TCP 套接字与服务器建立连接
int dns_connect_to_end() {
	std::string host = "llfc.club"; // 服务器域名
	std::string port_num = "3333"; // 服务器端口号
	boost::asio::io_context ios; // 上下文服务
	/*
	boost::asio::ip::tcp::resolver::query 用于指定 DNS 解析的查询参数。
	resolver_query 是一个查询对象，包含了主机名（host）和端口号（port_num）。
	boost::asio::ip::tcp::resolver::query::numeric_service 指定端口号是一个数字，
	而不是服务名称（例如，"http"、"ftp"等
	*/
	boost::asio::ip::tcp::resolver::query resolver_query(host, port_num, 
		boost::asio::ip::tcp::resolver::query::numeric_service); // 创建域名查询对象
	/*
	boost::asio::ip::tcp::resolver 是一个 DNS 解析器类，用于将域名解析为 IP 地址。
	resolver 是一个解析器对象，使用 io_context（ios）进行初始化。
	该解析器将用于执行 DNS 查询并获取服务器的 IP 地址。
	*/
	boost::asio::ip::tcp::resolver resolver(ios); // 创建解析器
	try {
		// 使用resolver.resolve(resolver_query)解析域名，返回一个迭代器it，
		// 指向与主机名和端口号匹配的端点列表
		boost::asio::ip::tcp::resolver::iterator it = resolver.resolve(resolver_query);
		boost::asio::ip::tcp::socket sock(ios);
		// 使用解析到的端点列表连接到服务器。它会自动尝试列表中的每一个端点，
		// 直到成功连接或所有端点都尝试完毕
		boost::asio::connect(sock, it);
	}
	catch(boost::system::system_error& e){
		std::cout << "Error occured! Error code = " << e.code()
			<< ". Message: " << e.what();
		return e.code().value();
	}
	return 0;
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 5）服务器接收连接

当有客户端连接时，服务器需要接收连接

```cpp
/ 创建一个 TCP 服务器端的接收器（acceptor），绑定到指定的 IP 地址和端口，并等待客户端连接。
// 一旦有新的客户端连接请求到达，服务器将接受这个连接
int accept_new_connection() {
	const int BACKLOG_SIZE = 30; // 监听队列的大小，服务器来不急监听时将链接放入队列
	unsigned short port_num = 3333;
	boost::asio::ip::tcp::endpoint ep(boost::asio::ip::address_v4::any(), port_num); // 端点
	boost::asio::io_context ios;
	try {
		boost::asio::ip::tcp::acceptor acceptor(ios, ep.protocol());
		acceptor.bind(ep);
		acceptor.listen(BACKLOG_SIZE);
		// 模拟创建了一个 TCP socket 对象，用于表示与客户端的连接
		boost::asio::ip::tcp::socket sock(ios);
		// 阻塞式地等待并接受一个新的连接请求，并将新的连接绑定到 sock。一旦有新的客户端连接请求，
		// acceptor 会接受它，并将新连接绑定到 sock
		acceptor.accept(sock);
	}
	catch(boost::system::system_error& e){
		std::cout << "Error occured! Error code = " << e.code()
			<< ". Message: " << e.what();

		return e.code().value();
	}
	return 0;
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

第一日总结：

1.网络层的“ip地址”可以唯一标识网络中的主机，而传输层的“协议+端口”可以唯一标识主机中的应用程序（进程）。利用[三元组](https://zhida.zhihu.com/search?content_id=247993602&content_type=Article&match_order=1&q=三元组&zhida_source=entity)（ip地址，协议，端口）就可以标识网络的进程，网络中的[进程通信](https://zhida.zhihu.com/search?content_id=247993602&content_type=Article&match_order=1&q=进程通信&zhida_source=entity)可以利用这个标志与其它进程进行交互。

2.什么是socket? socket起源于Unix，而Unix/Linux基本哲学之一就是“一切皆文件”，都可以用“打开open –> 读写write/read –> 关闭close”模式来操作。我的理解就是Socket就是该模式的一个实现，socket即是一种特殊的文件，一些socket函数就是对其进行的操作（读/写IO、打开、关闭）。

3.acceptor和socket有什么区别，分别有什么作用?

1)acceptor

**作用**: `acceptor` 是一个服务器端的对象，用于接收来自客户端的连接请求。

**功能**:

- **创建**: `acceptor` 对象通常在服务器端创建，它的作用是等待并接收客户端的连接请求。
- **绑定**: 通过 `acceptor.bind(endpoint)` 将其绑定到特定的 IP 地址和端口号，指定服务器要监听的地址。
- **监听**: 通过 `acceptor.listen(backlog_size)` 启动监听，`backlog_size` 是一个整数，表示连接[请求队列](https://zhida.zhihu.com/search?content_id=247993602&content_type=Article&match_order=1&q=请求队列&zhida_source=entity)的大小。
- **接受连接**: 通过 `acceptor.accept(socket)` 接受一个连接请求，并将连接绑定到一个新的 `socket` 对象上，`socket` 用于后续的数据传输。原因是现实中服务端都是面对多个客户端，那么为了区分各个客户端，则每个客户端都需再分配一个socket对象进行收发消息

```C++
boost::asio::ip::tcp::acceptor acceptor(io_context, endpoint);
acceptor.bind(endpoint);
acceptor.listen(backlog_size);
boost::asio::ip::tcp::socket socket(io_context);
acceptor.accept(socket);
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

2)socket

**作用**: `socket` 是用于实际数据传输的对象。它代表了一个[网络连接](https://zhida.zhihu.com/search?content_id=247993602&content_type=Article&match_order=1&q=网络连接&zhida_source=entity)的端点。

**功能**:

- **创建**: `socket` 对象在客户端和服务器端都可以创建。客户端用来发起连接，服务器端用来处理接收到的连接。
- **连接**: 在客户端，`socket` 通过 `socket.connect(endpoint)` 连接到服务器端的 `acceptor`。
- **接收和发送数据**: 一旦建立连接，`socket` 可以用来发送和[接收数据](https://zhida.zhihu.com/search?content_id=247993602&content_type=Article&match_order=1&q=接收数据&zhida_source=entity)，使用方法如 `socket.send()` 和 `socket.receive()`。
- **关闭连接**: 通过 `socket.close()` 关闭连接。

```c++
boost::asio::ip::tcp::socket socket(io_context);
socket.connect(endpoint);
boost::asio::write(socket, boost::asio::buffer("Hello, World!"));
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



## 二、day2

### 1）buffer

任何网络库都有提供buffer的数据结构，所谓buffer就是接收和发送数据时缓存数据的结构。

boost::asio提供了asio::mutable_buffer 和 asio::const_buffer这两个结构，他们是一段连续的空间，**首字节存储了后续数据的长度。**asio::mutable_buffer用于写服务，asio::const_buffer用于读服务。但是这**两个结构都没有被asio的api直接使用**。对于api的buffer参数，asio提出了MutableBufferSequence和ConstBufferSequence概念，他们是由多个asio::mutable_buffer和asio::const_buffer组成的。**也就是说boost::asio为了节省空间，将一部分连续的空间组合起来，作为参数交给api使用。**

我们可以理解为**MutableBufferSequence的数据结构为std::vector<asio::mutable_buffer>**



![img](/images/$%7Bfiilename%7D/format,png.png)

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑

buffer的结构

每个vector存储的都是**mutable_buffer的地址**，每个mutable_buffer的第一个字节表示数据的长度，后面跟着数据内容。

这么复杂的结构交给用户使用并不合适，所以asio提出了buffer()函数，**该函数接收多种形式的\**[字节流](https://zhida.zhihu.com/search?content_id=247993602&content_type=Article&match_order=1&q=字节流&zhida_source=entity)\**，该函数返回asio::mutable_buffers_1 o或者asio::const_buffers_1结构的对象**。如果传递给buffer()的参数是一个只读类型，则函数返回asio::const_buffers_1 类型对象。如果传递给buffer()的参数是一个可写类型，则返回asio::mutable_buffers_1 类型对象。

**asio::const_buffers_1和asio::mutable_buffers_1**是asio::mutable_buffer和asio::const_buffer的适配器，提供了符合MutableBufferSequence和ConstBufferSequence概念的接口，所以他们可以作为boost::asio的api函数的参数使用。

简单概括一下，我们**可以用buffer()函数生成我们要用的缓存存储数据**。比如boost的发送接口send要求的参数为ConstBufferSequence类型

```
template<typename ConstBufferSequence>
std::size_t send(const ConstBufferSequence & buffers);
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

如果将“Hello World”转换成该类型：

```cpp
// ConstBufferSequence类型构造函数1
void use_const_buffer() {
	std::string buf = "Hello World!";
	// 首先将buf转换为c风格字符串并获取首地址，然后将长度传至asio_buf函数，构造const_buffer
	boost::asio::const_buffer asio_buf(buf.c_str(), buf.length());
        // 构造ConstBufferSequence类型
	std::vector< boost::asio::const_buffer> buffers_sequence;
	// 将构造的const_buffer放入const_buffer容器中
	buffers_sequence.push_back(asio_buf);
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

 实际中我们使用并没有这么复杂，简化上述函数，用buffer函数将其转化为send需要的参数类型，output_buf可以直接传递给send接口使用，充当ConstBufferSequence类型：

```C++
// ConstBufferSequence类型构造函数2
void use_buffer_str() {
        // buffer函数返回asio::mutable_buffers_1 o或者asio::const_buffers_1结构的对象
	boost::asio::const_buffers_1 optput_buf = boost::asio::buffer("hello world");
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



也可以将数组转换为send需要的类型（ConstBufferSequence）：

```C++
// 使用动态分配的字符数组作为缓冲区，并使用Boost.Asio库将其包装成一个可用于异步 
// I / O 操作的缓冲区对象
void use_buffer_array() {
	// 使用无符号整数类型size_t作为缓冲区大小
	const size_t BUF_SIZE_BYTES = 20;
	// std::unique_ptr<char[]> 是一个智能指针，用于管理动态分配的数组的生命周期。它在不再需要时自动释放内存，
	// 避免了手动 delete[] 的需要
	std::unique_ptr<char[]> buf(new char[BUF_SIZE_BYTES]);
	// boost::asio::buffer 函数将一个原始指针（在此例中是一个指向字符数组的指针）和其大小（以字节为单位）包装
	// 成一个可用于异步 I/O 操作的 boost::asio::mutable_buffer 对象
	// buf.get() 返回指向智能指针 buf 所管理的字符数组的原始指针，并将其转换为void*类型
	auto input_buf = boost::asio::buffer(static_cast<void*>(buf.get()), BUF_SIZE_BYTES);
}
```

对于流式操作，我们可以用streambuf，将输入输出流和streambuf绑定，可以实现流式输入和输出：

```
void use_stream_buffer() {
    asio::streambuf buf;
    std::ostream output(&buf);
    // Writing the message to the stream-based buffer.
    output << "Message1\nMessage2";
    // Now we want to read all data from a streambuf
    // until '\n' delimiter.
    // Instantiate an input stream which uses our 
    // stream buffer.
    std::istream input(&buf);
    // We'll read data into this string.
    std::string message1;
    std::getline(input, message1);
    // Now message1 string contains 'Message1'.
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 2）**同步写write_some**

boost::asio提供了几种同步写的api，write_some可以每次向**指定的空间（socket）**写入固定的字节数，如果写缓冲区满了，就只写一部分，返回写入的字节数。举个栗子，用户buffer发送缓冲区长度为5，TCP发送缓冲区长度为12，虽然下面的write_some一开始希望发送buf.length() - total_bytes_written = 12 - 0=12个长度，但是用户buffer只有5个，所以只能先发送5个，剩下的7个循环继续发送：

```c++
// 写函数
void write_to_socket(boost::asio::ip::tcp::socket& sock) {
	std::string buf = "Hello World!";
	std::size_t total_bytes_wtitten = 0; // 已发送的字节数
	// 循环发送
	// write_some 返回每次写入的字节数, 该函数的输入类型是ConstBufferSequence
	/*
	持续写入：使用 while 循环持续尝试写入数据，直到所有数据都被写入完毕。
	部分写入：sock.write_some 尝试将部分数据写入到 sock 中，这个方法不是一次性写入所有数据，
			  而是可能写入一部分，因此需要用 total_bytes_written 变量来跟踪已写入的字节数。
	循环写入：每次写入后，更新 total_bytes_written，使写入指针向前移动，直到所有数据被写入。
	*/
	while (total_bytes_wtitten != buf.length()) {
		// 举个栗子，用户buffer发送缓冲区长度为5，TCP发送缓冲区长度为12，虽然下面的write_some一开始希望
		// 发送buf.length() - total_bytes_written = 12 - 0=12个长度，但是用户buffer只有5个，所以只能
		// 先发送5个，剩下的7个循环继续发送
		total_bytes_wtitten += sock.write_some(boost::asio::buffer(buf.c_str() + total_bytes_wtitten,
			buf.length() - total_bytes_wtitten));
	}
}

// 客户端发送函数
int send_data_by_write_some() {
	std::string raw_ip_address = "127.0.0.1";
	unsigned short port_num = 3333;
	try {
		boost::asio::ip::tcp::endpoint ep(boost::asio::ip::address::from_string(raw_ip_address), port_num);
		//boost::asio::io_service ios
		boost::asio::io_context ioc;
		boost::asio::ip::tcp::socket sock(ioc, ep.protocol());
		sock.connect(ep);
                // 发送信息至ep端点
		write_to_socket(sock);
	}
	catch(boost::system::system_error& e){
		std::cout << "Error occured! Error code = " << e.code()
			<< ". Message: " << e.what();

		return e.code().value();
	}
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 3）同步写send

write_some使用起来比较麻烦，需要**多次调用（while）**，asio提供了send函数。send函数会**一次性**将buffer中的内容发送给对端，如果有部分字节因为发送缓冲区满无法发送，则阻塞等待，直到发送缓冲区可用，则继续发送完成。

```C++
int send_data_by_send() {
	std::string raw_ip_address = "127.0.0.1";
	unsigned short port_num = 3333;
	std::string buf = "Hello World!";
	try {
		boost::asio::ip::tcp::endpoint ep(boost::asio::ip::address::from_string(raw_ip_address), port_num);
		//boost::asio::io_service ios
		boost::asio::io_context ioc;
		boost::asio::ip::tcp::socket sock(ioc, ep.protocol());
		sock.connect(ep);
		// 不使用write_to_socket(sock)函数，使用sock的send函数
		// 将buf长度为length的内容发送至socket,返回size_t类型，如果成功发送，
		// bytes_sent 可能等于 buf.length()，即所有数据都成功发送；如果返回值小于 
		// buf.length()，表示只有部分数据被发送；如果发送失败，send 将抛出一个 
		// boost::system::system_error 异常。
		// send 是一个同步操作，所以它会阻塞直到所有数据被成功发送或发生错误，
		// 这意味着在 send 操作完成之前，程序将不会继续执行后面的代码。
		int send_length = sock.send(boost::asio::buffer(buf.c_str(), buf.length())); 
		// send_length < 0:出现系统错误
		// send_length == 0:对端关闭
		// send_length = buf.length() > 0:发送成功
		if (send_length <= 0) // 发送失败
			return 0;
	}
	catch (boost::system::system_error& e) {
		std::cout << "Error occured! Error code = " << e.code()
			<< ". Message: " << e.what();

		return e.code().value();
	}
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 4）同步写write

类似send方法，asio还提供了一个write函数，可以**一次性**将所有数据发送给对端，如果发送缓冲区满了则阻塞，直到发送缓冲区可用，将数据发送完成。

```C++
int send_data_by_write() {
	std::string raw_ip_address = "127.0.0.1";
	unsigned short port_num = 3333;
	std::string buf = "Hello World!";
	try {
		boost::asio::ip::tcp::endpoint ep(boost::asio::ip::address::from_string(raw_ip_address), port_num);
		//boost::asio::io_service ios
		boost::asio::io_context ioc;
		boost::asio::ip::tcp::socket sock(ioc, ep.protocol());
		sock.connect(ep);
		// 和sock.send()函数类似
		int send_length = boost::asio::write(sock, boost::asio::buffer(buf.c_str(), buf.length()));
		// send_length < 0:出现系统错误
		// send_length == 0:对端关闭
		// send_length = buf.length() > 0:发送成功
		if (send_length <= 0) // 发送失败
			return 0;
	}
	catch (boost::system::system_error& e) {
		std::cout << "Error occured! Error code = " << e.code()
			<< ". Message: " << e.what();

		return e.code().value();
	}
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 5）同步读read_some

同步读和同步写类似，提供了读取指定字节数的接口read_some，但read_some可能会被多次调用，比较麻烦。

```C++
// 使用 Boost.Asio 库从 TCP 套接字 sock 读取固定大小（MESSAGE_SIZE）的数据，
// 并返回一个 std::string 对象，表示读取到的数据
std::string read_from_socket(boost::asio::ip::tcp::socket& sock) {
	const unsigned char MESSAGE_SIZE = 7;
	char buf[MESSAGE_SIZE];
	std::size_t total_byter_read = 0;
	while (total_byter_read != MESSAGE_SIZE) {
		total_byter_read += sock.read_some(boost::asio::buffer(buf + total_byter_read, MESSAGE_SIZE - total_byter_read));
	}

	return std::string(buf, total_byter_read);
}

// 从一个指定的 TCP 服务器（在 127.0.0.1:3333 上）建立连接，并调用 read_from_socket 
// 函数来同步读取数据
int read_data_by_read_some() {
	std::string raw_ip_address = "127.0.0.1";
	unsigned short port_num = 3333;
	try {
		boost::asio::ip::tcp::endpoint ep(boost::asio::ip::address::from_string(raw_ip_address), port_num);
		boost::asio::io_context ioc;
		boost::asio::ip::tcp::socket sock(ioc, ep.protocol());
		sock.connect(ep);
		// 调用同步读函数，读取来自连接套接字的数据
		read_from_socket(sock); // 同步读
	}
	catch (boost::system::system_error& e) {
		std::cout << "Error occured! Error code = " << e.code()
			<< ". Message: " << e.what();

		return e.code().value();
	}
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 6）同步读receive

可以一次性同步接收对方发送的数据：

```C++
int read_data_by_receive() {
	std::string raw_ip_address = "127.0.0.1";
	unsigned short port_num = 3333;
	try {
		boost::asio::ip::tcp::endpoint ep(boost::asio::ip::address::from_string(raw_ip_address), port_num);
		boost::asio::io_context ioc;
		boost::asio::ip::tcp::socket sock(ioc, ep.protocol());
		sock.connect(ep);
		const unsigned char BUFF_SIZE = 7;
		char buffer_receive[BUFF_SIZE];
		// 接收数据，并获取接收到的字节数
		int receive_length = sock.receive(boost::asio::buffer(buffer_receive, BUFF_SIZE));
		if (receive_length <= 0) {
			std::cout << "receive failed " << std::endl;
		}
			
	}
	catch (boost::system::system_error& e) {
		std::cout << "Error occured! Error code = " << e.code()
			<< ". Message: " << e.what();

		return e.code().value();
	}
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 7）同步读read

可以一次性同步读取对方发送的数据：

```C++
int read_data_by_read() {
	std::string raw_ip_address = "127.0.0.1";
	unsigned short port_num = 3333;
	try {
		boost::asio::ip::tcp::endpoint ep(boost::asio::ip::address::from_string(raw_ip_address), port_num);
		boost::asio::io_context ioc;
		boost::asio::ip::tcp::socket sock(ioc, ep.protocol());
		sock.connect(ep);
		const unsigned char BUFF_SIZE = 7;
		char buffer_receive[BUFF_SIZE];
		int receive_length = boost::asio::read(sock, boost::asio::buffer(buffer_receive, BUFF_SIZE));
		if (receive_length <= 0) {
			std::cout << "receive failed " << std::endl;
		}

	}
	catch (boost::system::system_error& e) {
		std::cout << "Error occured! Error code = " << e.code()
			<< ". Message: " << e.what();

		return e.code().value();
	}
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 8）读取直到指定字符

我们可以一直读取，直到读取指定字符结束：

```C++
std::string  read_data_by_until(boost::asio::ip::tcp::socket& sock) {
	boost::asio::streambuf buf;
	// Synchronously read data from the socket until
	// '\n' symbol is encountered.  
	boost::asio::read_until(sock, buf, '\n');
	std::string message;
	// Because buffer 'buf' may contain some other data
	// after '\n' symbol, we have to parse the buffer and
	// extract only symbols before the delimiter. 
	std::istream input_stream(&buf);
	std::getline(input_stream, message);
	return message;
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

总结：

1.c++中，sock.receive和boost::asio::read有什么区别？

同：`sock.receive` 和 `boost::asio::read` 都用于网络编程中的数据接收操作

不同：1）`sock.receive` 相对比较简单，只是接收数据，并返回接收的数据长度或错误代码。它不具备高级功能，如超时处理、数据分片重组、异步操作等。`boost::asio::read` 提供了更高层次的功能封装。例如，它可以在接收到特定数量的字节后才返回结果，还可以支持异步操作 (`async_read`)，使得程序能够同时处理多个 I/O 操作。2）`sock.receive` 本身是一个阻塞调用，如果需要非阻塞行为，需要使用特定的套接字选项来配置。`boost::asio::read` 支持同步和异步两种方式，通过 `boost::asio::io_context` 及回调机制，可以轻松实现异步 I/O 操作。
