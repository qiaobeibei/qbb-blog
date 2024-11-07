---
title: 网络编程（3）——异步读写api
date: 2024-11-03 11:08:44
categories:
- C++
- 网络编程
tags: 
- 异步api
typora-root-url: ./..
---

# 四、day4

今天学习异步读写操作的常用api

## 1）MsgNode类

封装一个Node结构，用来管理要发送和接收的数据，该结构包含数据域首地址，数据的总长度，以及已经处理的长度(已读的长度或者已写的长度)

```C++
class MsgNode {
public:
	int _total_len; // 数据的总长度
	int _cur_len; // 已经处理的长度(已读的长度或者已写的长度)
	char* _msg; // 数据域首地址

	MsgNode(const char* msg, int total_len) :_total_len(total_len), _cur_len(0) { // 构造写节点
		_msg = new char[total_len];
		memcpy(_msg, msg, total_len);
	}
	MsgNode(int total_len) : _total_len(total_len), _cur_len(0) { // 构造读节点
		_msg = new char[total_len];
	}
	~MsgNode() {
		delete[] _msg;
	}
};
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

## 2）Session类

定义Session类，表示服务器处理[客户端](https://zhida.zhihu.com/search?content_id=248112810&content_type=Article&match_order=1&q=客户端&zhida_source=entity)连接的管理类

```C++
class Session
{
private:
	// 按队列的方式（FIFO）存储消息节点，确保数据按发送顺序被处理
	std::queue<std::shared_ptr<MsgNode>> _send_queue;
	// 存储socket，负责与对端进行数据交换
	std::shared_ptr<asio::ip::tcp::socket> _socket;
	// 发送消息节点
	std::shared_ptr<MsgNode> _send_node;
	// true:发送操作尚未完成，新的发送请求需排队；false:发送完成
	bool _send_pending;
	// 接收消息节点
	std::shared_ptr<MsgNode> _recv_node;
	// true:接收操作尚未完成，新的接收请求需排队；false:接收完成
	bool _recv_pending;

public:
	// session只需接收参数socket进行数据交互
	Session(std::shared_ptr<asio::ip::tcp::socket> socket);
	// 连接到指定对端
	void Connect(const asio::ip::tcp::endpoint& ep);

	void WriteCallBackErr(const boost::system::error_code& ec, std::size_t bytes_transferred,
		std::shared_ptr<MsgNode>);
	void WriteToSocketErr(const std::string buf);

	void WriteCallBack(const boost::system::error_code& ec, std::size_t bytes_transferred);
	void WriteToSocket(const std::string buf);

	void WriteAllToSocket(const std::string buf);
	void WriteAllCallBack(const boost::system::error_code& ec, std::size_t bytes_transferred);

	void ReadFromSocket();
	void ReadCallBack(const boost::system::error_code& ec, std::size_t bytes_transferred);

	void ReadAllFromSocket();
	void ReadAllCallBack(const boost::system::error_code& ec, std::size_t bytes_transferred);
};
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

接下来详细介绍Session类中的每一个api。

------

### **1.WriteCallBackErr()**

**WriteCallBackErr** 函数是一个[回调函数](https://zhida.zhihu.com/search?content_id=248112810&content_type=Article&match_order=1&q=回调函数&zhida_source=entity)，在**每次异步写操作完成后调用**。它的作用是检查是否所有数据都已发送，如果没有，则继续发送剩余的数据。

```C++
void Session::WriteCallBackErr(const boost::system::error_code& ec, std::size_t bytes_transferred,
	std::shared_ptr<MsgNode> msg_node) {
	// bytes_transferred: 本次异步写操作已经成功传输的字节数
	// msg_node->_cur_len: 在之前的写操作中已经成功传输的字节数。
	// msg_node->_total_len: 整个消息的总长度
	// 当前已传输的字节数(bytes_transferred)和已发送的数据长度(msg_node->_cur_len)的总和是否小于消息的总长度(msg_node->_total_len)
	// 判断是否继续写入
	if (bytes_transferred + msg_node->_cur_len < msg_node->_total_len) {
		_send_node->_cur_len += bytes_transferred; // 更新已成功传输的字节长度
		// 使用异步写操作，将剩余数据写入套接字
		// 发送数据的起始地址是已发送数据的尾端，长度是剩余未发送数据量
		this->_socket->async_write_some(asio::buffer(_send_node->_msg + _send_node->_cur_len,
			_send_node->_total_len - _send_node->_cur_len),
			// 绑定回调函数，如果写操作不完整，将继续调用该函数，直至所有数据都被写入
			std::bind(&Session::WriteCallBackErr, this, std::placeholders::_1, std::placeholders::_2,
				_send_node));
	}
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

难点是理解_send_node和msg_node的区别、bind()函数的应用、回调函数中的ec、bytes_transferred等参数为什么不用显示更新以及在bind函数中this的作用，这些问题在总结中都会回答。

### **2.WriteToSocketErr()**

**WriteToSocketErr**函数负责**执行一次异步写操作**，但它不负责检查和处理数据是否全部发送完毕，不需要判断是否发完，当这次写操作完成后，回调函数 **WriteCallBackErr** 会被调用，用来判断信息是否发送完全。所以在**WriteToSocketErr**函数中不需要像**WriteCallBackErr** 函数一样判断信息是否发送完全。

```C++
// 开始一次异步写操作，但它不负责检查和处理数据是否全部发送完毕，不需要判断是否发完
// 当这次写操作完成后，回调函数 WriteCallBackErr 会被调用
void Session::WriteToSocketErr(const std::string buf) {
	// buf：要发送的错误消息内容
	// 将buf中的内容保存至MsgNode节点，并赋予私有成员_send_node
	_send_node = std::make_shared<MsgNode>(buf.c_str(), buf.length());
	// 因为这里不需要负责检查是否发完，所以数据的尾端就是首段，长度就是数据长度
	this->_socket->async_write_some(asio::buffer(_send_node->_msg, _send_node->_total_len),
		std::bind(&Session::WriteCallBackErr,this,std::placeholders::_1,std::placeholders::_2,
		_send_node));
}
```

------

### **3.WriteCallBack()**

WriteCallBack函数虽然和WriteCallBackErr函数一样都是回调函数，但是有一定的区别，我会在在总结中进行解释。

```C++
void  Session::WriteCallBack(const boost::system::error_code& ec, std::size_t bytes_transferred) {
	// 写操作是否出错
	if (ec.value() != 0) {
		std::cout << "Error, code is " << ec.value() << " .Message is " << ec.message();
		return; // 出现错误，退出回调函数
	}

	auto& send_data = _send_queue.front(); // 获取发送队列中的第一个消息
	send_data->_cur_len += bytes_transferred; // 更新当前消息已发送的字节数

	if (send_data->_cur_len < send_data->_total_len) { // 当前消息未全部发送
		this->_socket->async_write_some(asio::buffer(send_data->_msg + send_data->_cur_len,
			send_data->_total_len - send_data->_cur_len),
			std::bind(&Session::WriteCallBack, this, std::placeholders::_1, std::placeholders::_2));
		return; // 当前消息未发送完毕，退出回调函数
	}

	_send_queue.pop(); // 当前消息已全部发送完毕，将其移出队列
	if (_send_queue.empty()) { // 检查发送队列是否为空
		_send_pending = false; // 队列为空，标记发送操作为非挂起状态
	}
	else {
		auto& send_data = _send_queue.front(); // 队列不为空，开始发送下一个消息
		this->_socket->async_write_some(asio::buffer(send_data->_msg + _send_node->_cur_len,
			send_data->_total_len - send_data->_cur_len),
			std::bind(&Session::WriteCallBack, this, std::placeholders::_1, std::placeholders::_2));
	}
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### **4.WriteToSocket()**

Session::WriteToSocket 函数的主要作用是将一个新的消息添加到发送队列中，并启动异步写操作。如果已经有挂起的发送操作，它不会启动新的异步写操作。这个设计确保了异步写操作不会重叠，从而保证数据的有序发送。

```C++
void  Session::WriteToSocket(const std::string buf) {
	// 将新的消息添加到发送队列中
	_send_queue.emplace(new MsgNode(buf.c_str(), buf.length()));

	if (_send_pending) return; // 如果已经有挂起的发送操作，直接返回，不启动新的异步写操作
	
	 // 启动异步写操作，发送当前消息
	this->_socket->async_write_some(asio::buffer(buf),
		std::bind(&Session::WriteCallBack, this, std::placeholders::_1, std::placeholders::_2));
	_send_pending = true; // 标记当前有挂起的发送操作
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

但注意到_send_pending被挂起时会阻止下一个加入队列消息的发送操作。具体区别我在总结部分作了解释。

------

### **5.WriteAllToSocket()**

async_write_some函数不能保证每次回调函数触发时发送的长度为要总长度，这样我们每次都要在回调函数判断发送数据是否完成，asio提供了一个更简单的发送函数**async_send**，这个函数**在发送的长度未达到我们要求的长度时就不会触发回调**，所以触发回调函数时要么时发送出错了要么是发送完成了,其内部的实现原理就是帮我们**不断的调用async_write_some直到完成发送**，所以async_send**不能**和async_write_some混合使用，我们基于async_send封装另外一个发送函数.

```C++
void Session::WriteAllToSocket(const std::string buf) {
	// 将新的消息添加到发送队列中
	_send_queue.emplace(new MsgNode(buf.c_str(), buf.length()));
	// 如果已经有挂起的发送操作，直接返回，不启动新的异步写操作
	if (_send_pending) 
		return;
	
	// 启动异步发送操作，发送当前消息
	this->_socket->async_send(asio::buffer(buf), std::bind(&Session::WriteAllCallBack, this,
		std::placeholders::_1, std::placeholders::_2));
	_send_pending = true;
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

async_send是否发送完全由错误码ec判断，当ec不为0时，发送错误，当ec=0时，发送完全。

### **6.WriteAllCallBack()**

由于async_send发送函数的基本原理，在该回调函数中，只有发送成功和发送错误两种可能性。

```C++
void Session::WriteAllCallBack(const boost::system::error_code& ec, std::size_t bytes_transferred) {
	// 如果发生错误，打印错误信息并返回
	if (ec.value() != 0) {
		std::cout << "Error occured! Error code = " << ec.value() << " .Message: " << ec.message();
		return;
	}

	// 因为async_send在发送的长度未达到我们的要求时就不会触发回调函数，所以此时要么发送失败，要么发送完全
	// 从发送队列中移除已经成功发送的消息
	_send_queue.pop();
	// 如果发送队列为空，设置_send_pending为false并返回
	if (_send_queue.empty()) {
		_send_pending = false;
		return;
	}
	else {
		// 获取队列中的下一个消息
		auto& send_data = _send_queue.front();
		// 启动异步写操作，发送下一个消息
		this->_socket->async_write_some(asio::buffer(send_data->_msg + _send_node->_cur_len,
			send_data->_total_len - send_data->_cur_len),
			std::bind(&Session::WriteCallBack, this, std::placeholders::_1, std::placeholders::_2));
	}
}
```

------

### **7.ReadFromSocket()**

接下来介绍异步读操作，异步读操作和异步的写操作类似同样有async_read_some和async_receive函数，前者触发的回调函数获取的读数据的长度可能会小于要求读取的总长度，后者触发的回调函数读取的数据长度等于读取的总长度或读取错误。

```C++
// 从套接字中异步读取数据
void Session::ReadFromSocket() {
	if (_recv_pending) // 是否已经有挂起的异步读取操作
		return;
	// 分配一个新的接收缓冲区，用于存储即将读取到的数据
	_recv_node = std::make_shared<MsgNode>(RECVSIZE);
	// 启动异步读取操作，从套接字中读取数据
	// 该回调函数无论异步操作是否完成，都会被调用
	_socket->async_read_some(asio::buffer(_recv_node->_msg, _recv_node->_total_len),
		bind(&Session::ReadCallBack, this, std::placeholders::_1, std::placeholders::_2));
	// 挂起
	_recv_pending = true;
}
```

### **8.ReadCallBack()**

/异步读取的回调函数，在调用 async_read_some 进行数据读取后触发。回调函数根据读取的字节数来判断数据是否全部接收完毕，如果数据未完全接收则继续读取，直到所有数据都被读取完成。

```C++
void Session::ReadCallBack(const boost::system::error_code& ec, std::size_t bytes_transferred) {
	// 更新接收到的数据长度
	_recv_node->_cur_len += bytes_transferred;
	// 检查是否已经读取到足够的数据
	if (_recv_node->_cur_len < _recv_node->_total_len) {
		_socket->async_read_some(asio::buffer(_recv_node->_msg +_recv_node->_cur_len,
			_recv_node->_total_len-_recv_node->_cur_len),
			bind(&Session::ReadCallBack, this, std::placeholders::_1, std::placeholders::_2));
		return; // 未读取完成，退出回调函数
	}
	// 读取操作已经完成，将标志位还原
	_recv_pending = false;
	// 将_recv_node置为 nullptr，释放该消息节点，其由shared_ptr 管理，当引用计数为0时，自动释放空间
	_recv_node = nullptr;
}
```

------

### **9.ReadAllFromSocket()**

async_receive内部就是执行多次async_read_some，async_receive只有当数据全部读完或者读取错误才会调用回调函数，基于async_receive再封装一个[接收数据](https://zhida.zhihu.com/search?content_id=248112810&content_type=Article&match_order=1&q=接收数据&zhida_source=entity)的函数，同样**async_read_some和async_receive不能混合使用**

```C++
void Session::ReadAllFromSocket() {
	if (_recv_pending) {
		return;
	}

	_recv_node = std::make_shared<MsgNode>(RECVSIZE);
	_socket->async_receive(asio::buffer(_recv_node->_msg,
		_recv_node->_total_len),
		std::bind(&Session::ReadAllCallBack, this, std::placeholders::_1, std::placeholders::_2));
	_recv_pending = true;
}

void Session::ReadAllCallBack(const boost::system::error_code& ec, std::size_t bytes_transferred) {
	// 该回调函数只触发一次，且触发后数据已经接受完全
	_recv_node->_cur_len += bytes_transferred; // 更新长度
	_recv_node = nullptr;
	_recv_pending = false;
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

------

## 总结

1. **同步和异步的区别？**

同步就是指一个进程在执行某个请求的时候，若该请求需要一段时间才能返回信息，那么这个进程将会一直等待下去，直到收到返回信息才继续执行下去;同步就相当于是 当客户端发送请求给服务端，在等待服务端响应的请求时，客户端不做其他的事情。当服务端做完了才返回到客户端。这样的话客户端需要一直等待。用户使用起来会有不友好。

异步是指进程不需要一直等下去，而是继续执行下面的操作，不管其他进程的状态。当有消息返回时系统会通知进程进行处理，这样可以提高执行的效率。异步就相当于当客户端发送给服务端请求时，在等待服务端响应的时候，客户端可以做其他的事情，这样节约了时间，提高了效率。

**2.bind()函数**

将[原函数](https://zhida.zhihu.com/search?content_id=248112810&content_type=Article&match_order=1&q=原函数&zhida_source=entity)的几个参数通过bind绑定传值，返回一个新的可调用对象

```C++
//绑定全局函数
auto newfun1 = bind(globalFun2, placeholders::_1, placeholders::_2, 98, "worker");
//相当于调用globalFun2("Lily",22, 98,"worker");
newfun1("Lily", 22);
//多传参数没有用，相当于调用globalFun2("Lucy",28, 98,"worker");
newfun1("Lucy", 28, 100, "doctor");
auto newfun2 = bind(globalFun2, "zack", placeholders::_1, 100, placeholders::_2);
//相当于调用globalFun2("zack",33,100,"engineer");
newfun2(33, "engineer");
auto newfun3 = bind(globalFun2, "zack", placeholders::_2, 100, placeholders::_1);
newfun3("coder", 33);
```

**3. 在WriteCallBackErr()函数中，_send_node和msg_node有什么区别？**

**`_send_node`**

- **定义**: 是 `Session` 类的一个私有成员变量（`std::shared_ptr<MsgNode> _send_node;`）。
- **作用**: 用于存储当前正在发送的数据块（消息节点）。
- **目的**: 它是`Session`对象的一部分，表示整个会话（或连接）过程中正在被发送的那条消息。`_send_node` 在类的生命周期内可以被多次使用和更新，以便于保存和管理当前需要发送的数据块。

**`msg_node`**

- **定义**: 是 `WriteCallBackErr` 函数的一个参数（`std::shared_ptr<MsgNode> msg_node`）。
- **作用**: 代表当前函数回调时传递进来的消息节点，即正在处理的消息。
- **目的**: 它是一个局部变量，用于在回调函数中表示当前异步写操作处理的数据节点。它的值可能来自 `_send_node`，也可能是其他数据源。

**4.在WriteCallBackErr()中，bind()绑定的回调函数为什么没有显示的更新ec、bytes_transferred，而只是用占位符1、2代替？为什么需要绑定this?**

**1）**在 WriteCallBackErr 回调函数中，bytes_transferred 参数不用被显式更新，因为 bytes_transferred 的值是由异步写操作的结果自动传递给回调函数。在异步写操作的回调函数中，bytes_transferred 是只读的，它表示当前这次写操作成功写入的字节数；在每次异步写操作完成后，boost::asio 会重新调用回调函数 WriteCallBackErr，并提供新的 bytes_transferred 值（表示该次操作的写入字节数）。

回调函数的调用流程：

- 每次调用 `async_write_some` 时，会触发一次异步写操作。

- 当写操作完成时，

  ```C++
  WriteCallBackErr
  ```

   回调函数会被调用，并且 

  ```C++
  boost::asio
  ```

   会把当前写操作的结果传递给回调函数，包括： 

  - **`ec`**: 错误代码（如果没有错误，则表示操作成功）。
  - **`bytes_transferred`**: 该次操作实际传输的字节数。

- `WriteCallBackErr` 函数会根据 `bytes_transferred` 和当前消息的状态决定是否需要继续发送数据。

代码演示：

```C++
void Session::WriteCallBackErr(const boost::system::error_code& ec, std::size_t bytes_transferred,
    std::shared_ptr<MsgNode> msg_node) {
    if (bytes_transferred + msg_node->_cur_len < msg_node->_total_len) {
        _send_node->_cur_len += bytes_transferred; // 更新当前发送的数据长度

        this->_socket->async_write_some(
            asio::buffer(_send_node->_msg + _send_node->_cur_len, _send_node->_total_len - _send_node->_cur_len),
            std::bind(&Session::WriteCallBackErr, this, std::placeholders::_1, std::placeholders::_2, _send_node));
    }
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

第一步：异步写操作的开始：async_write_some 被调用，开始一个异步写操作，将 _send_node 中的数据写入套接字。

第二步：异步操作完成时调用回调函数：当异步写操作完成时，boost::asio 自动调用 WriteCallBackErr 回调函数。bytes_transferred 被设置为该次操作成功写入的字节数。

第三步：检查是否需要继续写入：WriteCallBackErr 函数检查 bytes_transferred + msg_node->_cur_len 是否小于 msg_node->_total_len。如果是，则说明数据还未全部写完，需要继续发送。

第四步：继续异步写操作：调用 async_write_some 继续写入剩余的数据，并绑定同一个回调函数 WriteCallBackErr。

**2）**在回调函数 WriteCallBackErr 中绑定 this 是因为 WriteCallBackErr() 是 Session 类的一个[成员函数](https://zhida.zhihu.com/search?content_id=248112810&content_type=Article&match_order=1&q=成员函数&zhida_source=entity)，而不是一个普通的全局或静态函数。成员函数在调用时需要一个对象实例来访问类的成员变量和成员函数。

```C++
std::bind(&Session::WriteCallBackErr, this, std::placeholders::_1, std::placeholders::_2, _send_node)
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

- `std::bind` 是用于将函数或成员函数与特定参数绑定在一起的[标准库](https://zhida.zhihu.com/search?content_id=248112810&content_type=Article&match_order=1&q=标准库&zhida_source=entity)工具。
- `&Session::WriteCallBackErr` 表示你要绑定的函数是 `Session`[类的成员函数](https://zhida.zhihu.com/search?content_id=248112810&content_type=Article&match_order=1&q=类的成员函数&zhida_source=entity)。
- `this` 指针是指向当前对象实例的指针。它将当前的 `Session` 实例与 `WriteCallBackErr` 函数绑定，这样在调用时，`WriteCallBackErr` 知道要操作哪个 `Session` 对象的成员变量。

**5.WriteCallBack函数和WriteCallBackErr函数的区别？**

**1）WriteCallBackErr 函数**

**功能**

- 处理单个消息的异步写操作。
- 检查当前消息是否已全部发送，如果没有则继续发送剩余部分。

**特点**

1. **处理单条消息**：`WriteCallBackErr` 函数每次只处理一条消息，不涉及[消息队列](https://zhida.zhihu.com/search?content_id=248112810&content_type=Article&match_order=1&q=消息队列&zhida_source=entity)。
2. **错误处理**：如果出现错误，函数直接输出错误信息并返回。
3. **继续发送**：如果当前消息未完全发送，则继续发送剩余部分，直到消息发送完毕。

2）**WriteCallBack 函数**

**功能**

- 处理消息队列中的异步写操作。
- 检查当前消息是否已全部发送，如果没有则继续发送剩余部分。
- 如果当前消息已发送完毕，则处理队列中的下一个消息。

**特点**

1. **处理消息队列**：`WriteCallBack` 函数处理一个消息队列，其中可能有多个消息等待发送。
2. **错误处理**：如果出现错误，函数直接输出错误信息并返回。
3. **继续发送**：如果当前消息未完全发送，则继续发送剩余部分。
4. **队列管理**：如果当前消息已发送完毕，移出队列，并开始发送队列中的下一个消息（如果有）

**3）区别总结**

1. **消息处理**：

- `WriteCallBackErr`：只处理单条消息。
- `WriteCallBack`：处理消息队列中的多个消息。
- **数据发送逻辑**：
- `WriteCallBackErr`：检查和发送单条消息的数据。
- `WriteCallBack`：检查和发送当前消息队列中的消息，并在当前消息发送完毕后处理队列中的下一个消息。
- **队列管理**：
- `WriteCallBackErr`：没有队列管理，只处理一个 `msg_node`。
- `WriteCallBack`：管理发送队列，处理多条消息的发送。
- **函数参数**：
- `WriteCallBackErr`：需要传入一个 `std::shared_ptr<MsgNode>`，以确保 `MsgNode` 在异步操作期间不被销毁。
- `WriteCallBack`：不需要额外的 `MsgNode` 参数，直接使用 `_send_queue` 进行消息管理。

**6.布尔类型_send_pending的作用？**

虽然在 WriteToSocket 函数中，当 _send_pending 为 true 时不会启动新的异步写操作，但这并不意味着队列中只有一个元素。实际上，新的消息仍然会被添加到 _send_queue 中，只是当前异步写操作完成后，回调函数 WriteCallBack 会负责处理队列中的消息，并启动下一个消息的异步写操作。通过这种机制，确保了消息的有序发送，并避免了重叠的异步写操作。

这样设计的目的是为了确保数据按顺序发送，同时处理多个消息。即使多个消息连续调用 WriteToSocket，它们也会按顺序添加到队列，并由 WriteCallBack 依次处理

**详细解释**

1.消息添加到队列中：

- - 每次调用 `WriteToSocket` 时，新的消息都会被添加到 `_send_queue` 中。即使 `_send_pending` 为 `true`，消息仍然会被添加到队列，只是不会立即启动新的异步写操作。

2.挂起操作检查：

- - `WriteToSocket` 中检查 `_send_pending`，如果为 `true`，则返回。这意味着当前已经有一个异步写操作正在进行，新的异步写操作不会启动。

3.启动异步写操作：

- - 如果 `_send_pending` 为 `false`，则启动新的异步写操作，并将 `_send_pending` 设置为 `true`。这个标记确保在当前写操作完成之前，不会启动新的写操作。

4.回调函数处理：

- - 当异步写操作完成时，会调用 `WriteCallBack` 回调函数。
  - 回调函数首先检查错误代码，如果有错误则输出错误信息并返回。
  - 然后，回调函数更新当前消息的已发送字节数。如果当前消息没有发送完毕，则继续发送剩余部分。
  - 如果当前消息发送完毕，从队列中移除该消息。如果队列不为空，则启动下一个消息的发送；如果队列为空，则将 `_send_pending` 设置为 `false`。

**7.为什么async_read_some和async_receive不能混合使用，async_send和async_write_some不能混合使用？**

1）async_read_some和async_receive

**行为差异**：async_read_some 和 async_receive 的行为不完全一致。async_read_some 可能返回部分数据，而 async_receive 可能会处理完整的消息，并且它们的回调机制和内部状态管理不同。

**状态管理冲突**：每个异步操作在启动后都处于“挂起”状态，并且管理着套接字的读写状态。如果在同一个会话中交替使用 async_read_some 和 async_receive，会造成状态混乱，因为 Boost.Asio 的异步操作需要确保套接字的连续性和一致性。混合使用可能导致重复读取、读取冲突或错误状态，无法正确管理缓冲区的数据传递。

2）async_send和async_write_some

**行为不一致**：async_send 和 async_write_some 的核心区别在于它们处理数据发送的方式不同。async_send 期望一次性发送完整的数据，而 async_write_some 允许部分发送。混合使用它们会导致逻辑上的混乱，尤其是在处理未发送完成的数据时。

**状态冲突**：每个异步发送操作（无论是 async_send 还是 async_write_some）都依赖内部状态和[缓冲区管理](https://zhida.zhihu.com/search?content_id=248112810&content_type=Article&match_order=1&q=缓冲区管理&zhida_source=entity)。如果你在某一时刻使用 async_send 发送了部分数据，然后立刻使用 async_write_some 来发送剩余的数据，两个操作之间的状态可能会发生冲突，因为它们的缓冲区和字节管理方式不同。async_send 期望发送完整的数据，它会在发送完毕后触发回调。而 async_write_some 只发送部分数据，并不会跟踪剩余部分的发送进度，因此，混合使用会导致不一致的进度跟踪和数据管理。

**双重挂起风险**：异步操作不能同时存在多个挂起的操作。比如如果正在执行 async_send，而你同时又启动 async_write_some，会出现冲突，因为套接字上只能有一个挂起的异步写操作。
