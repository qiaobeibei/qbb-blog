---
title: 网络编程（20）——了解http报文头格式并搭建http服务器
date: 2024-11-03 11:59:13
categories:
- 网络编程
tags: 
- http
- http报文头格式
typora-root-url: ./..
---

## 二十、day20

前面的所有章节均是为了实现并发的长连接tcp服务器，后面学习一些其他知识点，比如如何利用boost库实现http服务器、websocket服务器，并初步学习如何利用grpc进行通信。

| 类型            | 定义                        | 特点                                               | 应用场景                         |
| --------------- | --------------------------- | -------------------------------------------------- | -------------------------------- |
| 异步服务器      | 通过异步I/O处理请求的服务器 | 使用事件循环或回调处理高并发；资源利用率高；低延迟 | 实时应用、在线聊天、实时数据推送 |
| HTTP服务器      | 专门处理HTTP请求的服务器    | 基于HTTP/HTTPS协议；通常是无状态的；可同步或异步   | 静态网站、API服务、文件下载      |
| WebSocket服务器 | 支持WebSocket协议的服务器   | 支持全双工通信；持久连接；通常基于异步模型         | 实时聊天、在线游戏、实时数据更新 |

在学习http服务器之前，先学习http的报文头格式，这主要是为了避免粘包问题，告诉服务器一个数据包的开始和结尾，并在包头里标识请求的类型如get或post等信息

资料参考自博主恋恋风辰：

【C++ 网络编程(21) asio实现http服务器】

https://www.bilibili.com/video/BV1Ns4y1C72v?vd_source=cb95e3058c2624d2641da6f4eeb7e3a1www.bilibili.com/video/BV1Ns4y1C72v?vd_source=cb95e3058c2624d2641da6f4eeb7e3a1

## **1. HTTP报头（**(HTTP header**）**

HTTP服务器不会保存关于客户的任何信息，是一个**无状态协议**。**请求头（**Request Headers**）**和**响应头（**Response Headers**）**两部分共同组成一个表征的http报文头格式。

### 1）HTTP请求头

HTTP请求头包括以下字段：

- **Host**：指定服务器的主机名和端口号，用于确定请求的目标服务器。在一般情况下，HTTP请求中的Host头部和URL中的主机部分是相同的，因为Host头部指定了目标服务器的主机名。但是在一些特殊情况下，Host头部和URL中的主机部分可能会不同：
- **代理服务器（Proxy Server）**：当客户端通过代理服务器发送请求时，Host头部通常指定的是目标服务器的主机名，而URL中的主机部分则是代理服务器的主机名。这是因为代理服务器会将客户端的请求转发给目标服务器，但Host头部应该指示目标服务器的主机名，以确保服务器正确识别请求的目标
- **虚拟主机（Virtual Hosting）**：在共享主机环境下，一台服务器可能托管了多个网站，这些网站共享同一个IP地址。在这种情况下，服务器根据请求中的Host头部来确定应该将请求转发给哪个网站。因此，URL中的主机部分可能是共享主机的IP地址，而Host头部指示的是请求的实际域名
- **Request-line**：包含用于描述请求类型、要访问的资源以及所使用的HTTP版本的信息
- **Accept**：指定客户端所能接受的内容类型，通常用于告知服务器客户端支持哪些媒体类型（如HTML、XML、JSON等）
- **User-Agent**：客户端使用的浏览器类型和版本号，供服务器统计用户代理信息
- **Cookie**：包含了之前由服务器通过Set-Cookie响应头设置的Cookie信息，用于在客户端和服务器之间维护会话状态，如果请求中包含cookie信息，则通过这个字段将cookie信息发送给Web服务器
- Cookie是由服务器发送到客户端，并存储在客户端的浏览器中的小型数据片段。它用于在客户端和服务器之间存储会话信息或跟踪用户状态，以便在用户访问同一网站时保持持久性和状态。Cookie通常包含了一些键值对的数据，以及一些关于Cookie的属性：
- **名称（Name）**：Cookie的名称，用于唯一标识一个Cookie
- **值（Value）**：Cookie的值，存储在客户端的数据
- **域（Domain）**：指定了Cookie所属的域名。默认情况下，Cookie的域为创建它的服务器的域名，但也可以通过设置Domain属性来指定其他域名
- **路径（Path）**：指定了Cookie的可见路径。只有在指定路径下的页面才能访问到这个Cookie，默认情况下，Cookie的路径为创建它的页面路径
- **过期时间（Expires/Max-Age）**：指定了Cookie的过期时间。过期时间可以是一个具体的日期时间，也可以是从当前时间开始的秒数。当过期时间到达后，Cookie将被自动删除
- **安全标志（Secure）**：指示浏览器仅在通过加密协议（如HTTPS）发送请求时才发送Cookie到服务器。这样可以确保Cookie在传输过程中不被窃取或篡改
- **HttpOnly标志（HttpOnly）**：指示浏览器禁止JavaScript访问Cookie，这样可以防止某些类型的跨站点脚本攻击
- **Connection**：指定是否需要保持持久连接，或者是否需要进行连接升级等

举例：

```
GET /index.html HTTP/1.1
Host: www.example.com
Accept: text/html, application/xhtml+xml, */*
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:123.0) Gecko/20100101 Firefox/123.0
Cookie: sessionid=abcdefg1234567
Connection: keep-alive
```

请求报文的第一行叫做**请求行**，其后继的行叫作**首部行**。请求行有三个字段：方法字段、URL字段和HTTp版本字段。一般在首部行（和附加的额外一行）后有一个“**实体体**”，使用GET方法时实体体为空，而使用POST时实体体需要用户写入相关内容。如下图：

![img](/images/$%7Bfiilename%7D/v2-ab640142bcae401f4d03f2000f70413b_720w.png)

<center>图片来源：计算机网络：自顶向下法</center>

上述请求头包括了以下字段：

- **Request-line**：指定使用GET方法请求/index.html资源，并使用HTTP/1.1协议版本
- **Host**：指定被请求资源所在主机名或IP地址和端口号
- **Accept**：客户端期望接收的媒体类型列表，本例中指定了text/html、application/xhtml+xml和任意类型的文件（*/*）
- **User-Agent**：客户端浏览器类型和版本号
- **Cookie**：客户端发送给服务器的cookie信息
- **Connection**：客户端请求后是否需要保持长连接

> **报文每一行最后都由一个回车和一个换行符组成，最后一行在附加一个回车换行符（额外增加一个空行）。**

### 2） HTTP响应头

响应报文由一个初始状态行、若干个首部行和实体体组成。实体体是报文的主要组成部分，它包含了所请求的对象本身；状态行有3个字段：协议版本字段、状态码和相应状态信息。

![img](/images/$%7Bfiilename%7D/v2-ab640142bcae401f4d03f2000f70413b_720w-1730606386723-174.png)

<center>图片来源：计算机网络：自顶向下法</center>

HTTP响应头包括以下字段：

- **Status-line**：包含协议版本、状态码和状态消息
- **Content-Type**：指定了响应体的内容类型。与上同
- **Content-Length**：指定了响应体的长度。与上同
- **Set-Cookie**：用于设置新的Cookie或更新已有的Cookie
- **Server**：指定了响应的服务器软件信息
- **Connection**：表示是否需要保持长连接（keep-alive）

实际HTTp报文头中，还可以包含其他可选字段，比如

- **Cache-Control**：指定了响应的缓存控制方式
- **Expires**：指定了响应的过期时间
- **Last-Modified**：指定了响应的最后修改时间
- **Location**：指定了重定向的目标位置
- **ETag**：指定了响应内容的实体标签，用于验证资源是否被修改

举例：

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 1024
Set-Cookie: sessionid=abcdefg1234567; HttpOnly; Path=/
Server: Apache/2.2.32 (Unix) mod_ssl/2.2.32 OpenSSL/1.0.1e-fips mod_bwlimited/1.4
Connection: keep-alive
```

上述响应头包括了以下字段：

- **Status-line**：指定HTTP协议版本、状态码和状态消息
- **Content-Type**：指定响应体的MIME类型及字符编码格式
- **Content-Length**：指定响应体的字节数
- **Set-Cookie**：服务器向客户端发送cookie信息时使用该字段
- **Server**：服务器类型和版本号
- **Connection**：服务器是否需要保持长连接

### 3）用户与服务器的交互：cookie

> 上文提到过了http服务器是**无状态的**，这允许服务器可以同时处理数以千计的TCp连接，但我们如何能够识别用户（一把是需要识别用户来限制用户的访问，或者需要把内容与用户身份联系起来）？

可以实现无状态服务器对用户身份的识别。

**cookie有4个组件**：①在HTTP响应报文中的一个cookie首部行；②在HTTP请求报文中的一个cookie首部行；③在用户端系统中保留一个cookie文件，并由用户的浏览器进行管理；④位于Web站点的一个后端数据库。

![img](/images/$%7Bfiilename%7D/v2-13b15f802af5dbabe5d7101a9302ec6a_720w.jpeg)

<center>图片来源：计算机网络：自顶向下法</center>

上图为cookie的工作过程：

假设Susan首次与Amazon.com联系，我们假定过去她已经访问过eBay站点。当请求报文到达该Amazon Web服务器时，该Web站点将产生一个唯一识别码，并以此作为索引在它的后端数据库中产生一个表项。接下来Amazon Web服务器用一个包含Set-cookie: 首部的HTTP响应报文对suSan的浏览器进行响应，其中Set-cookie: 首部含有该识别码。例如，该首部行可能是：

```
Set-cookie: 1678
```

当Susan的浏览器收到了该HTTP响应报文时，它会看到该Set-cookie：首部。该浏览器在它管理的特定cookie文件中添加一行，该行包含服务器的主机名和在Set-cookie：首部中的识别码。值得注意的是该cookie文件已经有了用于eBay的表项，因为SUsan过去访问过该站点。当Susan继续浏览Amazon网站时，每请求一个Web页面，浏览器就会查询该cookie文件并抽取她对这个网站的识别码，并放到HTTP请求报文中包括识别码的cookie首部行中。特别是，发往该Amazon服务器的每个HTTP请求报文都包括以下首部行：

```
Set-cookie: 1678
```

在这种方式下，Amazon服务器可以跟踪Susan在Amazon站点的活动。尽管Amazon Web站点不必知道Susan的名字，但它确切地知道用户1678按照什么顺序、在什么时间、访问了哪些页面！Amazon使用cookie来提供它的购物车服务，即Amazon能够维护Susan希望购买的物品列表，这样在Susan结束会话时可以一起为它们付费。

如果Susan再次访问Amazon站点，比如说一个星期后，她的浏览器会在其请求报文中继续放入首部行cookie：1768。Amazon将根据Susan过去在Amazon访问的网页向她推荐产品。如果Susan也在Amazon注册过，即提供了她的全名、电子邮件地址、邮政地址和信用卡账户，则Amaon能在其数据库中包括这些信息，将Susan的名字与识别码相关联（以及她在过去访问过的本站点的所有页面）。这就理解了Amazon和其他一些电子商务网站实现“点击购物”的道路，即当SUsan在后继的访问中选择购买某个物品时，他不必重新输入姓名、信用卡账户和地址等信息。

> 参考：《计算机网络：自顶向下法》

## 2. 客户端

```c++
#include <iostream>
#include <istream>
#include <ostream>
#include <string>
#include <boost/asio.hpp>
#include <boost/bind/bind.hpp>

using boost::asio::ip::tcp;

class client
{
public:
    // server:服务器地址（包含ip地址和端口号），path：请求的资源路径，比如"/index.html"
    client(boost::asio::io_context& io_context,
        const std::string& server, const std::string& path)
        : resolver_(io_context), // resolver对象，用于解析服务器地址，将域名或IP地址转换为可以用于连接的端点
        socket_(io_context)
    {
        // 构建HTTP请求
        std::ostream request_stream(&request_); // 用于存储即将发送的http请求数据
        request_stream << "GET " << path << " HTTP/1.0\r\n"; // 指定HTTP方法（GET），请求路径，以及HTTP版本（1.0）
        request_stream << "Host: " << server << "\r\n"; // 添加Host头，用于指定请求的服务器
        request_stream << "Accept: */*\r\n"; // 指定接受所有类型的响应
        request_stream << "Connection: close\r\n\r\n"; // 表示请求完成后关闭连接

        // 解析服务器的IP和端口，比如域名：“127.0.0.1:80”
		size_t pos = server.find(":"); // 查找服务器地址中的冒号位置，用于将IP和端口分开
		std::string ip = server.substr(0, pos);
		std::string port = server.substr(pos + 1);

        //  异步解析和连接
        resolver_.async_resolve(ip, port, // 开始异步解析，将ip和port转换为可以用于连接的端点。
            boost::bind(&client::handle_resolve, this,
                boost::asio::placeholders::error,
                boost::asio::placeholders::results));
    }

private:
    // resolver_解析成功后，调用该函数连接服务器端点
    void handle_resolve(const boost::system::error_code& err,
        const tcp::resolver::results_type& endpoints)
    {
        if (!err) // 解析成功
        {
            // 异步连接
            boost::asio::async_connect(socket_, endpoints,
                boost::bind(&client::handle_connect, this,
                    boost::asio::placeholders::error));
        }
        else
        {
            std::cout << "Error: " << err.message() << "\n";
        }
    }

    // 连接成功后，调用该函数进行异步发操作
    void handle_connect(const boost::system::error_code& err)
    {
        if (!err)
        {
            boost::asio::async_write(socket_, request_,
                boost::bind(&client::handle_write_request, this,
                    boost::asio::placeholders::error));
        }
        else
        {
            std::cout << "Error: " << err.message() << "\n";
        }
    }

    // 异步发成功后，调用该函数执行异步读，直到读取到指定的分隔符（\r\n）
    void handle_write_request(const boost::system::error_code& err)
    {
        if (!err)
        {
            boost::asio::async_read_until(socket_, response_, "\r\n",
                boost::bind(&client::handle_read_status_line, this,
                    boost::asio::placeholders::error));
        }
        else
        {
            std::cout << "Error: " << err.message() << "\n";
        }
    }

    // 读取到分隔符（\r\n）后，调用该函数解析http状态响应
    void handle_read_status_line(const boost::system::error_code& err)
    {
        if (!err)
        {
            // 解析HTTP响应状态行，“HTTP/1.1 200 OK\r\n”
            std::istream response_stream(&response_);
            std::string http_version;
            response_stream >> http_version; // 读取 HTTP 版本号（如 HTTP/1.1）
            unsigned int status_code; 
            response_stream >> status_code; // 读取状态码（如 200、404）
            std::string status_message;
            std::getline(response_stream, status_message); // 读取状态消息（如 "OK" 或 "Not Found"）

            // 检查 response_stream 是否有效，以及 http_version 是否以 "HTTP/" 开头，
            // 以验证响应是否符合 HTTP 标准。如果不符合，则输出 "Invalid response" 并返回
            if (!response_stream || http_version.substr(0, 5) != "HTTP/")
            {
                std::cout << "Invalid response\n";
                return;
            }
            // 判定状态码是否为 200（表示请求成功）
            if (status_code != 200)
            {
                std::cout << "Response returned with status code ";
                std::cout << status_code << "\n";
                return;
            }

            // 继续异步读取 HTTP 响应头，直到读取到响应头结束
            boost::asio::async_read_until(socket_, response_, "\r\n\r\n",
                boost::bind(&client::handle_read_headers, this,
                    boost::asio::placeholders::error));
        }
        else
        {
            std::cout << "Error: " << err << "\n";
        }
    }

    // 响应头读取成功后，调用该函数解析响应头
    void handle_read_headers(const boost::system::error_code& err)
    {
        if (!err)
        {
            // 处理响应头
            std::istream response_stream(&response_);
            std::string header;
            // 逐行读取HTTP头部，每一行都存储在header中，并检查是否是空行
            // 如果为空行，表示该行没有数据，响应头结束
            while (std::getline(response_stream, header) && header != "\r")
                std::cout << header << "\n";
            std::cout << "\n";

            // 如果缓冲区response_中还有数据（可能是部分或全部的响应正文）
            if (response_.size() > 0)
                std::cout << &response_;

            // 继续异步读取服务器的响应内容
            boost::asio::async_read(socket_, response_,
                boost::asio::transfer_at_least(1), // 保证至少读取一个字符
                boost::bind(&client::handle_read_content, this,
                    boost::asio::placeholders::error));
        }
        else
        {
            std::cout << "Error: " << err << "\n";
        }
    }

    // 该函数用于处理异步读取HTTP响应正文的数据
    void handle_read_content(const boost::system::error_code& err)
    {
        if (!err)
        {
            // 输出已读取的数据
            std::cout << &response_;

            // 继续读取剩余的数据
            boost::asio::async_read(socket_, response_,
                boost::asio::transfer_at_least(1),
                boost::bind(&client::handle_read_content, this,
                    boost::asio::placeholders::error));
        }
        else if (err != boost::asio::error::eof) // 是否读取到文件尾，读取完成
        {
            std::cout << "Error: " << err << "\n";
        }
    }

    tcp::resolver resolver_; // 服务器域名解析器
    tcp::socket socket_; 
    boost::asio::streambuf request_; // 存储要发送给服务器的请求数据
    boost::asio::streambuf response_; // 
};

int main(int argc, char* argv[])
{
    try
    {
		/*   if (argc != 3)
		   {
			   std::cout << "Usage: async_client <server> <path>\n";
			   std::cout << "Example:\n";
			   std::cout << "  async_client www.boost.org /LICENSE_1_0.txt\n";
			   return 1;
		   }*/

        boost::asio::io_context io_context;
        // "/" 表示网站的根目录，服务器返回“/index.html”
        client c(io_context, "127.0.0.1:8080", "/");
        io_context.run();
        getchar();
    }
    catch (std::exception& e)
    {
        std::cout << "Exception: " << e.what() << "\n";
    }

    return 0;
}
```

定义Client类作为客户端，通过发送http请求接收响应头和响应内容。

```c++
    tcp::resolver resolver_; // 服务器域名解析器
    tcp::socket socket_; 
    boost::asio::streambuf request_; // 存储要发送给服务器的请求数据
    boost::asio::streambuf response_; // 接收和解析从服务器获取到的响应
```

### **a. 构造函数**

```c++
client(boost::asio::io_context& io_context,
        const std::string& server, const std::string& path)
```

Client构造函数接收ioc、域名**server**、路径**path**，其中**server**一般由服务器的ip和端口组成，比如“127.0.0.1:80”，**path**是请求资源的路径，比如"/index.html"。然后构建http请求并存储至**request_**，解析服务器的IP和端口存储至**ip**和**port**变量。最后，调用**async_resolve**将服务器ip和端口异步解析为可以用于连接的端点，如果解析成功，调用**handle_resolve**函数连接该服务器端点。

### **b. handle_resolve**

```c++
    // resolver_解析成功后，调用该函数连接服务器端点
    void handle_resolve(const boost::system::error_code& err,
        const tcp::resolver::results_type& endpoints)
    {
        if (!err) // 解析成功
        {
            // 异步连接
            boost::asio::async_connect(socket_, endpoints,
                boost::bind(&client::handle_connect, this,
                    boost::asio::placeholders::error));
        }
        else
        {
            std::cout << "Error: " << err.message() << "\n";
        }
    }
```

调用**async_connect**函数异步连接解析成功的服务器端点，连接成功后调用**handle_connect**函数，进行异步发操作。

### **c. handle_connect**

```c++
    void handle_connect(const boost::system::error_code& err)
    {
        if (!err)
        {
            boost::asio::async_write(socket_, request_,
                boost::bind(&client::handle_write_request, this,
                    boost::asio::placeholders::error));
        }
        else
        {
            std::cout << "Error: " << err.message() << "\n";
        }
    }
```

连接成功后，调用异步发async_write函数发送请求头（构造函数构建的请求头存储至request_），发送成功后调用handle_write_request函数，读取http相应行，也就是“HTTP/1.1200 OK\r\n”

### **d. handle_write_request**

```c++
    // 异步发成功后，调用该函数执行异步读，直到读取到指定的分隔符（\r\n）
    void handle_write_request(const boost::system::error_code& err)
    {
        if (!err)
        {
            boost::asio::async_read_until(socket_, response_, "\r\n",
                boost::bind(&client::handle_read_status_line, this,
                    boost::asio::placeholders::error));
        }
        else
        {
            std::cout << "Error: " << err.message() << "\n";
        }
    }
```

HTTP 请求和响应消息的每一行都是以 **\r\n** 作为行结束符。这包括起始行（请求行或状态行）、头部字段行以及空行（表示头部和正文的分隔）。

```
HTTP/1.1 200 OK\r\n
Content-Type: text/html\r\n
Content-Length: 123\r\n
\r\n
<html>...</html>
```

- 在状态行（HTTP/1.1 200 OK）后面有 \r\n，表示这一行结束。
- 每个头部字段（如 Content-Type 和 Content-Length）后面也有 \r\n，表示每个字段的结束。
- 最后一个空行（"\r\n"）表示头部和正文（HTML 内容）之间的分隔。

两个“\r\n\r\n”读取两个换行，两个换行是空行，相当于HTTP 响应头的结束标志，比如

```
Content-Length: 123\r\n
\r\n
```

**连着两个换行表示响应头结束**

响应状态行读取成功后，调用**handle_read_status_line**函数读取并解析http响应状态行

### **e. handle_read_status_line**

```c++
// 读取到分隔符（\r\n）后，调用该函数解析http状态响应
    void handle_read_status_line(const boost::system::error_code& err)
    {
        if (!err)
        {
            // 解析HTTP响应状态行，“HTTP/1.1 200 OK\r\n”
            std::istream response_stream(&response_);
            std::string http_version;
            response_stream >> http_version; // 读取 HTTP 版本号（如 HTTP/1.1）
            unsigned int status_code; 
            response_stream >> status_code; // 读取状态码（如 200、404）
            std::string status_message;
            std::getline(response_stream, status_message); // 读取状态消息（如 "OK" 或 "Not Found"）

            // 检查 response_stream 是否有效，以及 http_version 是否以 "HTTP/" 开头，
            // 以验证响应是否符合 HTTP 标准。如果不符合，则输出 "Invalid response" 并返回
            if (!response_stream || http_version.substr(0, 5) != "HTTP/")
            {
                std::cout << "Invalid response\n";
                return;
            }
            // 判定状态码是否为 200（表示请求成功）
            if (status_code != 200)
            {
                std::cout << "Response returned with status code ";
                std::cout << status_code << "\n";
                return;
            }

            // 继续异步读取 HTTP 响应头，直到读取到响应头结束
            boost::asio::async_read_until(socket_, response_, "\r\n\r\n",
                boost::bind(&client::handle_read_headers, this,
                    boost::asio::placeholders::error));
        }
        else
        {
            std::cout << "Error: " << err << "\n";
        }
    }
```

对状态行进行判断处理，如果一切准备就绪，调用async_read_until函数读取响应头，直至响应头结束。两个换行表示响应头读取结束，结束后调用**handle_read_headers**函数解析响应头。

### **f. handle_read_headers**

```c++
// 响应头读取成功后，调用该函数解析响应头
    void handle_read_headers(const boost::system::error_code& err)
    {
        if (!err)
        {
            // 处理响应头
            std::istream response_stream(&response_);
            std::string header;
            // 逐行读取HTTP头部，每一行都存储在header中，并检查是否是空行
            // 如果为空行，表示该行没有数据，响应头结束
            while (std::getline(response_stream, header) && header != "\r")
                std::cout << header << "\n";
            std::cout << "\n";

            // 如果缓冲区response_中还有数据（可能是部分或全部的响应正文）
            if (response_.size() > 0)
                std::cout << &response_;

            // 继续异步读取服务器的响应内容
            boost::asio::async_read(socket_, response_,
                boost::asio::transfer_at_least(1), // 保证至少读取一个字符
                boost::bind(&client::handle_read_content, this,
                    boost::asio::placeholders::error));
        }
        else
        {
            std::cout << "Error: " << err << "\n";
        }
    }
```

响应头处理完成后，继续读取响应头后面的响应文本

```c++
boost::asio::transfer_at_least(1) 
```

确保每次读取至少一个字节的数据，保证读取的有效性。读取到正文数据后，调用**handle_read_content**函数进行处理。

### **g. handle_read_content**

```c++
    // 该函数用于处理异步读取HTTP响应正文的数据
    void handle_read_content(const boost::system::error_code& err)
    {
        if (!err)
        {
            // 输出已读取的数据
            std::cout << &response_;

            // 继续读取剩余的数据
            boost::asio::async_read(socket_, response_,
                boost::asio::transfer_at_least(1),
                boost::bind(&client::handle_read_content, this,
                    boost::asio::placeholders::error));
        }
        else if (err != boost::asio::error::eof) // 是否读取到文件尾，读取完成
        {
            std::cout << "Error: " << err << "\n";
        }
    }
```

### **h. 主函数**

```c++
int main(int argc, char* argv[])
{
    try
    {
        boost::asio::io_context io_context;
        // "/" 表示网站的根目录，服务器返回“/index.html”
        client c(io_context, "127.0.0.1:8080", "/");
        io_context.run();
        getchar();
    }
    catch (std::exception& e)
    {
        std::cout << "Exception: " << e.what() << "\n";
    }

    return 0;
}
```

实例化client ，"127.0.0.1:8080"表示服务器域名，"/" 表示网站的根目录，服务器返回“/index.html”

## 3.服务器

### *a. 主函数*

```c++
#include <iostream>
#include <string>
#include <boost/asio.hpp>
#include "server.hpp"
#include <filesystem>

int main(int argc, char* argv[])
{
	try
	{
		// 获取当前工作目录，并添加一个名为"res"的子目录
		// std::filesystem是C++17的新特性
		std::filesystem::path path = std::filesystem::current_path() / "res";

		// 输出拼接后的路径
		std::cout << "Path: " << path.string() << '\n';
		std::cout << "Usage: http_server <127.0.0.1> <8080> "<< path.string() <<"\n";
		// 服务器IP为"127.0.0.1"，端口号为“8080”，服务器根目录为path.string()
		http::server::server s("127.0.0.1", "8080", path.string());

		s.run();
	}
	catch (std::exception& e)
	{
		std::cerr << "exception: " << e.what() << "\n";
	}

	return 0;
}
```

主函数中，首先获取当前工作目录，并添加一个名为"res"的子目录，然后将**服务器IP**："127.0.0.1"，**端口号**：“8080”，**服务器根目录**：path.string()，这三个参数传入**server**的实例。

然后调用server.run()使得ioc运行，也就是让ioc_context开始run

```c++
void server::run()
{
	// The io_service::run() call will block until all asynchronous operations
	// have finished. While the server is running, there is always at least one
	// asynchronous operation outstanding: the asynchronous accept call waiting
	// for new incoming connections.
	io_service_.run();
}
```

### *b. server*

```c++
#ifndef HTTP_SERVER_HPP
#define HTTP_SERVER_HPP

#include <boost/asio.hpp>
#include <string>
#include "connection.hpp"
#include "connection_manager.hpp"
#include "request_handler.hpp"

namespace http {
	namespace server {
		class server
		{
		public:
			server(const server&) = delete;
			server& operator=(const server&) = delete;

			/// Construct the server to listen on the specified TCP address and port, and
			/// serve up files from the given directory.
			explicit server(const std::string& address, const std::string& port,
				const std::string& doc_root);

			/// Run the server's io_service loop.
			void run();

		private:
			/// Perform an asynchronous accept operation.
			void do_accept();
			/// Wait for a request to stop the server.
			void do_await_stop();
			/// The io_service used to perform asynchronous operations.
			boost::asio::io_service io_service_;
			/// The signal_set is used to register for process termination notifications.
			boost::asio::signal_set signals_;
			/// Acceptor used to listen for incoming connections.
			boost::asio::ip::tcp::acceptor acceptor_;
			/// The connection manager which owns all live connections.
			connection_manager connection_manager_;
			/// The next socket to be accepted.
			boost::asio::ip::tcp::socket socket_;
			/// The handler for all incoming requests.
			request_handler request_handler_;
		};

	} // namespace server
} // namespace http
#endif
```

server类的实现函数如下：

```c++
server::server(const std::string& address, const std::string& port,
			const std::string& doc_root)
			: io_service_(),
			signals_(io_service_), 
			acceptor_(io_service_),
			connection_manager_(),
			socket_(io_service_),
			// 初始化处理请求类（自己封装）
			request_handler_(doc_root)
		{
			signals_.add(SIGINT);
			signals_.add(SIGTERM);
#if defined(SIGQUIT)
			signals_.add(SIGQUIT);
#endif // defined(SIGQUIT)
			do_await_stop();
			// resolver用于将服务器域名解析为ip和端口号
			boost::asio::ip::tcp::resolver resolver(io_service_);
			// 将解析出来的ip和端口号通过resolve方法解析为端点
			boost::asio::ip::tcp::endpoint endpoint = *resolver.resolve({ address, port });
			acceptor_.open(endpoint.protocol()); // acceptor使用端点的协议
			// 允许地址复用（reuse_address(true)）
			// 作用是让多个程序或多个进程能够在地址已经被占用的情况下重用同一个地址和端口
			acceptor_.set_option(boost::asio::ip::tcp::acceptor::reuse_address(true));
			acceptor_.bind(endpoint);
			// 将 acceptor_ 设置为监听模式，准备接受传入的客户端连接
			acceptor_.listen();
			do_accept();
		}
```

首先初始化各个成员变量，绑定signals_，acceptor_，socket_，其中connection_manager_和request_handler_是自己封装的类。

然后，绑定**SIGINT**、**SIGTERM**信号，如果定义过**SIGQUIT**，同样将SIGQUIT绑定，如果这三个信号中任意一个被激活，执行**do_await_stop()**函数，使得服务器可以主动优雅地退出。

```c++
void server::do_await_stop()
{
	signals_.async_wait(
		[this](boost::system::error_code, int)
		{
			acceptor_.close(); // 关闭acceptor
			connection_manager_.stop_all(); // 移除所有客户端连接
		});
}
```

异步等待函数signals_.async_wait被调用后，主线程进行异步等待。

同时，定义一个resolver解析器，用于将将服务器域名解析为ip和端口号；并通过*resolver.resolve()方法将解析出来的ip和端口号解析为端点，方便acceptor绑定，并执行以下操作

```c++
acceptor_.open(endpoint.protocol()); // acceptor使用端点的协议
// 允许地址复用（reuse_address(true)）
// 作用是让多个程序或多个进程能够在地址已经被占用的情况下重用同一个地址和端口
acceptor_.set_option(boost::asio::ip::tcp::acceptor::reuse_address(true));
acceptor_.bind(endpoint);
// 将 acceptor_ 设置为监听模式，准备接受传入的客户端连接
acceptor_.listen();
```

然后调用do_accept()函数，执行异步连接async_accept，等待客户端的连接

```c++
void server::do_accept()
{
	acceptor_.async_accept(socket_,
		[this](boost::system::error_code ec)
		{
			// Check whether the server was stopped by a signal before this
			// completion handler had a chance to run.
			if (!acceptor_.is_open())
			{
				return;
			}

			if (!ec)
			{
				// 创建新的connection对象管理这个客户端连接，并将其插入connections_集合set中，然后开始读写
				connection_manager_.start(std::make_shared<connection>(
					std::move(socket_), connection_manager_, request_handler_));
			}

			do_accept();
		});
}
```

在do_accept()函数中，我们创建新的connection对象管理这个客户端连接，并将其插入**connections_（将每一个connection都插入connections_中管理）**集合set中，然后开始读写

```c++
connection_manager_.start(std::make_shared<connection>(
	std::move(socket_), connection_manager_, request_handler_));
```

connection是我们自己封装好的类，它的构造函数为

```c++
connection::connection(boost::asio::ip::tcp::socket socket,
	connection_manager& manager, request_handler& handler)
	: socket_(std::move(socket)),
	connection_manager_(manager),
	request_handler_(handler)
{}
```

connection_manager_也是我们封装好的类，connection_manager_.start()的执行过程如下

```c++
void connection_manager::start(connection_ptr c)
{
	connections_.insert(c);
	c->start();
}
```

将我们创建好的connection插入至connections_，并调用新连接的start函数进行客户端与服务器的读写（每一个connection负责一个客户端的连接，互相的独立的）

```c++
void connection::start()
{
	do_read();
}
```

**do_read()**如下：

```c++
void connection::do_read()
{
	auto self(shared_from_this());// 伪闭包，防止connection被销毁
	socket_.async_read_some(boost::asio::buffer(buffer_),
		[this, self](boost::system::error_code ec, std::size_t bytes_transferred)
		{
			if (!ec)
			{
				request_parser::result_type result;
				std::tie(result, std::ignore) = request_parser_.parse(
					request_, buffer_.data(), buffer_.data() + bytes_transferred);

				if (result == request_parser::good)
				{
					request_handler_.handle_request(request_, reply_);
					do_write();
				}
				else if (result == request_parser::bad)
				{
					reply_ = reply::stock_reply(reply::bad_request);
					do_write();
				}
				else
				{
					do_read();
				}
			}
			else if (ec != boost::asio::error::operation_aborted)
			{
				connection_manager_.stop(shared_from_this());
			}
		});
}
```

首先，C++通过智能指针实现伪闭包操作，防止防止connection被提取销毁，然后调用异步读函数，读取客户端发送的数据，读取成功后调用自定义的lambda函数。

如果读取错误，那么在lambda函数中调用**connection_manager_.stop(shared_from_this())**，减少connection的引用计数，使connection可以关闭连接。

如果没有错误，通过**request_parser_**解析请求，request_parser_也是我们封装好的类。

```c++
template <typename InputIterator>
std::tuple<result_type, InputIterator> parse(request& req,
	InputIterator begin, InputIterator end)
{
	while (begin != end)
	{
		result_type result = consume(req, *begin++);
		if (result == good || result == bad)
			return std::make_tuple(result, begin);
	}
	return std::make_tuple(indeterminate, begin);
}
```

**parse** 函数用于**解析 HTTP 请求**，它会调用内部的 consume 函数来逐行处理**请求头**的数据。consume 函数根据输入的字符情况，逐步解析 HTTP 请求头的内容，并且在解析的过程中根据每个字符的不同来判断当前的状态（state_）。

- consume 函数内部逐字符解析请求数据，并根据当前输入的字符和状态来进行相应操作。它的主要任务包括：
- **解析 HTTP 请求方法、URI 和 HTTP 版本号**：将这些信息填充到 request 结构体中。
- **解析请求头部字段的名称和值**：将每个字段的名称和值存入 request 结构体的 headers 向量中。
- **处理换行符**：如果检测到换行符（\r\n），则表示当前行已解析完毕，状态会更新以准备解析下一行。
- 最终，**consume** 函数会返回一个枚举类型 request_parser::**result_type** 作为解析结果。这个结果有以下三种状态
- **indeterminate**（未定）：表示当前还需要更多字符输入才能完成解析。
- **good**（成功）：表示成功解析出一个完整的 HTTP 请求头。
- **bad**（失败）：表示遇到无效字符或格式错误，解析失败。

**std::ignore**用于忽略不需要的值，占位但不实际接收某个返回值。作用类似python中的**'_'，**通常和std::tie 或 std::tuple 等配合使用。

```c++
enum result_type { good, bad, indeterminate };
```

consume函数如下，内部使用**switch-case**方法**逐字符**解析请求数据，并根据当前输入的字符和状态进行相应的操作。

```c++
request_parser::result_type request_parser::consume(request& req, char input)
{
	switch (state_)
	{
	case method_start:
		if (!is_char(input) || is_ctl(input) || is_tspecial(input))
		{
			return bad;
		}
		else
		{
			state_ = method;
			req.method.push_back(input);
			return indeterminate;
		}
	case method:
		if (input == ' ')
		{
			state_ = uri;
			return indeterminate;
		}
		else if (!is_char(input) || is_ctl(input) || is_tspecial(input))
		{
			return bad;
		}
		else
		{
			req.method.push_back(input);
			return indeterminate;
		}
	case uri:
		if (input == ' ')
		{
			state_ = http_version_h;
			return indeterminate;
		}
		else if (is_ctl(input))
		{
			return bad;
		}
		else
		{
			req.uri.push_back(input);
			return indeterminate;
		}
	case http_version_h:
		if (input == 'H')
		{
			state_ = http_version_t_1;
			return indeterminate;
		}
		else
		{
			return bad;
		}
	case http_version_t_1:
		if (input == 'T')
		{
			state_ = http_version_t_2;
			return indeterminate;
		}
		else
		{
			return bad;
		}
	case http_version_t_2:
		if (input == 'T')
		{
			state_ = http_version_p;
			return indeterminate;
		}
		else
		{
			return bad;
		}
	case http_version_p:
		if (input == 'P')
		{
			state_ = http_version_slash;
			return indeterminate;
		}
		else
		{
			return bad;
		}
	case http_version_slash:
		if (input == '/')
		{
			req.http_version_major = 0;
			req.http_version_minor = 0;
			state_ = http_version_major_start;
			return indeterminate;
		}
		else
		{
			return bad;
		}
	case http_version_major_start:
		if (is_digit(input))
		{
			req.http_version_major = req.http_version_major * 10 + input - '0';
			state_ = http_version_major;
			return indeterminate;
		}
		else
		{
			return bad;
		}
	case http_version_major:
		if (input == '.')
		{
			state_ = http_version_minor_start;
			return indeterminate;
		}
		else if (is_digit(input))
		{
			req.http_version_major = req.http_version_major * 10 + input - '0';
			return indeterminate;
		}
		else
		{
			return bad;
		}
	case http_version_minor_start:
		if (is_digit(input))
		{
			req.http_version_minor = req.http_version_minor * 10 + input - '0';
			state_ = http_version_minor;
			return indeterminate;
		}
		else
		{
			return bad;
		}
	case http_version_minor:
		if (input == '\r')
		{
			state_ = expecting_newline_1;
			return indeterminate;
		}
		else if (is_digit(input))
		{
			req.http_version_minor = req.http_version_minor * 10 + input - '0';
			return indeterminate;
		}
		else
		{
			return bad;
		}
	case expecting_newline_1:
		if (input == '\n')
		{
			state_ = header_line_start;
			return indeterminate;
		}
		else
		{
			return bad;
		}
	case header_line_start:
		if (input == '\r')
		{
			state_ = expecting_newline_3;
			return indeterminate;
		}
		else if (!req.headers.empty() && (input == ' ' || input == '\t'))
		{
			state_ = header_lws;
			return indeterminate;
		}
		else if (!is_char(input) || is_ctl(input) || is_tspecial(input))
		{
			return bad;
		}
		else
		{
			req.headers.push_back(header());
			req.headers.back().name.push_back(input);
			state_ = header_name;
			return indeterminate;
		}
	case header_lws:
		if (input == '\r')
		{
			state_ = expecting_newline_2;
			return indeterminate;
		}
		else if (input == ' ' || input == '\t')
		{
			return indeterminate;
		}
		else if (is_ctl(input))
		{
			return bad;
		}
		else
		{
			state_ = header_value;
			req.headers.back().value.push_back(input);
			return indeterminate;
		}
	case header_name:
		if (input == ':')
		{
			state_ = space_before_header_value;
			return indeterminate;
		}
		else if (!is_char(input) || is_ctl(input) || is_tspecial(input))
		{
			return bad;
		}
		else
		{
			req.headers.back().name.push_back(input);
			return indeterminate;
		}
	case space_before_header_value:
		if (input == ' ')
		{
			state_ = header_value;
			return indeterminate;
		}
		else
		{
			return bad;
		}
	case header_value:
		if (input == '\r')
		{
			state_ = expecting_newline_2;
			return indeterminate;
		}
		else if (is_ctl(input))
		{
			return bad;
		}
		else
		{
			req.headers.back().value.push_back(input);
			return indeterminate;
		}
	case expecting_newline_2:
		if (input == '\n')
		{
			state_ = header_line_start;
			return indeterminate;
		}
		else
		{
			return bad;
		}
	case expecting_newline_3:
		return (input == '\n') ? good : bad;
	default:
		return bad;
	}
}
```

解析完头部之后，会相应的结果执行相应的函数，比如执行do_read()或do_write()。

不过在调用读、写函数之前，需要调用处理请求的函数**handle_request**，或**stock_reply**

- **handle_request**函数用于处理 HTTP 请求，并根据请求的路径生成相应的回复，函数根据url中的‘**.**’来做切割，获取请求的文件类型，然后根据‘**/**’切割url，获取资源目录，最后返回资源文件
- 首先，对请求的 URL 进行解码，并验证路径是否合法（必须是绝对路径且不包含 ".."）。如果路径是目录，则默认返回 "index.html"。
- 然后，函数尝试在指定的文档根目录下找到对应的文件，如果找到则读取文件内容并设置响应的状态和头部信息（如内容长度和 MIME 类型）。如果文件不存在，则返回 404 错误响应。

我们也可以返回json或者text格式，可以重写处理请求的逻辑。

```c++
		reply reply::stock_reply(reply::status_type status)
		{
			reply rep;
			rep.status = status;
			rep.content = stock_replies::to_string(status);
			rep.headers.resize(2);
			rep.headers[0].name = "Content-Length";
			rep.headers[0].value = std::to_string(rep.content.size());
			rep.headers[1].name = "Content-Type";
			rep.headers[1].value = "text/html";
			return rep;
		}
```

如果解析头部返回的是**bad，那么就是执行**stock_reply函数，创建响应头，并对响应头的信息进行添加。

代码可参考博主恋恋风辰的个人仓库：

## 4. 总结

**1.acceptor_.listen()和acceptor_.async_accept()的区别？**

**新版本中，****boost::asio::ip::tcp::acceptor** 的构造函数已经可以自动调用 **listen** ，只通过以下代码即可实现对客户端的监听，而不用繁琐的进行各种规定：

```c++
 boost::asio::ip::tcp::acceptor a(ios, asio::ip::tcp::endpoint(asio::ip::tcp::v4(), 3333));
```

当你使用这种方式创建 **acceptor** 时，它会：

- 打开套接字。
- 绑定到给定的端点。
- 自动调用 **listen**。

这样写是简洁的，因为构造函数已经完成了所有必要的步骤。但是，如果你想要手动控制这些步骤，比如延迟调用 **listen** 或在绑定前设置更多的选项，那么显式调用 **listen** 就有它的意义。
