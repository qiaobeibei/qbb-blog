---
title: 网络编程（10）——json序列化
date: 2024-11-03 11:31:44
categories:
- C++
- 网络编程
tags: 
- json序列化
typora-root-url: ./..
---

# 十、day10

今天学习如何使用**jsoncpp**将json数据解析为c++对象，将c++对象序列化为json数据。jsoncp经常在网络通信中使用，也就是服务器和客户端的通信一般使用json（可视化好）；而protobuf一般在服务器之间的通信中使用

jsoncpp下载地址：[open-source-parsers/jsoncpp: A C++ library for interacting with JSON. (github.com)](https://link.zhihu.com/?target=https%3A//github.com/open-source-parsers/jsoncpp)

jsoncpp如何配置使用可参考博主

[恋恋风辰官方博客llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/2Q5XIMAjJ76n2snyNEHstog2W9b](https://link.zhihu.com/?target=https%3A//llfc.club/category%3Fcatid%3D225RaiVNI8pFDD5L4m807g7ZwmF%23!aid/2Q5XIMAjJ76n2snyNEHstog2W9b)

**1.如果从github上下载最新版本，cmake编译后使用jsoncpp库，博主恋恋风辰给出的示例代码会造成内存泄漏，应该是最新版的库内部做了一些调整，该问题我没有解决，使用的仍是旧版本的jsoncpp。**

**2.如果旧版本的jsoncpp没有x64平台，需要自己在管理那里添加设置，确保平台和线程使用的统一性。**

**3.客户端和服务器做如下调整：**

## **客户端**

**客户端和服务器都需要包含库目录和包含目录，比如**

![img](/images/$%7Bfiilename%7D/format,png-1730604725983-38.png)

![img](/images/$%7Bfiilename%7D/format,png-1730604725983-39.png)

且需要在链接器→输入→附加依赖项中，添加需要的库

```
libprotobufd.lib
libprotocd.lib
json_vc71_libmtd.lib
```

客户端代码做以下调整

```c++
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
        //转为网络字节序
        int request_host_length = boost::asio::detail::socket_ops::host_to_network_short(request_length);
        memcpy(send_data, &request_host_length, 2);
        memcpy(send_data + 2, request.c_str(), request_length);
        boost::asio::write(sock, boost::asio::buffer(send_data, request_length + 2));

        char reply_head[HEAD_LENGTH]; // 首先读取对端发送消息的总长度
        size_t reply_length = boost::asio::read(sock, boost::asio::buffer(reply_head, HEAD_LENGTH));
        short msglen = 0; // 消息总长度
        memcpy(&msglen, reply_head, HEAD_LENGTH); // 将消息总长度赋值给msglen
        //转为本地字节序
        msglen = boost::asio::detail::socket_ops::network_to_host_short(msglen);
        char msg[MAX_LENGTH] = { 0 }; // 构建消息体（不含消息总长度）
        size_t msg_length = boost::asio::read(sock, boost::asio::buffer(msg, msglen));

        Json::Reader reader;
        reader.parse(std::string(msg, msg_length), root);
        std::cout << "msg id is " << root["id"] << " msg is " << root["data"] << endl;
        std::getchar();
    }
    catch (std::exception& e) {
        std::cerr << "Exception: " << e.what() << endl;
    }
    return 0;
}
```

## 服务器

服务也通常和客户端做相同的设置，包含库。并在回调读函数中新加这样一段

```c++
			//jsoncpp序列化
			Json::Reader reader;
			Json::Value root;
			reader.parse(std::string(_recv_msg_node->_msg, _recv_msg_node->_total_len), root);
			std::cout << "recevie msg id  is " << root["id"].asInt() << " msg data is "
				<< root["data"].asString() << endl;
			root["data"] = "Server has received msg, msg data is " + root["data"].asString();
			std::string return_str = root.toStyledString();
			Send_protoc(return_str);
```

完整的回调读函数如下：

```c++
void CSession::handle_read(const boost::system::error_code& error, size_t bytes_transferred,
	std::shared_ptr<CSession> _self_shared) {
	if (!error) {

		// 打印缓存区的数据并将该线程暂停2s
		//PrintRecvData(_data, bytes_transferred);
		//std::chrono::milliseconds dura(2000);
		//std::this_thread::sleep_for(dura);

		// 每触发一次handale_read，它会返回实际读取的字节数bytes_transferred，copy_len表示已处理的长度，每处理一字节，copy_len便加一
		int copy_len = 0; // 已经处理的字符数
		while (bytes_transferred > 0) { // 只要读取到数据就对其处理
			if (!_b_head_parse) { // 判断消息头部是否已处理，_b_head_parse默认为false
				// 异步读取到的字节数 + 已接收到的头部长度 < 头部总长度
				if (bytes_transferred + _recv_head_node->_cur_len < HEAD_LENGTH) { // 收到的数据长度小于头部长度，说明头部还未全部读取
					// 如果未完全接收消息头，则将接收到的数据复制到头部缓冲区
					// _recv_head_node->_msg，更新当前头部的接收长度，并继续异步读取剩余数据。
					memcpy(_recv_head_node->_msg + _recv_head_node->_cur_len, _data + copy_len, bytes_transferred);
					_recv_head_node->_cur_len += bytes_transferred;
					// 缓冲区清零，无需更新copy_len追踪已处理的字符数，因为之前读取的数据已经全部写入头部节点，下一个
					// 读入的消息从头开始（copy_len=0）往头节点写
					::memset(_data, 0, MAX_LENGTH);
					// 继续读消息
					_socket.async_read_some(boost::asio::buffer(_data, MAX_LENGTH), std::bind(&CSession::headle_read, this,
						std::placeholders::_1, std::placeholders::_2, _self_shared));
					return;
				}

				// 如果接收到的数据量足够处理消息头部，则计算头部剩余的未接收字节，
				// 并将其从 _data 缓冲区复制到头部消息缓冲区 _recv_head_node->_msg
				int head_remain = HEAD_LENGTH - _recv_head_node->_cur_len; // 头部剩余未复制的长度
				// 填充头部节点
				memcpy(_recv_head_node->_msg + _recv_head_node->_cur_len, _data + copy_len, head_remain);
				copy_len += head_remain; // 更新已处理的data长度
				bytes_transferred -= head_remain; // 更新剩余未处理的长度

				short data_len = 0; // 获取头部数据（消息长度）
				memcpy(&data_len, _recv_head_node->_msg, HEAD_LENGTH);
				//网络字节序转化为本地字节序
				data_len = boost::asio::detail::socket_ops::network_to_host_short(data_len);
				cout << "data_len is " << data_len << endl;
				
				if (data_len > MAX_LENGTH) { // 判断头部长度是否非法
					std::cout << "invalid data length is " << data_len << endl;
					_server->ClearSession(_uuid);
					return;
				}

				_recv_msg_node = std::make_shared<MsgNode>(data_len); // 已知数据长度data_len，构建消息内容载体
				//消息的长度小于头部规定的长度，说明数据未收全，则先将部分消息放到接收节点里
				if (bytes_transferred < data_len) {
					memcpy(_recv_msg_node->_msg + _recv_msg_node->_cur_len, _data + copy_len, bytes_transferred);
					_recv_msg_node->_cur_len += bytes_transferred;
					// copy_len不用更新，缓冲区会清零，下一个读入data的数据从头开始写入，copy_len也会被初始化为0
					::memset(_data, 0, MAX_LENGTH);
					_socket.async_read_some(boost::asio::buffer(_data, MAX_LENGTH),
						std::bind(&CSession::headle_read, this, std::placeholders::_1, std::placeholders::_2, _self_shared));
					
					_b_head_parse = true; //头部处理完成
					return;
				}

				// 接收的长度多于消息内容长度
				memcpy(_recv_msg_node->_msg + _recv_msg_node->_cur_len, _data + copy_len, data_len);
				_recv_msg_node->_cur_len += data_len;
				copy_len += data_len;
				bytes_transferred -= data_len;
				_recv_msg_node->_msg[_recv_msg_node->_total_len] = '\0';
				// cout << "receive data is " << _recv_msg_node->_msg << endl;

				// protobuf序列化
				//MsgData msgdata;
				//std::string receive_data;
				//msgdata.ParseFromString(std::string(_recv_msg_node->_msg, _recv_msg_node->_total_len));
				//std::cout << "receive msg id is " << msgdata.id () << " msg data is  " << msgdata.data() << endl;
				//std::string return_str = "Server has received msg, msg data is " + msgdata.data();
				//MsgData msgreturn;
				//msgreturn.set_id(msgdata.id());
				//msgreturn.set_data(return_str);
				//msgreturn.SerializeToString(&return_str);
				//Send_protoc(return_str);

				// jsoncpp序列化
				Json::Reader reader;
				Json::Value root;
				reader.parse(std::string(_recv_msg_node->_msg, _recv_msg_node->_total_len), root);
				std::cout << "recevie msg id  is " << root["id"].asInt() << " msg data is "
					<< root["data"].asString() << endl;
				root["data"] = "Server has received msg, msg data is " + root["data"].asString();
				std::string return_str = root.toStyledString();
				Send_protoc(return_str);

				//Send(_recv_msg_node->_msg, _recv_msg_node->_total_len); // 回传
				// 清理已处理的头部消息并重置，准备解析下一条消息
				_b_head_parse = false;
				_recv_head_node->Clear();

				// 如果当前数据已经全部处理完，重置缓冲区 _data，并继续异步读取新的数据
				if (bytes_transferred <= 0) {
					::memset(_data, 0, MAX_LENGTH);
					_socket.async_read_some(boost::asio::buffer(_data, MAX_LENGTH),
						std::bind(&CSession::headle_read, this, std::placeholders::_1, std::placeholders::_2, _self_shared));
					return;
				}

				continue; // 异步读取的消息未处理完，继续填充头节点乃至新的消息节点
			}

			//已经处理完头部，处理上次未接受完的消息数据
			int remain_msg = _recv_msg_node->_total_len - _recv_msg_node->_cur_len;
			if (bytes_transferred < remain_msg) { //接收的数据仍不足剩余未处理的
				memcpy(_recv_msg_node->_msg + _recv_msg_node->_cur_len, _data + copy_len, bytes_transferred);
				_recv_msg_node->_cur_len += bytes_transferred;
				::memset(_data, 0, MAX_LENGTH);
				_socket.async_read_some(boost::asio::buffer(_data, MAX_LENGTH),
					std::bind(&CSession::headle_read, this, std::placeholders::_1, std::placeholders::_2, _self_shared));
				return;
			}
			// 接收的数据多于剩余未处理的长度
			memcpy(_recv_msg_node->_msg + _recv_msg_node->_cur_len, _data + copy_len, remain_msg);
			_recv_msg_node->_cur_len += remain_msg;
			bytes_transferred -= remain_msg;
			copy_len += remain_msg;
			_recv_msg_node->_msg[_recv_msg_node->_total_len] = '\0';
			//cout << "receive data is " << _recv_msg_node->_msg << endl;

			// protobuf序列化
			//MsgData msgdata;
			//std::string receive_data;
			//msgdata.ParseFromString(std::string(_recv_msg_node->_msg, _recv_msg_node->_total_len));
			//std::cout << "receive msg id is " << msgdata.id() << " msg data is  " << msgdata.data() << endl;
			//std::string return_str = "Server has received msg, msg data is " + msgdata.data();
			//MsgData msgreturn;
			//msgreturn.set_id(msgdata.id());
			//msgreturn.set_data(return_str);
			//msgreturn.SerializeToString(&return_str);
			//Send_protoc(return_str);

			//jsoncpp序列化
			Json::Reader reader;
			Json::Value root;
			reader.parse(std::string(_recv_msg_node->_msg, _recv_msg_node->_total_len), root);
			std::cout << "recevie msg id  is " << root["id"].asInt() << " msg data is "
				<< root["data"].asString() << endl;
			root["data"] = "Server has received msg, msg data is " + root["data"].asString();
			std::string return_str = root.toStyledString();
			Send_protoc(return_str);

			//此处可以调用Send发送测试
			//Send(_recv_msg_node->_msg, _recv_msg_node->_total_len);
			//继续轮询剩余未处理数据
			_b_head_parse = false;
			_recv_head_node->Clear();
			if (bytes_transferred <= 0) {
				::memset(_data, 0, MAX_LENGTH);
				_socket.async_read_some(boost::asio::buffer(_data, MAX_LENGTH),
					std::bind(&CSession::headle_read, this, std::placeholders::_1, std::placeholders::_2, _self_shared));
				return;
			}
			continue;
		}
	}
	else {
		std::cout << "handle read failed, error is " << error.what() << endl;
		Close();
		_server->ClearSession(_uuid);
	}
}
```

