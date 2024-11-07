---
title: 网络编程（21）——通过beast库快速实现http服务器
date: 2024-11-03 12:02:06
categories:
- C++
- 网络编程
tags: 
- http
- beast库
- http1.0
typora-root-url: ./..
---

## 二十一、day21

上一节学习了一个http服务器是如何实现的，以及每一步函数的实现过程。但上面大部分函数其实以及被封装好了，我们只需要调用就行了，今天跟着恋恋风辰大佬学习如何通过beast网络库快速搭建http服务器。

其实官网以及给出了如何通过beast快速搭建服务器的示例，可以参考boost官网“

[Chapter 1. Boost.Beastwww.boost.org/doc/libs/1_86_0/libs/beast/doc/html/index.html](https://link.zhihu.com/?target=https%3A//www.boost.org/doc/libs/1_86_0/libs/beast/doc/html/index.html)

视频参考：

【C++ 网络编程(22) beast网络库实现http服务器】

[https://www.bilibili.com/video/BV1Ck4y1T7na?vd_source=cb95e3058c2624d2641da6f4eeb7e3a1www.bilibili.com/video/BV1Ck4y1T7na?vd_source=cb95e3058c2624d2641da6f4eeb7e3a1](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1Ck4y1T7na%3Fvd_source%3Dcb95e3058c2624d2641da6f4eeb7e3a1)

## 1. 头文件和作用域重命名

```cpp
#include <boost/beast/core.hpp>
#include <boost/beast/http.hpp>
#include <boost/beast/version.hpp>
#include <boost/asio.hpp>
#include <ctime>
#include <cstdlib>
#include <memory>
#include <string>
#include <json/json.h>
#include <json/value.h>
#include <json/reader.h>
#include <iostream>

// 因为beast、http、net是作用域，所以可以直接用namespace重命名
// 但是tcp是一个类，所以只能通过using重命名
namespace beast = boost::beast;
namespace http = beast::http;
namespace net = boost::asio;
using tcp = boost::asio::ip::tcp;
```

## 2. reponse时调用的一些函数

```cpp
namespace my_program_state {
    std::size_t request_count() { // 统计请求的次数
        static std::size_t count = 0;
        return count++;
    }

    std::time_t now() { // 当前的时间
        return std::time(0);
    }
}
```

只封装两个函数，一个返回请求次数，一个返回当前的时间戳

## 3. http_connection

封装一个http_connection类用于处理 HTTP 请求和响应。负责接收客户端请求，解析请求内容，生成适当的响应，并将响应发送回客户端。同时，该类还管理连接的超时处理，确保长时间未响应的连接能够及时关闭

```cpp
class http_connection : public std::enable_shared_from_this<http_connection> {
private:
    tcp::socket _socket;
    beast::flat_buffer _buffer{ 8192 }; // 缓存区_buffer
    http::request<http::dynamic_body> _request;// 请求头
    http::response<http::dynamic_body> _response; // 回应
    net::steady_timer _deadline{ // 定时器
        _socket.get_executor(),
        std::chrono::seconds(60)
    };

    void read_request() {
    }

    void check_deadline() {
    }

    void process_request() {
    }

    void create_response() {
    }

    void write_response() {
    }

    void create_post_response() {
    }

public:
    http_connection(tcp::socket&& socket) : _socket(std::move(socket)) {}

    void start() {
        read_request(); // 读请求
        check_deadline(); // 判断超时
    }

};
```

http_connection类继承std::enable_shared_from_this<T>模板类，便于使用**shared_from_this()**函数实现伪闭包。http_connection类成员变量的解释如下：

- _buffer：定义一个缓存区，缓存区大小不超过8k

- _request：构造一个请求头，类型为http::request<http::

  dynamic_body

  \> 

  - **dynamic_body：**允许发送各种类型的请求
  - **string_body：**只允许发文本类型的请求

- _response：响应，同样也是dynamic_body类型

- _deadline：定时器，用于异步操作时的

  超时控制或延时任务

  ，使用的是steady_timer定时器对象，即使系统时间被改变，定时器仍能够按照预期的时间触发 

  - get_executor()：获取于socket相关的执行器，也就是说，定时器和socket共享相同的IO线程资源
  - std::chrono::seconds(60)：定时器在60s后被触发

成员函数的实现如下：

### a. 构造函数

```cpp
http_connection(tcp::socket socket) : _socket(socket) {}
```

该构造函数会报错，因为boost::asio::basic_stream_socket 类**不支持复制构造**。因此，在构造函数中尝试通过复制传递 socket 时，会发生编译错误，socket 只能通过**移动语义**来传递。

正确的构造函数如下：

```cpp
http_connection(tcp::socket&& socket) : _socket(std::move(socket)) {}
```

**为什么http_connection(tcp::socket socket)中按值传递不会调用复制构造函数，而是在初始化成员变量_socket(socket)时才调用复制构造函数？**

http_connection(tcp::socket socket) 中的参数 socket 是按值传递的，这意味着它是从调用者那里“拷贝”进来的。由于 tcp::socket **禁止复制，但支持移动**，所以编译器会**自动**尝试使用**移动构造函数**来初始化这个按值传递的参数。如果调用者传入的是一个临时对象（例如 std::move(socket) 或新创建的对象），那么这个按值传递的 socket 实际上是“移动”进来的，而不是复制进来的。

**为什么仅在 _socket(socket) 这里出错？**

在构造函数的初始化列表中，_socket(socket) 试图**复制**socket 参数，因为没有显式地使用 std::move。按值传递的参数 socket 本身是一个左值，要想移动它，必须显式地使用 std::move，将其转换为右值引用。这就是为什么必须在初始化列表中使用 std::move(socket)。

总结：

- 按值传递的 tcp::socket 是通过移动语义构造的，不会有问题。
- 但在初始化成员变量时，如果不显式使用 std::move，编译器会尝试复制 socket 参数，这是不允许的。因此，必须使用 std::move 使其变成右值，从而调用移动构造函数来初始化成员变量

除此之外，还有另外一种方式可以调用外界的socket，就是通过引用：

1）引用

引用绑定到已有的 socket 对象，并且始终指向这个对象，没有复制或移动。保证了在 http_connection 类中操作的 socket 与外部的 socket 是同一个对象，因此可以共享状态。

**但是**，由于 _socket 引用的是外部传入的对象，所以在 http_connection 对象的生命周期内，这个 socket 对象必须始终存在。如果 socket 对象被销毁了，引用就变成悬空引用，可能导致未定义行为。

```cpp
tcp::socket& _socket;
http_connection(tcp::socket& socket) : _socket(socket)
```

2）移动构造函数

但如果我们像http_connection独立的管理connection，而不需要接收外界传递进来的socket，那么如何做？

```cpp
tcp::socket _socket;
http_connection(tcp::socket socket) : _socket(std::move(socket))
```

或者这样写

```cpp
tcp::socket _socket;
http_connection(tcp::socket&& socket) : _socket(std::move(socket))
```

其实 **tcp::socket&&** 和 **tcp::socket** 完全不一样，一个是按值传递，一个是右值引用，但因为socket中禁用了复制构造函数，允许移动，所以编译器会自动尝试使用**移动构造函数**来初始化这个按值传递的参数，所以这里都是相同的。但在其他时候，不能这样用，这两种传递**完全不同**。

| 特性           | 引用 (tcp::socket&)                    | 移动语义 (tcp::socket&&)                      |
| -------------- | -------------------------------------- | --------------------------------------------- |
| 对象所有权     | 由外部对象管理                         | 由 http_connection 管理                       |
| 效率           | 无复制或移动，效率高                   | 通过移动语义高效转移对象所有权                |
| 生命周期依赖性 | 依赖于外部对象的生命周期               | 不依赖外部对象生命周期，自行管理              |
| 适用场景       | 适合共享已有对象并保证生命周期的一致性 | 适合 http_connection 需要完全控制 socket 对象 |

### b. start()

```cpp
    void start() {
        read_request(); // 读请求
        check_deadline(); // 判断超时
    }
```

http_connection类就是负责服务器与客户端的联系，如果server接收到一个新的客户端连接，那么就创建一个http_connection对象，并调用start函数开始读客户端的请求。

其中，read_request()函数的实现如下

```cpp
    void read_request() {
        auto self = shared_from_this(); // 伪闭包
        http::async_read(_socket, _buffer, _request,
            [self](beast::error_code ec, std::size_t bytes_transferred) {
                boost::ignore_unused(bytes_transferred); // 没用到bytes_transferred参数，编译器会警告，这里直接忽略掉
                if (!ec) {
                    self->process_request();
                }
            });
    }
```

调用异步读async_read函数从socket读取HTTP请求，数据存储至_buffer，解析后的数据存储至_request

- **_socket**: 表示与客户端的通信套接字，负责接收数据
- **_buffer**: 用于临时存储接收到的原始数据（未解析的字节流）
- **_request**: HTTP 请求对象，解析后的 HTTP 请求会存储在这个对象中

读取成功后调用lambda函数，因为async_read的回调函数中必须有**error_code** 和 **bytes_transferred**参数，但后者我们并没有在lambda函数体中使用到，为了避免编译器警告，通过**boost::ignore_unused**函数忽略该参数。当HTTP请求被成功读取后，调用**process_request**函数处理请求头。

```cpp
    void check_deadline() {
        auto self = shared_from_this(); // 伪闭包
        _deadline.async_wait([self](boost::system::error_code ec) {
            if (!ec) {
                self->_socket.close(ec);
            }
            });
    }
```

**check_deadline()** 通常用于实现超时处理。比如在一个网络服务中，客户端可能需要在一定时间内完成请求或响应。如果超过了这个时间（如60秒），定时器会触发，然后关闭客户端的连接。这样可以避免占用资源过久，确保服务器的稳定运行。

之前学的tcp是一个**长连接**，但今天学习的http是一个**短连接**，如果处理请求时间太长，我们应该主动将其连接断掉。

通过调用异步等待async_wait函数，用于等待定时器的超时事件发生。这个函数不会阻塞当前线程，而是让程序继续执行其他任务，当定时器时间到时，执行给定的回调函数。该回调函数用于关闭套接字，断开客户端与服务器的通信连接。

**该函数还有另外一种写法，你觉得对吗？**

```cpp
    void check_deadline() {
        auto self = shared_from_this(); // 伪闭包
        _deadline.async_wait([this](boost::system::error_code ec) {
            if (!ec) {
                this->_socket.close(ec);
            }
            });
    }
```

其实这个函数有风险存在，因为当调用check_deadline()时，需要过60s后判断超时后才会执行lambda函数，但是比如在58s在执行这个lambda函数时，http_connection因为某种原因（网络断开，引用计数减为0）被回收，那么lambda就会出错（this->_socket.close(ec);中this指向的空间发生了改变，那么这个close的执行是无效的，系统会崩溃）。所以必须将self传进来，让lambda捕获该参数，使其引用计数加一，实现伪闭包。

```cpp
        auto self = shared_from_this(); // 伪闭包
        _deadline.async_wait([sel,thisf](boost::system::error_code ec) {
            if (!ec) {
                this->_socket.close(ec);
            }
            });
```

还有，如果将self显式放在捕获列表中，那么无论是否在 lambda 体内使用，引用计数都会增加，因为捕获行为本身就会创建一个 shared_ptr 的拷贝。

**注意【self】和【=】的区别，后者虽然也会捕获self，但如果在函数体没有使用到的话，编译器会优化，不让其创建实例。**

### **c. process_request()**

该函数用于处理不同的HTTP请求类型（GET或POST），并根据请求的具体方法生成相应的响应数据，返回给客户端。

```cpp
    void process_request() {
        _response.version(_request.version());
        _response.keep_alive(false); // true是长连接,false是短连接
        switch (_request.method()) {
        case http::verb::get:
            _response.result(http::status::ok);
            _response.set(http::field::server, "Beast");
            create_response();
            break;
        case http::verb::post:
            _response.result(http::status::ok);
            _response.set(http::field::server, "Beast");
            create_post_response();
            break;
        default:
            _response.result(http::status::bad_request);
            _response.set(http::field::content_type, "text/plain");
            beast::ostream(_response.body()) << "Invalid request-method"
                << std::string(_request.method_string()) << "'";
            break;
        }

        write_response();
    }
```

首先，将响应对象**_response**的HTTP版本设置为与请求对象**_request**相同的版本**。**HTTP 请求和响应都包含一个版本号，通常是 HTTP/1.1 或 HTTP/1.0，该操作确保响应与请求的协议版本匹配。

然后设置为短连接，keep_alive 用于决定是否启用**长连接**。在 HTTP/1.1 中，长连接允许客户端在同一个 TCP 连接上发送多个请求。

- **false**: 表示使用短连接，即在发送完响应后，服务器将关闭连接。
- **true**: 表示启用长连接，连接不会在响应后立即关闭，客户端可以继续发送更多请求。

**最后，判断请求方法是GET还是POST**。

如果请求方法是GET，那么

- 设置响应状态为 `http::status::ok`（200 OK），表示请求成功
- 设置响应头字段 `server`，其值为 `"Beast"`，表明服务器使用了 `Beast` 库
- 调用 `create_response()` 来生成 GET 请求的响应数据（内容在 `create_response()` 中处理）

如果请求方法是POST，那么

- 设置响应状态为 `http::status::ok`，表示请求成功
- 同样设置 `server` 字段
- 调用 `create_post_response()`，处理 POST 请求的特定逻辑

如果是未被明确处理的 HTTP 方法（如 PUT、DELETE 等），默认行为是

- 设置响应状态为 `http::status::bad_request`，表示这是一个无效请求
- 设置响应头字段 `content-type` 为 `"text/plain"`，告知客户端响应的内容类型是纯文本
- 使用 `beast::ostream(_response.body())` 向响应**_response**的正文中写入一条错误消息，说明这个请求使用了无效的 HTTP 方法，并附上具体的请求方法字符串

### d. create_response()

该函数生成GET请求的响应数据。

```cpp
    void create_response() {
        if (_request.target() == "/count") {
            _response.set(http::field::content_type, "text/html");
            beast::ostream(_response.body())
                << "<html>\n"
                << "<head><title>Request count</title></head>\n"
                << "<body>\n"
                << "<h1>Request count</h1>\n"
                << "<p>There have been "
                << my_program_state::request_count()
                << " requests so far.</p>\n"
                << "</body>\n"
                << "</html>\n";
        }
        else if (_request.target() == "/time") {
            _response.set(http::field::content_type, "text/html");
            beast::ostream(_response.body())
                << "<html>\n"
                << "<head><title>Current time</title></head>\n"
                << "<body>\n"
                << "<h1>Current time</h1>\n"
                << "<p>The current time is "
                << my_program_state::now()
                << " seconds since the epoch.</p>\n"
                << "</body>\n"
                << "</html>\n";
        }
        else {
            _response.result(http::status::not_found);
            _response.set(http::field::content_type, "text/plain");
            beast::ostream(_response.body()) << "File not found\r\n";
        }
    }
```

1）判断请求的URL目标路径是否为**‘/count’**，表示客户端请求获取请求计数

- 在这种情况下，响应将设置内容类型为 text/html，表示返回的是一个 HTML 页面
- 然后通过 **beast::ostream(_response.body())** 将 HTML 数据写入到响应的正文中，内容包括页面的标题“Request count”和请求计数
- **my_program_state::request_count()** 返回当前的请求计数，该值会被插入到响应的 HTML 中，显示在页面上

生成的HTML如下：

```html
<html>
<head><title>Request count</title></head>
<body>
<h1>Request count</h1>
<p>There have been [请求计数] requests so far.</p>
</body>
</html>
```

2）判断请求的URL目标路径是否为**‘/time’**，表示客户端请求当前的系统时间

- 响应同样设置内容类型为 text/html
- 生成一个 HTML 页面，显示当前时间。时间通过 **my_program_state::now()** 来获取，返回自 Unix 纪元以来的秒数

生成的HTML如下：

```html
<html>
<head><title>Current time</title></head>
<body>
<h1>Current time</h1>
<p>The current time is [当前秒数] seconds since the epoch.</p>
</body>
</html>
```

3）判断请求的URL目标路径既不是/count，也不是/time，则认为该资源不存在

- 设置响应状态为 http::status::not_found，即 HTTP 404 Not Found
- 设置内容类型为 text/plain，表示返回的是纯文本
- 然后向响应正文写入一条简单的错误消息：File not found\r\n

### e. create_post_response()

该函数处理客户端发送的 **POST** 请求，主要处理 **/email** 路径的请求，并解析接收到的 JSON 数据。函数根据收到的 POST 数据生成不同的响应。

```cpp
void create_post_response() {
        if (_request.target() == "/email") {
            // 读取并打印收到的request
            auto& body = this->_request.body();
            auto body_str = boost::beast::buffers_to_string(body.data());
            std::cout << "receive body is " << body_str << std::endl;
            // 构造返回的response
            this->_response.set(http::field::content_type, "text/json");
            Json::Value root; // 发送的根
            Json::Reader reader;
            Json::Value src_root; // 原始的根
            // 解析数据
            bool parse_success = reader.parse(body_str, src_root);
            if (!parse_success) { // 解析失败
                std::cout << "Failed to parse JSON data!" << std::endl;
                root["error"] = 1001;
                std::string jsonstr = root.toStyledString();
                beast::ostream(this->_response.body()) << jsonstr; // 将数据写入body
                return;
            }
            // 解析成功
            auto email = src_root["email"].asString(); // 将收到的email转为string
            std::cout << "email is " << email << std::endl;
            root["error"] = 0;
            root["email"] = src_root["email"];
            root["msg"] = "recevie email post success";
            std::string jsonstr = root.toStyledString(); // 序列化root数据
            beast::ostream(this->_response.body()) << jsonstr; // 将数据写入body
        }
        else {
            _response.result(http::status::not_found);
            _response.set(http::field::content_type, "text/plain");
            beast::ostream(_response.body()) << "File not found\r\n";
        }
    }
```

1）如果客户端发送的POST请求的目标路径是**‘/email’：**

- 读取并打印请求的 body： 从 _request.body() 中获取请求体数据（通常是客户端通过 POST 方式发送的内容），并将其转换为字符串格式，然后打印
- 设置响应的内容类型：服务器将返回 JSON 格式的响应，因此设置响应的 Content-Type 为 **text/json**
- 解析 JSON 数据：使用 **Json::Reader** 对请求体中的字符串进行 JSON 解析，解析 **body_str** 中的 JSON 数据，并将其存储在 **src_root** 中。如果解析失败，parse_success 将为 false。
  - 如果解析失败，打印错误消息并构造带有错误码的 JSON 响应，比如

```cpp
{
  "error": 1001
}
```

- 如果解析成功，从 src_root 中提取 email 字段，并打印它
- 构造返回的 JSON 响应，包含 email 和成功消息
- 最后，将 root 数据序列化为字符串并写入响应体中

返回的 JSON 响应内容类似于：

```cpp
{
  "error": 0,
  "email": "example@example.com",
  "msg": "receive email post success"
}
```

2）如果客户端的 POST 请求目标路径不是 /email，服务器返回 HTTP 404 Not Found 错误

### f. write_response()

当系统解析完请求后并执行过GET/POST的相应操作生成响应后，执行write_response函数，将生成的 HTTP 响应异步写回客户端，并在发送完成后执行一些清理操作。

```cpp
    void write_response() {
        auto self = shared_from_this(); // 伪闭包
        _response.content_length(_response.body().size()); // 响应的长度
        http::async_write(_socket, _response, [self](beast::error_code ec, std::size_t) {
            // 只关闭服务器发送端，客户端收到服务器的响应后，也关闭客户端的发送端
            self->_socket.shutdown(tcp::socket::shutdown_send, ec);
            self->_deadline.cancel();
            });
    }
```

首先，设置响应的 Content-Length，即响应正文的长度，表示服务器将要发送的内容大小，并将该长度赋值到 HTTP 头部的 Content-Length 字段。

然后，调用异步写函数将响应_response发送到客户端，发送完成后关闭连接：

- 关闭 TCP 连接的**发送端**，这意味着服务器已经完成了发送操作，但连接仍然可以用于接收数据
- tcp::socket::shutdown_send 表示只关闭**发送端**，而保留接收端未关闭。这是为了遵循 TCP 的四次挥手协议：客户端在接收到服务器的响应后，应关闭自己的发送端，并最终关闭整个连接
- 在调用 shutdown() 时遇到错误，错误信息会存储在 ec 中，但不做额外处理

最后，取消定时器，表示当前操作已经完成，不再需要超时检测了，这样可以防止超时处理逻辑错误地关闭仍然活跃的连接。

## 4. Server

为了方便，server直接写为一个函数，不封装为类

```cpp
void http_server(tcp::acceptor& acceptor, tcp::socket& socket) {
    acceptor.async_accept(socket, [&](boost::system::error_code ec) {
        if (!ec) {
            // 创建一个http_connection新实例，并调用start函数
            std::make_shared<http_connection>(std::move(socket))->start();
        }

        http_server(acceptor, socket);
        });
}
```

调用async_accept函数异步等待客户端的连接，如果有新的连接，那么创建一个http_connection新实例，并调用start函数，读取客户端发送的请求头并执行相应操作。

## 5. 主函数

```cpp
int main()
{
    try {
        auto const address = net::ip::make_address("127.0.0.1");
        unsigned short port = static_cast<unsigned short>(8080);
        net::io_context ioc{ 1 };
        tcp::acceptor acceptor( ioc,{address,port} );
        tcp::socket socket(ioc);
        http_server(acceptor, socket);
        ioc.run();

    }
    catch (std::exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
        return EXIT_FAILURE;
    }
    return 0;
}
```

## 6. 测试

首先，从PostMan官网下载postman：

[Download Postman | Get Started for Freewww.postman.com/downloads/](https://link.zhihu.com/?target=https%3A//www.postman.com/downloads/)

### 1）测试get

然后启动服务器并打开浏览器的新页面，输入‘127.0.0.1:8080/count’，测试get。

服务器成功返回数据，并展示位HTML页面，当刷新页面后，count也会相应的增加

![img](/images/$%7Bfiilename%7D/89c3ab131786440fa3aa3e52fc217a37.png)

![img](/images/$%7Bfiilename%7D/fcb6b03300f648f2bd30af7fc9f1adea.png)





测试time，输入‘127.0.0.1:8080/time’，页面会显示当前的时间戳

![img](/images/$%7Bfiilename%7D/37b9ca9d55634aaaaf9b8068d9755c51.png)编辑![img](/images/$%7Bfiilename%7D/21be6155db264b2bbb42bf94697abf71.png)编辑





### 2）测试post

打开postman，按照图片进行如下操作

![img](/images/$%7Bfiilename%7D/a1d9c57d65ac417e9913213e833b1159.png)编辑



首先，创建一个新的请求，并设置位post，然后输入‘**127.0.0.1:8080/email**’，在raw中输入数据并点击发送

```html
{
    "email":"123456@edu.com"
}
```



服务器返回相应数据

![img](/images/$%7Bfiilename%7D/f338b249f79d44f4b8759aacd74e6097.png)编辑

![img](/images/$%7Bfiilename%7D/82ff42db2ba04b14b00de8fe3f6aab90.png)编辑





同时，服务器这里也打印处POST的相应数据

![img](/images/$%7Bfiilename%7D/c626ad88cae14b599498f55a7ed4a01e.png)编辑



同理，也可以用postman做get测试，我这里就不再演示了
