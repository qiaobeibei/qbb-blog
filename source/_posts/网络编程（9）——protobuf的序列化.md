---
title: 网络编程（9）——protobuf的序列化
date: 2024-11-03 11:28:04
categories:
- C++
- 网络编程
tags: 
- protobuf
typora-root-url: ./..
---

# 九、day9

这几天参加数模比赛没有时间学习，今天提交完论文赶忙过来学习**protobuf的配置和使用。**

参考：

[恋恋风辰官方博客](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF)

[visual studio配置C++ boost库_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1FY4y1S7QW/?spm_id_from=333.999.0.0&vd_source=29868cdbb6b2fb1514ce3c7c31892d68)

## **1）什么是protobuf？**

rotocol Buffers（protobuf）是一种由谷歌开发的数据序列化格式，用于高效地序列化结构化数据。它可以用于将结构化数据序列化到***[二进制](https://zhida.zhihu.com/search?content_id=248600734&content_type=Article&match_order=1&q=二进制&zhida_source=entity)*格式**，并广泛用于数据存储、通信协议、配置文件等领域。

我们的逻辑是有类等抽象数据构成的，而tcp是面向字节流的，所以我们需要将类结构序列化为字符串来传输，这便需要借助protobuf。

下载地址：[Releases · protocolbuffers/protobuf (github.com)](https://link.zhihu.com/?target=https%3A//github.com/protocolbuffers/protobuf/releases)

## 2）Cmake？

CMake 是一个跨平台的[开源构建系统](https://zhida.zhihu.com/search?content_id=248600734&content_type=Article&match_order=1&q=开源构建系统&zhida_source=entity)，用于管理项目的构建过程。它使用简单的配置文件（CMakeLists.txt）来控制软件的编译和构建，可以生成标准的构建文件，如 Makefile 或 Visual Studio 项目文件。

**主要特点：**

1. **跨平台支持**：可以在不同的操作系统上使用，包括 Windows、Linux 和 macOS。
2. **灵活性**：支持多种编译器和构建工具，易于集成第三方库。
3. **模块化**：支持模块化构建，便于管理大型项目。
4. **依赖管理**：自动处理项目依赖关系，简化构建过程。

下载地址：[Download CMake](https://link.zhihu.com/?target=https%3A//cmake.org/download/)

protobuf的相比编译过程请参考

[恋恋风辰官方博客 (llfc.club)](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/2Pp1SSXN9MDHMFG9WtkayoC3BeC)

## 3）生成pb文件

pb文件包含了要序列化的类信息，首先创建一个msg.proto，该文件用来定义要发送的类信息。

```c++
syntax = "proto3";
message MsgData
{
    int32 id = 1;
    string data = 2;
}
```

该文件定义了一个名为MsgData的消息类型，包含两个字段：id, data。其中每一个字段都有一个数字标识符，用于标识该字段在二进制流中的位置。我们需要使用proroc.exe基于msg.proto生成我们要用的C++类，在终端cd到msg.proto所处目录中，并输入如下命令：

```c++
protoc --cpp_out=. ./msg.proto
```

会生成如下两个文件，`msg.http://pb.cc`和`msg.pb.h`，其中`msg.pb.h`中定义了我们需要的MsgData类，`http://msg.pb.cc`中定义的MsgData[成员函数](https://zhida.zhihu.com/search?content_id=248600734&content_type=Article&match_order=1&q=成员函数&zhida_source=entity)

![img](/images/$%7Bfiilename%7D/format,png-1730604508787-23.png)

然后，在当前项目中将这两个文件添加进来。

右键项目→属性→C/C++→常规→附加包含目录，将`msg.http://pb.cc`和`msg.pb.h`文件所处目录包含进去

![img](/images/$%7Bfiilename%7D/format,png-1730604508787-24.png)

右键项目→属性→配置属性→vc++目录→包含目录，将proroc\include包含进来

![img](/images/$%7Bfiilename%7D/format,png-1730604508787-25.png)

右键项目→属性→配置属性→vc++目录→库目录，将proroc\bin包含进来

![img](/images/$%7Bfiilename%7D/format,png-1730604508787-26.png)

在visual中，在项目属性中，配置选择Debug，**平台选择X64**，选择VC++目录， 在包含目录中添加 `D:\cppsoft*protoc\include`在库目录中添加 `D:\cppsoft*protoc\bin`

在链接器的输入选项中添加protobuf用到的[lib库](https://zhida.zhihu.com/search?content_id=248600734&content_type=Article&match_order=1&q=lib库&zhida_source=entity)

![img](/images/$%7Bfiilename%7D/format,png-1730604508787-27.png)

```
libprotobufd.lib
libprotocd.lib
```

若是遇到以下报错：

```c++
1>msg.pb.obj : error LNK2001: 无法解析的外部符号 "class google::protobuf::internal::ExplicitlyConstructed<class std::basic_string<char,struct std::char_traits<char>,class std::allocator<char> >,8> google::protobuf::internal::fixed_address_empty_string" (?fixed_address_empty_string@internal@protobuf@google@@3V?$ExplicitlyConstructed@V?$basic_string@DU?$char_traits@D@std@@V?$allocator@D@2@@std@@$07@123@A)
1>D:\app\visual studio\file\C++\异步服务器demo\AsyncClient\x64\Debug\AsyncClient.exe : fatal error LNK1120: 1 个无法解析的外部命令
```

右键项目→属性→C/C++/[预处理器](https://zhida.zhihu.com/search?content_id=248600734&content_type=Article&match_order=1&q=预处理器&zhida_source=entity)/预处理器定义，编辑输入

```
PROTOBUF_USE_DLLS
```

## 4）客户端修改

首先利用protoc将发送的消息先序列化，然后发送给服务器

```c++
        MsgData msgdata; // 定义一个由protoc.exe依据msg.proto文件生成的MsgData类
        msgdata.set_id(100);
        msgdata.set_data("hello world"); 
        std::string request;
        msgdata.SerializeToString(&request);
        short request_length = request.length();
        char send_data[MAX_LENGTH] = { 0 };
        // 转换为网络字节序
        short request_host_length = boost::asio::detail::socket_ops::host_to_network_short(request_length);
        memcpy(send_data, &request_host_length, 2); // 将消息总长度放入存储区前两个字节
        memcpy(send_data + 2, request.c_str(), request_length); // 将消息体放入存储区首地址两字节偏移之后
        // 一次性发送数据，数据长度为消息总长度（2字节）+消息体（发送内容）
        boost::asio::write(sock, boost::asio::buffer(send_data, request_length + 2));
```

## 5）服务器修改

首先在CSession类中新加一个[公有成员函数](https://zhida.zhihu.com/search?content_id=248600734&content_type=Article&match_order=1&q=公有成员函数&zhida_source=entity)

```
void Send_protoc(std::string msg);
```

定义如下

```c++
void CSession::Send_protoc(std::string msg) {
	bool pending = false; // 发送标志，true时有未完成的发送操作，false为空
	// 使用lock_guard锁住_send_lock，确保_send_lock（发送队列）访问的线程安全的
	// 锁的存在确保了多个线程不会同时修改发送队列
	std::lock_guard<std::mutex> lock(_send_lock);
	int send_que_size = _send_que.size();
	if (send_que_size > MAX_SENDQUE) {
		cout << "session: " << _uuid << " send que fulled, size is " << MAX_SENDQUE << endl;
		return;
	}

	// 判断队列是否有未完成的发送操作
	if (_send_que.size() > 0) {
		pending = true;
	}
	_send_que.push(std::make_shared<MsgNode>(msg.c_str(), msg.length())); // 将发送消息存储至队列
	if (pending) { // 如果有未完成的发送，直接返回
		return;
	}
	// 异步发送
	auto& msgnode = _send_que.front();
	boost::asio::async_write(_socket, boost::asio::buffer(msgnode->_msg, msgnode->_total_len),
		std::bind(&CSession::haddle_write, this, std::placeholders::_1, shared_from_this()));
} // 当'}'结束后，_send_lock解锁，发送队列解锁
```

在回调读函数中新加一段代码

```c++
				MsgData msgdata;
				std::string receive_data;
				msgdata.ParseFromString(std::string(_recv_msg_node->_msg, _recv_msg_node->_total_len));
				std::cout << "receive msg id is " << msgdata.id () << " msg data is  " << msgdata.data() << endl;
				std::string return_str = "Server has received msg, msg data is " + msgdata.data();
				MsgData msgreturn;
				msgreturn.set_id(msgdata.id());
				msgreturn.set_data(return_str);
				msgreturn.SerializeToString(&return_str);
				Send_protoc(return_str);
```

这一段主要是通过函数**ParseFromString**将接收到的消息体数据（在客户端被转换为了二进制格式）解析出消息数据，并填充到msgdata对象的各个字段中；然后，使用函数**SerializeToString**将msgreturn对象中数据序列化成string（二进制），并存储至return_str。
