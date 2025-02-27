---
title: 网络编程（23）——使用grpc进行通信
date: 2024-11-03 12:22:00
categories:
- C++
- 网络编程
tags: 
- grpc
- grpc通信方式
- http2.0
- protobuf
typora-root-url: ./..
---

##  二十三、day23

windows环境下编译好grpc库之后，学习如何将grpc库配置到所用的项目中，步骤包含：

1）编写grpc服务定义（.proto后缀文件）；

2）环境配置；

3）简单的grpc服务器搭建

参考：

[一文掌握gRPC-CSDN博客](https://blog.csdn.net/qq_43456605/article/details/138647102)

[protobuf简介_protbuf-CSDN博客](https://blog.csdn.net/qq_62321047/article/details/140466202)

[恋恋风辰官方博客](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/2QSEHcC1he1RgiewYG93ilaAMiY)

## 1. grpc

### 1）概念

gRPC（Google Remote Procedure Call）是一个高性能的开源RPC框架，由Google开发。它基于**HTTP/2**协议，支持**多种语言**（如C++、Java、Python、Go等），并使用**Protocol Buffers**（protobuf）作为接口定义语言（IDL）来序列化和反序列化数据

grpc的设计思路：

- 协议：使用Http2协议。与HTTP/1.1的文本格式不同，HTTP/2： 
  - 传输数据使用二进制数据内容；
  - 允许在一个连接中并发处理多个请求和响应（连接的多路复用），避免了HTTP/1.1中的队头阻塞问题；
  - 支持双向流[双工]；
  - HTTP/2通常通过单一连接与服务器进行通信，减少了连接建立和管理的开销。
- 序列化：基于二进制（protobuf- 谷歌开源的一种序列化方式，之前学习的jsoncpp是基于文本格式存储，可读性比proto好，但效率不如proto）
- 代理的创建 ：让调用者像调本地方法一样去调用远端的方法
- 跨语言

**a. 跨语言**

> grpc可以实现**跨语言的通信**，比如服务器通过C++实现，客户端通过python实现，但二者仍然可以通信，实现了跨语言。

步骤如下：

- 定义 Proto 文件：定义服务和消息。
- 生成 gRPC 代码：使用 protoc 编译生成对应语言的 gRPC 代码。
- 实现 gRPC 服务器（C++）：编写服务器端代码。
- 实现 gRPC 客户端（Python）：编写客户端代码。
- 启动服务器和客户端：验证跨语言通信。

注意，生成grpc代码时，必须既生成C++的protobuf 数据结构代码和 gRPC 服务代码，还需要生成python的数据结构代码和grpc服务代码。

假设 .proto 文件名为 demo.proto，目录结构如下：

```
project/
│
├── demo.proto
├── cpp_output/
└── python_output/
```

那么生成命令为

```
# 生成 C++ 代码
protoc -I. --cpp_out=cpp_output --grpc_out=cpp_output --plugin=protoc-gen-grpc=grpc_cpp_plugin demo.proto

# 生成 Python 代码
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. demo.proto
```

这样，通过`cpp_out`和`--grpc_out`不仅会生成C++数据结构代码，而且还会生成C++的grpc服务代码。python同理，通过`python_out`和`grpc_python_out`”生成对应类型的代码。也可以将命令分成两步分别执行：

```
# 生成grpc代码
D:\cppsoft\grpc\visualpro\third_party\protobuf\Debug\protoc.exe  -I="." --grpc_out="." --plugin=protoc-gen-grpc="D:\cppsoft\grpc\visualpro\Debug\grpc_cpp_plugin.exe" "demo.proto"
# 生成数据结构代码
D:\cppsoft\grpc\visualpro\third_party\protobuf\Debug\protoc.exe --cpp_out=. "demo.proto"
```

我们可以自己选择指定目录下的proto编译器和插件编译代码。

**b. 代理的创建**

> **此外，grpc还非常适合服务之间的通信。一般有两种情况：1）分布式；2）不同的服务功能不同，需要进行服务直接的调用**

*分布式*

如果我们只启动了一个能容纳8000个连接的服务Server1，但是需要连接响应的客户端有一万多个，这时候我们一般需要启动多个服务，这些服务的功能基本类似。

*服务之间的调用*

但如果每个服务的功能都不同，那么就需要实现服务之间的调用，比如有四个服务分别执行以下功能：

- 订单服务：负责处理订单的创建、修改和删除。
- 用户服务：管理用户的注册、登录和信息更新。
- 支付服务：负责处理支付和账单。
- 库存服务：更新和管理库存信息。

当一个服务需要另一个服务的数据或功能时，它会通过**grpc服务之间的调用**来进行协作，对于调用者来说，就好像仅仅只执行了一个本地函数，但事实上这个调用是在网络上进行的，只不过过程被grpc封装好了，我们只需要像执行本地函数一样调用即可。

同时，每个服务之间也可以**使用不同的编程语言写**，因为grpc支持跨语言的服务。

### 2）三种HTTP协议

**Http1.0协议**：

- 基于请求/响应模式；
- 是一个短连接（无状态的协议，每个请求都是独立的，服务器不保留客户端的状态信息）；
- 每次请求都会建立一个新的TCP连接，请求完成后连接关闭；
- 不支持多路复用（即复用同一连接进行多个请求），在需要多个资源时，需要频繁建立和关闭连接；
- 传输文本格式的数据；
- 单工（只有客户端找服务端，而无法实现服务端的主动推送）。

> **Http底层使用TCP连接，而Tcp是长连接协议，为什么Http1.0是一个短连接协议？**
>  虽然TCP本身的特性可以支持长连接，但在HTTP/1.0中并未被利用。因为Http1.0的设计选择和默认行为导致其成功短连接协议，即客户端在发送请求后会建立一个新的TCP连接，并在接收到响应后立即关闭该连接。这种设计方式是因为当时服务器性能比较差，无法维持大量的长连接。

**Http1.1协议**：

- 同样基于请求/响应模式；
- 支持长连接，允许多个请求和响应通过同一TCP连接进行复用（**不是多路复用**，虽然允许多个请求在同一连接中发送，但每个请求仍需等待前一个请求完成后才能发送，这可能导致**队头堵塞**问题）。但该连接的时长有限，在维持一段时间后 Keepalived 字段后自动断开连接），基于此设计出websocket；
- 传输文本格式的数据；
- 管道化，支持请求的管道化，客户端可以在等待响应时发送多个请求，从而提高网络利用率。但需注意，仍可能存在**队头阻塞问题**。

> Http1.0和Http1.1协议在数据传输上均采用**文本格式**（后者的请求体可以是二进制数据），具有良好的可读性，但效率较低。此外，HTTP/1.x协议均不支持双工通信。在资源请求时，客户端需要发送多个请求并建立多个连接。例如，当客户端从服务器请求一个网页时，虽然可以一次获取页面，但该页面中包含的超链接、CSS和JS资源又需要发送新的请求，因此必须建立多个连接。

**Http2.0协议**：

- 二进制分帧：使用二进制格式代替文本格式，通过将数据分成小的帧进行传输，相比文本格式，该格式提高了数据解析的效率，但是可读性比较差；
- 多路复用：允许在同一TCP连接中**并发处理**多个请求和响应，避免了HTTP/1.x中的队头阻塞问题。多个请求可以同时进行，减少延迟；
- 双工通信：服务器可以主动向客户端推送资源，而不需要客户端先请求；
- 单一连接：通过单一连接与服务器进行通信

> **为什么Http2.0可以实现多路复用？**

Http2.0抽取了3个重要的概念，分别是数据流（Stream）、消息（Message）和帧（Frame），这三个概念有机的整合在一起就可以实现多路复用。如下图所示

![img](../images/$%7Bfiilename%7D/6cff6349858f41b59a6216525a4a9a03.png)

<center>图片来源：https://blog.csdn.net/qq_43456605/article/details/138647102</center>

- 在HTTP/2.0中，数据传输以Stream为基本单位，允许在一个连接上同时处理多个请求。例如，当请求一个页面时，可以通过一个连接复用三个Stream：一个Stream用于获取HTML页面，另两个分别用于请求CSS和JS资源。这种方式消除了在HTTP/1.x中需要为每个请求建立多个连接的需求。
- 每个Stream包含一个或多个Message，而每个Message又由两个或多个Frame组成。其中，一个Frame用于存放请求头，另一个Frame则包含请求体。响应也可以通过多个Stream发送，进一步提升了数据传输的效率和灵活性。通过这种复用机制，HTTP/2.0显著降低了延迟并提高了资源利用率。

![img](../images/$%7Bfiilename%7D/70a361c73cfc4bff8ff2c5f6a587cf3b.png)

<center>图片来源：https://blog.csdn.net/qq_43456605/article/details/138647102</center>

- 在请求Stream1中，包含了一个Message（RequestMessage），该Message包含了两个frame，其中一个帧保存了请求头，另一个帧中保存了请求体；
- 同样，响应Stream1中也包含了一个Message（ResponseMessage），该Message包含了两个frame，其中一个帧保存了响应头，另一个帧中保存了响应体；

> 1. 数据流的优先级：各个Stream有优先级，通过给不同的Stream设置权重，来限制不同流的传输顺序
> 2. 流控，如果客户端的流的发送速度大于服务端处理数据，导致服务端处理不过来，此时服务端可以通知客户端暂时停止发送流

### 3）protobuf序列化

之前也学习过protobuf，但只是简单的做了一些了解，今天详细介绍protobuf。

Protobuf是一种与编程语言无关且与平台无关的序列化协议，使用自定义的IDL（接口定义语言）来定义数据结构，方便在客户端和服务端进行RPC（远程过程调用）传输。Protobuf有两个主要版本：Protobuf2和Protobuf3，目前主流使用的是Protobuf3。在使用Protobuf时，需要借助**Protobuf编译器**（使用proto.exe 编译.proto后缀的文件），该编译器的作用是将Protobuf的IDL定义转换为具体的编程语言实现。这使得开发者可以在不同的编程语言中轻松地使用相同的数据结构，提高了跨平台和跨语言的兼容性与效率。

步骤：如下图所示

![img](../images/$%7Bfiilename%7D/b07cee461dc545ec8490a4d4b59a89d6.png)

<center>图片来源：https://blog.csdn.net/qq_62321047/article/details/140466202</center>

- 编写 `.proto` ⽂件，定义结构对象（message）及属性内容。
- 使⽤ protoc 编译器编译 `.proto` ⽂件，⽣成⼀系列接⼝代码，存放在新⽣成头⽂件和源⽂件中。
- 依赖⽣成的接⼝，将编译⽣成的头⽂件包含进我们的代码中，实现对 .proto ⽂件中定义的字段进行设置和获取，和对 message 对象进行序列化和反序列化

#### .proto文件写法

可在官网查看详细内容：

[Language Guide (proto 3) | Protocol Buffers Documentation](https://protobuf.dev/programming-guides/proto3/)

##### a. 指定 proto3

```
syntax = "proto3";
```

显示指定proto3为proto语法，默认为proto2.

##### b. package 声明符

```
package hello;
```

package 是⼀个可选的声明符，能表示 .proto ⽂件的**命名空间**，在项⽬中要有唯⼀性。它的作⽤是为了避免我们定义的消息出现冲突。

> **导入**：在实际的开发中我们可能有多个.proto，每个.proto管理自己的内容，现在可能出现一个问题，其中一个.proto依赖另一个.proto的内容，这里就需要使用导入语法：**import "xxx/Userservice.proto"**

##### c. 定义消息（message）

protobuf可以支持很多类型，比如基本数据类型、枚举类型、消息类型以及复合类型。

```
 1）枚举类型
enum Status {
    UNKNOWN = 0;
    ACTIVE = 1;
    INACTIVE = 2;
}

2）消息类型（可以嵌套）
message searchResponse{
	message Result{
		string url=1;
		string title=2;
	}
	string xxx=1;
	int32 yyy=2;
	Result ppp=3;
}

message AAA{
	string xxx=1;
	searchResponse.Result yyy=2;
}
3）复合类型
a. repeated
b. 其他嵌套
```

- 枚举的编号从0开始；
- message的编号从1开始到2^9-1结束，注意其中19000-19999不能用，这个区间内的编号是protobuf自己保留的；
- repeated关键字用于定义重复字段，即允许在消息中包含零个或多个同类型的值。它相当于数组或python的列表，可以用于存储多个相同类型的元素；

比如，使用repeated关键字定义一个message类型：

```
message Book {
    string title = 1;
    repeated string authors = 2; // 可以有多个作者
    repeated string tags = 3;    // 可以有多个标签
}
```

在这个Book消息中，authors和tags都是repeated字段，允许分别存储多个作者和标签；然后，通过protobufvi按一起生成相应的编程语言代码时，repeated字段会被映射为该语言中的集合类型。比如C++会将其映射为std::vector，在python中被映射为list。

##### d. 服务定义

服务的定义使用 **service** 关键字，后跟服务名称，并包含一个或多个方法的定义。每个方法需要指定请求类型和响应类型。比如：

```
service MyService {
    rpc MyMethod (RequestType) returns (ResponseType);
}
```

Protobuf支持多种类型的RPC调用：

- **简单RPC**：客户端发送一个请求，服务器返回一个响应（如上面的SayHello）。

- 流式 RPC：允许客户端和服务器之间的流式数据传输。 

  - **客户端流式**：客户端发送一个请求流，服务器返回一个响应。
  - **服务器流式**：客户端发送一个请求，服务器返回一个响应流。
  - **双向流式**：客户端和服务器可以同时发送和接收流。

##### e. option 关键字

option关键字用于设置不同的配置选项，以调整消息、字段、服务等的行为或属性。选项可以在多个层级上使用，包括全局、消息、字段、服务和方法级别。

```
syntax = "proto3";  // 使用Protobuf3语法

import "google/protobuf/descriptor.proto";  // 导入描述符定义

// 定义自定义选项
extend google.protobuf.MessageOptions {
    string my_custom_option = 50001;  // 自定义消息选项
}

extend google.protobuf.FieldOptions {
    string my_field_option = 50002;  // 自定义字段选项
}

extend google.protobuf.ServiceOptions {
    string my_service_option = 50003;  // 自定义服务选项
}

// 服务定义
service MyService {
    option (my_service_option) = "This is a custom service option.";  // 使用服务选项

    rpc GetPerson (GetPersonRequest) returns (PersonResponse);  // RPC方法定义
}

// 请求消息
message GetPersonRequest {
    int32 id = 1;  // 请求中包含一个ID字段
}

// 响应消息
message PersonResponse {
    option (my_custom_option) = "This is a custom option for PersonResponse.";  // 使用消息选项

    string name = 1 [(my_field_option) = "This is a custom field option for name."];  // 字段选项
    int32 age = 2;  // 另一个字段
}
```

- 自定义选项： 
  - 首先定义了几个自定义选项，通过extend关键字将其扩展到Protobuf内置选项（如MessageOptions、FieldOptions和ServiceOptions）。
  - 每个选项都定义了一个唯一的标识符（如50001、50002等），用于在代码中引用。
- 服务定义： 
  - 我们定义了一个名为MyService的服务，其中包含一个RPC方法GetPerson。
  - 在服务上使用了自定义选项my_service_option，为该服务添加了额外信息。
- 消息定义： 
  - GetPersonRequest消息中只包含一个ID字段。
  - PersonResponse消息中，使用了my_custom_option来自定义选项，同时在name字段上使用了my_field_option来附加更多的信息。

**一些默认的C++ option选项：**

```
# 启用或禁用内存池（arena）分配，默认值为false
option (cc_enable_arenas) = true;

# 是否生成 #pragma once 语句，以防止头文件多次包含，默认值为true
option (cc_include_once) = false;

# 设置生成代码的优化级别（如快速、空间等），默认值欸LITE（轻量级）
option (cc_optimize_for) = "Speed"; // 或 "Space"
```

还有一些别的，请参考grpc官网：

[Language Guide (proto 3)protobuf.dev/programming-guides/proto3/](https://link.zhihu.com/?target=https%3A//protobuf.dev/programming-guides/proto3/)

### 4）grpc项目组成

- xxxx-api模块：定义protobuf的idl语言，并且通过命令来创建具体的代码，后序服务端和客户端引入使用（客户端和服务端公共模块），包括定义message和service
- xxxx-server模块：服务提供方模块。实现API模块中定义的接口，需要和具体的业务相结合。然后发布整个gRPC服务（创建服务端程序）
- xxxx-client模块：创建服务端的代理，基于这个代理来进行RPC调用

### 5）gRPC的四种通信方式

| 服务类型       | 特点                                                         |
| -------------- | ------------------------------------------------------------ |
| 简单 RPC       | ⼀般的rpc调⽤，传⼊⼀个请求对象，返回⼀个返回对象            |
| 服务端流式 RPC | 传⼊⼀个请求对象，服务端可以返回多个结果对象                 |
| 客户端流式 RPC | 客户端传⼊多个请求对象，服务端返回⼀个结果对象               |
| 双向流式 RPC   | 结合客户端流式RPC和服务端流式RPC，可以传⼊多个请求对象，返回多个结果对象 |

##### a. 一元RPC

```
service HelloService{
  rpc hello(HelloRequest) returns (HelloResponse){
  }
}
```

- 客户端发起⼀次请求，服务端响应⼀个数据，即标准RPC通信。
- 这种模式，每⼀次都是发起⼀个独⽴的tcp连接，经历一次三次握⼿和四次挥⼿！

![img](../images/$%7Bfiilename%7D/8d833817306d4ffa9789ac135026b4b8.png)

<center>图片来源：https://blog.csdn.net/qq_43456605/article/details/138647102</center>

##### b. 服务端流式 RPC

```
service HelloService{
  rpc hello(HelloRequest) returns (stream HelloResponse){
  }
}
```

- 服务端流式rpc ⼀个请求对象，服务端可以传回多个结果对象
- 服务端流 RPC 下，客户端发出⼀个请求，但不会⽴即得到⼀个响应，⽽是在服务端与客户端之间建⽴⼀个单向的流，服务端可以随时向流中写⼊多个响应消息，最后主动关闭流，⽽客户端需要监听这个流，不断获取响应直到流关闭

例如股票系统中，客户端发送一个股票的编号到服务端（一个数据），服务端就会返回一系列数据，这些数据就是这个股票在某一时刻的行情。

![img](../images/$%7Bfiilename%7D/ab8ee9eaa646446aa93da013d4babd30.png)

<center>图片来源：https://blog.csdn.net/qq_43456605/article/details/138647102</center>

##### c. 客户端流式RPC

```
service HelloService{
  rpc hello(stream HelloRequest) returns (HelloResponse){
  }
}
```

- 客户端流式rpc 客户端传⼊多个请求对象，服务端返回⼀个响应结果
- 应用场景如：物联⽹终端向服务器报送数据

![img](../images/$%7Bfiilename%7D/c7571531c6bf432ea64f2fca6c9512ae.png)

<center>图片来源：https://blog.csdn.net/qq_43456605/article/details/138647102</center>

##### d. 双向流式 RPC

- 双向流式rpc 结合客户端流式rpc和服务端流式rpc，可以传⼊多个对象，返回多个响应对象
- 应⽤场景：聊天应⽤

![img](../images/$%7Bfiilename%7D/28f829406fe145878533ab7b60f39f56.png)

<center>图片来源：https://www.cnblogs.com/Mcoming/p/18080564</center>

## 2. 配置

### 2.1 编写grpc服务定义（.proto后缀文件）

```
syntax = "proto3";

package hello;

# 一元RPC
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

message HelloRequest {
  string message = 1;
}
message HelloReply {
  string message = 1;
}
```

该文件用于定义名为Greeter 的 **gRPC 服务**和两个**消息类型**：HelloRequest 和 HelloReply 。

- 指定使用 Protocol Buffers 的语法版本，这里使用的是 proto3 版本；
- 定义 protobuf 文件所属的包，包名 hello 将这个 proto 文件中定义的所有内容（包括服务和消息）归属于一个逻辑命名空间，防止与其他 proto 文件中的定义冲突；
- 定义名为 Greeter 的 gRPC 服务； 
  - service 关键字用于声明一个 gRPC 服务
  - 服务内部使用rpc 关键字定义了一个远程过程调用（Remote Procedure Call，RPC）的SayHello方法，该方法接受一个名为 HelloRequest 的消息作为请求参数，并返回一个名为 HelloReply 的消息作为响应。注意：该方法的实现必须为空，因为proto文件只定义接口，不实现逻辑
- 定义名为 HelloRequest 的消息类型； 
  - message 关键字用于声明一个消息类型
  - 这个消息类型用作 SayHello 方法的请求参数
  - string message = 1; 定义了一个名为 message 的字段，它是一个字符串类型（string），它在消息中的标识符是 1 ，用于标识该字段在二进制流中的位置
- 定义名为 HelloReply 的消息类型。 
  - 这个消息类型用作 SayHello 方法的响应
  - 定义一个字段message 用于存储服务器响应时发送的消息，1同样是消息中的标识符，用于标识该字段在二进制流中的位置

**a.** 我们需要使用proroc.exe基于msg.proto生成我们要用的C++类，在终端**cd**到demo.proto所处目录中，并输入如下命令生成dmo.proto的头文件和源文件：

```
D:\app\cppsoft\grpc\vs\third_party\protobuf\Debug\protoc.exe -I="." --grpc_out="." --plugin=protoc-gen-grpc="D:\app\cppsoft\grpc\vs\Debug\grpc_cpp_plugin.exe" "demo.proto"
```

- `-I="."` ：指定 demo.proto所在的路径为当前路径。
- `--grpc_out="."`： 表示生成的pb.h和`http://pb.cc`文件的输出目录。
- `grpc_cpp_plugin.exe`：要使用的插件为cpp插件，也就是生成cpp类的头文件和源文件。使用 grpc_cpp_plugin 插件来生成 gRPC 的服务端和客户端代码

不能使用之前学习单独编译的protobuf版本进行编译该文件，因为之前下载的protobuf库和现在下的grpc库不匹配，我们必须使用grpc库自己的proto.exe对该文件进行编译。注意，命令的路径应改为自己电脑中grpc库的目录路径。

然后会在该目录下生成`http://demo.grpc.pb.cc`和`demo.grpc.pb.h`文件，如下：

![img](../images/$%7Bfiilename%7D/format,png-1730607732038-281.png)

b. 这两个文件是给grpc服务的，我们还需要为消息类生成对应的`http://demo.pb.cc`和`demo.pb.h`文件：

```
D:\app\cppsoft\grpc\vs\third_party\protobuf\Debug\protoc.exe --cpp_out=. "demo.proto"
```

![img](../images/$%7Bfiilename%7D/format,png-1730607732038-282.png)

**区别：**这两行代码的作用不同，**前者**生成的是 gRPC 相关代码，使用 grpc_cpp_plugin 插件扩展了 Protobuf 的基本功能，用于实现 gRPC 服务的通信逻辑；**后者**则仅生成 Protobuf 消息的 C++ 代码，不涉及 gRPC 逻辑。这个命令生成的代码只包含消息结构的定义和操作函数。

如果我们只是使用protobuf消息代码进行序列化操作，我们只用执行第二个命令即可，但如果我们的 **.proto** 文件中定义了grpc服务，那么第一个和第二个命令都必须执行。

| 命令                                                         | 作用                                                   | 输出文件类型                                                 |
| ------------------------------------------------------------ | ------------------------------------------------------ | ------------------------------------------------------------ |
| protoc.exe -I="." --grpc_out="." --plugin=protoc-gen-grpc="grpc_cpp_plugin.exe" "demo.proto" | 生成 gRPC 服务代码，包括客户端和服务端的接口定义       | .grpc.pb.h 和 .[http://grpc.pb.cc](https://link.zhihu.com/?target=http%3A//grpc.pb.cc) |
| protoc.exe --cpp_out=. "demo.proto"                          | 生成 Protobuf 消息类型的序列化、反序列化和数据访问代码 | .pb.h 和 .[http://pb.cc](https://link.zhihu.com/?target=http%3A//pb.cc) |

### 2）visual studio配置

首先，右键项目->属性->VC++目录->包含目录，包含下面的目录

```
D:\app\cppsoft\grpc\third_party\re2
D:\app\cppsoft\grpc\include
D:\app\cppsoft\grpc\third_party\protobuf\src
D:\app\cppsoft\grpc\third_party\abseil-cpp
D:\app\cppsoft\grpc\third_party\address_sorting\include
```

![img](../images/$%7Bfiilename%7D/format,png-1730607732038-283.png)

然后，在库目录中，包含下面的目录

```
D:\app\cppsoft\grpc\visualpro\third_party\abseil-cpp\absl\profiling\Debug
D:\app\cppsoft\grpc\visualpro\third_party\abseil-cpp\absl\flags\Debug
D:\app\cppsoft\grpc\visualpro\third_party\abseil-cpp\absl\random\Debug
D:\app\cppsoft\grpc\visualpro\third_party\protobuf\Debug
D:\app\cppsoft\grpc\visualpro\third_party\re2\Debug
D:\app\cppsoft\grpc\visualpro\third_party\abseil-cpp\absl\types\Debug
D:\app\cppsoft\grpc\visualpro\third_party\abseil-cpp\absl\synchronization\Debug
D:\app\cppsoft\grpc\visualpro\third_party\abseil-cpp\absl\status\Debug
D:\app\cppsoftt\grpc\visualpro\third_party\abseil-cpp\absl\random\Debug
D:\app\cppsoftt\grpc\visualpro\third_party\abseil-cpp\absl\flags\Debug
D:\app\cppsoft\grpc\visualpro\third_party\abseil-cpp\absl\debugging\Debug
D:\app\cppsoft\grpc\visualpro\third_party\abseil-cpp\absl\container\Debug
D:\app\cppsoft\grpc\visualpro\third_party\abseil-cpp\absl\hash\Debug
D:\app\cppsoft\grpc\visualpro\third_party\boringssl-with-bazel\Debug
D:\app\cppsoft\grpc\visualpro\third_party\abseil-cpp\absl\numeric\Debug
D:\app\cppsoft\grpc\visualpro\third_party\abseil-cpp\absl\time\Debug
D:\app\cppsoft\grpc\visualpro\third_party\abseil-cpp\absl\base\Debug
D:\app\cppsoft\grpc\visualpro\third_party\abseil-cpp\absl\strings\Debug
D:\app\cppsoft\grpc\visualpro\third_party\protobuf\Debug
D:\app\cppsoft\grpc\visualpro\third_party\zlib\Debug
D:\app\cppsoft\grpc\visualpro\Debug
D:\app\cppsoft\grpc\visualpro\third_party\cares\cares\lib\Debug
```

![img](../images/$%7Bfiilename%7D/format,png-1730607732038-284.png)

最后，链接器->输入->附加依赖库，把用到的库文件写进去

```
json_vc71_libmtd.lib
libprotobufd.lib
gpr.lib
grpc.lib
grpc++.lib
grpc++_reflection.lib
address_sorting.lib
ws2_32.lib
cares.lib
zlibstaticd.lib
ssl.lib
crypto.lib
absl_bad_any_cast_impl.lib
absl_bad_optional_access.lib
absl_bad_variant_access.lib
absl_base.lib
absl_city.lib
absl_civil_time.lib
absl_cord.lib
absl_debugging_internal.lib
absl_demangle_internal.lib
absl_examine_stack.lib
absl_exponential_biased.lib
absl_failure_signal_handler.lib
absl_flags_config.lib
absl_flags_internal.lib
absl_flags_marshalling.lib
absl_flags_parse.lib
absl_flags_program_name.lib
absl_flags_usage.lib
absl_flags_usage_internal.lib
absl_graphcycles_internal.lib
absl_hash.lib
absl_hashtablez_sampler.lib
absl_int128.lib
absl_leak_check.lib
absl_log_severity.lib
absl_malloc_internal.lib
absl_periodic_sampler.lib
absl_random_distributions.lib
absl_random_internal_distribution_test_util.lib
absl_random_internal_pool_urbg.lib
absl_random_internal_randen.lib
absl_random_internal_randen_hwaes.lib
absl_random_internal_randen_hwaes_impl.lib
absl_random_internal_randen_slow.lib
absl_random_internal_seed_material.lib
absl_random_seed_gen_exception.lib
absl_random_seed_sequences.lib
absl_raw_hash_set.lib
absl_raw_logging_internal.lib
absl_scoped_set_env.lib
absl_spinlock_wait.lib
absl_stacktrace.lib
absl_status.lib
absl_strings.lib
absl_strings_internal.lib
absl_str_format_internal.lib
absl_symbolize.lib
absl_synchronization.lib
absl_throw_delegate.lib
absl_time.lib
absl_time_zone.lib
absl_statusor.lib
re2.lib
```

![img](../images/$%7Bfiilename%7D/format,png-1730607732038-285.png)

注意，配置和平台必须和编译器中的选项相同：

![img](../images/$%7Bfiilename%7D/format,png-1730607732038-286.png)

![img](../images/$%7Bfiilename%7D/format,png-1730607732038-287.png)

## 2. 实现grpc通信

搭建一个简单的grpc服务器和客户端进行通信。

### 1）实现grpc服务器

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <grpcpp/grpcpp.h>
#include "demo.grpc.pb.h"

using grpc::Server;
using grpc::ServerBuilder;
using grpc::ServerContext;
using grpc::Status;

// 通过.proto 文件编译成功的
using hello::HelloRequest;
using hello::HelloReply;
using hello::Greeter;

// 终极继承
class GreeterServiceImpl final :public Greeter::Service {
    virtual ::grpc::Status SayHello(::grpc::ServerContext* context,
        const ::hello::HelloRequest* request, ::hello::HelloReply* response) {
        std::string prefix("server has received: ");
        response->set_message(prefix + request->message());
        return Status::OK;
    }
};

void RunServer() {
    std::string server_address("0.0.0.0:50051");
    GreeterServiceImpl service;
    ServerBuilder builder;
    builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());
    builder.RegisterService(&service);

    std::unique_ptr<Server> server(builder.BuildAndStart());
    std::cout << "Server listening on " << server_address << std::endl;
    server->Wait();
}

int main(int argc, char** argv) {
    RunServer();
    return 0;
}
```

其中，GreeterServiceImpl类用于处理客户端发送的请求并返回相应的响应。它通过final关键字公有继承Greeter::Service(通过 gRPC 的 .proto 文件生成的基类)，该类被GreeterServiceImpl继承后无法被其他类继承。

SayHello方法的声明可以从.proto生成的文件中查找，这里我包含了该文件#include"demo.grpc.pb.h"”，从里面可以找到SayHello方法的具体声明。其中：

- ::grpc::ServerContext* context：提供有关 RPC 调用的上下文信息，比如身份验证、超时等；
- const ::hello::HelloRequest* request：从客户端接收到的请求对象，包含客户端发送的数据。在 .proto 文件中，HelloRequest 定义了一个 message 字段；
- ::hello::HelloReply* response：服务器需要填充并返回给客户端的响应对象。在 .proto 文件中，HelloReply 定义了一个 message 字段。

这里，我们从客户端接收请求，然后将消息进行回传，并返回一个状态：OK，表示RPC调用完成。

```cpp
void RunServer() {
    std::string server_address("0.0.0.0:50051");
    GreeterServiceImpl service;
    ServerBuilder builder;
    builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());
    builder.RegisterService(&service);

    std::unique_ptr<Server> server(builder.BuildAndStart());
    std::cout << "Server listening on " << server_address << std::endl;
    server->Wait();
}
```

我们通过RunServer()方法启动gRPC服务器。

- 首先，定义服务器的监听地址和端口号；
- 创建一个GreeterServiceImpl 对象**service**用于处理客户端的请求；
- 使用grpc提供的工具类，定义一个builder，用于配置并构建 gRPC 服务器实例； 
  - builder首先将服务器监听地址绑定，然后设置非安全凭证”，即服务器不会加密传输数据，也不使用任何SSL/TLS凭证。
  - builder将服务对象service注册到服务器构建器中，当客户端与服务器连接成功后，服务器将处理操作委托给GreeterServiceImpl实现。
- 调用 builder.BuildAndStart() 方法构建并启动 gRPC 服务器，方法返回一个 std::unique_ptr 指向一个 Server 对象，表示已经启动的服务器。其中，BuildAndStart() 方法会根据之前的配置创建一个服务器实例，并让它开始监听和处理请求。
- 最后，调用server->Wait() 方法阻塞当前线程，保持服务器的运行状态，直到服务器被显式关闭或遇到异常。服务器将在此期间持续监听客户端请求并进行处理。

### 2）实现grpc客户端

```cpp
#include <string>
#include <iostream>
#include <memory>
#include <grpcpp/grpcpp.h>
#include "demo.grpc.pb.h"
using grpc::ClientContext;
using grpc::Channel;
using grpc::Status;
using hello::HelloReply;
using hello::HelloRequest;
using hello::Greeter;

class FCClient {
public:
    FCClient(std::shared_ptr<Channel> channel)
        :stub_(Greeter::NewStub(channel)) {
    }
    std::string SayHello(std::string name) {
        ClientContext context;
        HelloReply reply;
        HelloRequest request;
        request.set_message(name);
        Status status = stub_->SayHello(&context, request, &reply);
        if (status.ok()) {
            return reply.message();
        }
        else {
            return "failure " + status.error_message();
        }
    }
private:
    std::unique_ptr<Greeter::Stub> stub_;
};

int main(int argc, char* argv[]) {
    auto channel = grpc::CreateChannel("127.0.0.1:50051", grpc::InsecureChannelCredentials());
    FCClient client(channel);
    // block until get result from RPC server
    std::string result = client.SayHello("hello , llfc.club !");
    printf("get result [%s]\n", result.c_str());
    return 0;
}
```

首先，定义一个名为 FCClient 的 gRPC 客户端类，它可以向 gRPC 服务器发送请求并接收响应。

```
FCClient(std::shared_ptr<Channel> channel)
```

- 构造函数 FCClient，其参数是 `std::shared_ptr<Channel> channel`，表示 gRPC 的通信通道（Channel 是客户端与服务器通信的核心）。这个通道封装了客户端与服务器的连接细节，允许客户端发送 RPC 请求。
- `:stub_(Greeter::NewStub(channel))`：这里的 stub_ 是一个 Greeter::Stub 对象，用于发起 RPC 请求。Greeter::NewStub(channel) 会根据 gRPC 框架生成的客户端代码，创建一个新的客户端存根（stub），它通过传入的 channel 连接到服务器。 
  - Greeter::Stub：这是 gRPC 自动生成的类，用于封装 gRPC 方法的客户端调用逻辑。

```
std::string SayHello(std::string name){}
```

该方法向服务器发送一个 SayHello 请求并获取响应。在函数体内，定义ClientContext 、HelloReply 和HelloRequest 的对象，然后再请求request设置想要添加的内容，最后调用stub_ 对象的 SayHello 方法，也就是我们编写的proto生成的SayHello 实现。

## 2. 项目在linux中测试

如何在linux中使用docker配置C++环境，并安装grpc请参考：

[爱吃土豆：如何使用docker在linux中配置C++环境1 赞同 · 0 评论文章![img](../images/$%7Bfiilename%7D/icon-default-1730607732042-290.png)https://zhuanlan.zhihu.com/p/1813068065](https://zhuanlan.zhihu.com/p/1813068065)

首先，启动配置好的docker环境，拉取博主恋恋风辰的代码仓库：

[secondtonone1/boostasio-learngitee.com/secondtonone1/boostasio-learn![img](../images/$%7Bfiilename%7D/icon-default-1730607732042-290.png)https://link.zhihu.com/?target=https%3A//gitee.com/secondtonone1/boostasio-learn](https://link.zhihu.com/?target=https%3A//gitee.com/secondtonone1/boostasio-learn)

然后cd至grpc代码目录下

```
cd /test/code/boostasio-learn/network/day19-Grpc-Server/day19-Grpc-Server/
```

![img](../images/$%7Bfiilename%7D/format,png-1730607732038-288.png)

这里已经有CMakeLists.txt文件存在，可以直接通过cmake编译或者使用g++编译。

```
mkdir build
cmake ..
make
./GrpcServer
```

![img](../images/$%7Bfiilename%7D/format,png-1730607732038-289.png)

服务器启动成功

然后，启动另一个终端，进入docker容器内，进入Client目录下，编译客户端代码

```
cd /test/code/boostasio-learn/network/day19-Grpc-Server/Grpc-Client/Grpc-Client
mkdir build
cd ./build
cmake ..
make
./GrpcServer
```

![img](/images/$%7Bfiilename%7D/format,png-1730608064359-325.png)
