---
title: IO多路复用——select、poll、epoll
date: 2024-11-02 04:08:50
categories:
- 网络编程
tags: 
- 网络编程
typora-root-url: ./..
---

 **目录**

[1. 网络IO模型](#h_2632349361_0)

[1.1 信号驱动IO](#h_2632349361_1)

[1.2 同步阻塞IO](#h_2632349361_2)

[1.3 非阻塞IO](#h_2632349361_3)

[1.4 异步非阻塞IO](#h_2632349361_4)

[1.5 总结（什么是阻塞/非阻塞？什么是同步/异步？）](#h_2632349361_5)

[2. IO多路复用](#h_2632349361_6)

[2.1 什么是IO？](#h_2632349361_7)

[2.2 什么是 IO多路复用？](#h_2632349361_8)

[2.3 从阻塞 IO 到 IO 多路复用](#h_2632349361_9)

[2.4 文件描述符](#h_2632349361_10)

[3. 基础socket模型](#h_2632349361_11)

[4. select 模型（linux/windows）](#h_2632349361_12)

[4.1 原理](#h_2632349361_13)

[4.2 函数原型](#h_2632349361_14)

[4.3 fd_set操作过程](#h_2632349361_15)

[4.4 select实现网络通信流程](#h_2632349361_16)

[4.5 select使用示例](#h_2632349361_17)

[4.6 优缺点](#h_2632349361_18)

[5. poll 模型（linux）](#h_2632349361_19)

[5.1 函数原型](#h_2632349361_20)

[5.2 示例](#h_2632349361_21)

[6. epoll 模型（linux）](#h_2632349361_22)

[6.1 epoll函数](#h_2632349361_23)

[6.2 示例](#h_2632349361_24)

[6.3 epoll的工作模式](#h_2632349361_25)

[6.4 使用边沿非阻塞IO模式下的示例代码](#h_2632349361_26)

[6.5 关于边缘模式的问题](#h_2632349361_27)

[6.6 epoll的优点](#h_2632349361_28)

------



参考：

[IO多路复用，一文彻底搞懂！-51CTO.COMwww.51cto.com/article/794412.html![img](https://csdnimg.cn/release/blog_editor_html/release2.3.7/ckeditor/plugins/CsdnLink/icons/icon-default.png?t=O83A)https://link.zhihu.com/?target=https%3A//www.51cto.com/article/794412.html](https://link.zhihu.com/?target=https%3A//www.51cto.com/article/794412.html)

[勤劳的小手：100%弄明白5种IO模型1252 赞同 · 101 评论文章![img](../images/$%7Bfiilename%7D/icon-default-1730491750666-5.png)https://zhuanlan.zhihu.com/p/115912936](https://zhuanlan.zhihu.com/p/115912936)

[[操作系统\]IO多路复用 - Duancf - 博客园www.cnblogs.com/DCFV/p/18379027编辑![img](../images/$%7Bfiilename%7D/icon-default-1730491750666-5.png)https://link.zhihu.com/?target=https%3A//www.cnblogs.com/DCFV/p/18379027](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/DCFV/p/18379027)

[IO多路转接（复用）之selectsubingwen.cn/linux/select/#2-1-%E5%87%BD%E6%95%B0%E5%8E%9F%E5%9E%8B编辑![img](../images/$%7Bfiilename%7D/icon-default-1730491750666-5.png)https://link.zhihu.com/?target=https%3A//subingwen.cn/linux/select/%232-1-%25E5%2587%25BD%25E6%2595%25B0%25E5%258E%259F%25E5%259E%258B](https://link.zhihu.com/?target=https%3A//subingwen.cn/linux/select/%232-1-%E5%87%BD%E6%95%B0%E5%8E%9F%E5%9E%8B)

[程序员林夕：【面试】彻底理解 IO多路复用？2 赞同 · 0 评论文章编辑![img](../images/$%7Bfiilename%7D/icon-default-1730491750666-5.png)https://zhuanlan.zhihu.com/p/691368029](https://zhuanlan.zhihu.com/p/691368029)

[【操作系统】I/O 多路复用，select / poll / epoll 详解imageslr.com/2020/02/27/select-poll-epoll.html#whynonblock编辑![img](../images/$%7Bfiilename%7D/icon-default-1730491750666-5.png)https://link.zhihu.com/?target=https%3A//imageslr.com/2020/02/27/select-poll-epoll.html%23whynonblock](https://link.zhihu.com/?target=https%3A//imageslr.com/2020/02/27/select-poll-epoll.html%23whynonblock)

## 1. 网络IO模型

目前，常见的网络IO模型有五种，分别为：信号驱动IO（Signal-driven IO，SIO）、阻塞IO（Blocking IO，BIO）、非阻塞IO（Non-blocking IO，NIO）、异步非阻塞IO（Asynchronous IO，AIO）和IO复用（IO Multiplexing）。只有异步非阻塞IO是异步的，剩余四种方式都是同步。

### 1.1 信号驱动IO

信号驱动IO是一种通过信号通知程序IO事件的机制。在这种模型中，程序首先设置信号处理程序并将文件描述符配置为非阻塞模式。随后，当IO事件发生（如数据可读或可写）时，内核会向进程发送相应的信号（如SIGIO），程序的信号处理程序会被调用以执行实际的IO操作，这种方式避免了轮询的开销，提高了IO操作的效率。![img](../images/$%7Bfiilename%7D/6e016a46cac74957b51e4c1417f10df9.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑



图片来源：https://zhuanlan.zhihu.com/p/115912936

如上图，信号驱动IO不是用循环请求询问的方式去监控数据就绪状态，而是在调用sigaction时候建立一个SIGIO的信号联系，当内核数据准备好之后再通过SIGIO信号通知线程数据准备好后的可读状态，当线程收到可读状态的信号后，此时再向内核发起recvfrom读取数据的请求，因为信号驱动IO的模型下应用线程在发出信号监控后即可返回，不会阻塞。

该模型我们经常在服务器主动退出中使用，通过绑定异常信号SIGINT和SIGTERM，当服务器异步接收到异常信号时，关闭所有允许的线程。可以参考：

[爱吃土豆：网络编程（15）——服务器如何主动退出0 赞同 · 0 评论文章![img](../images/$%7Bfiilename%7D/icon-default-1730491750666-5.png)https://zhuanlan.zhihu.com/p/872943721](https://zhuanlan.zhihu.com/p/872943721)

### 1.2 同步阻塞IO

阻塞IO是一种简单的输入输出模型，调用该操作时会阻塞程序的执行，直到数据准备好或操作完成。这意味着在线程处理过程中，如果涉及到IO操作，那么当前线程会被阻塞，直到IO处理完成，线程才接着处理后续流程。如下图，服务器针对客户端的每个socket都会分配一个新的线程处理，每个线程的业务处理分2步，当步骤1处理完成后遇到IO操作(比如：加载文件)，这时候，当前线程会被阻塞，直到IO操作完成，线程才接着处理步骤2。![img](../images/$%7Bfiilename%7D/ac6438b52a9b45ebbb7b3f1a85e641c3.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑



图片来源：https://www.51cto.com/article/794412.html

实际使用场景：使用线程池的方式去连接数据库，使用的就是同步阻塞IO模型。 模型的缺点：因为每个客户端都需要一个新的线程，势必导致线程被频繁阻塞和切换带来开销。

### 1.3 非阻塞IO

同步非阻塞IO：在线程处理过程中，如果涉及到IO操作，那么当前的线程不会被阻塞，而是会去处理其他业务代码，然后等过段时间再来查询 IO 交互是否完成。如下图，当用户进程发起read调用时，如果内核的数据没有准备好，那么操作系统会返回一个EAGAIN error，用户进程可以根据该error判断出是数据未准备好，可以等会再来问。如果在轮询期间，内核准备好数据了，用户进程就可以把数据拷贝到用户态空间了。recvform复用一个线程，来查询已就绪的通道，这样大大减少 IO 交互引起的频繁切换线程的开销。![img](../images/$%7Bfiilename%7D/405388a951a14c279b342c7da80b07b1.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑



图片来源：https://blog.csdn.net/qq_43416206/article/details/131405607

从上图可以看出， 非阻塞IO模型需要应用进程不断地主动询问内核数据是否已准备好了。

- 优点:模型简单，实现难度低;与阻塞IO模型对比，它在等待数据报的过程中，进程并没有阻塞，它可以做其他的事情。
- 缺点:轮询发送 recvform，消耗CPU 资源

### 1.4 异步非阻塞IO

AIO是异步IO的缩写（Asynchronous IO）。与传统IO不同，异步非阻塞IO在**IO操作完成后**通知线程，而不是在准备好时通知。这意味着AIO不会阻塞，业务逻辑转变为一个回调函数，等待系统自动触发。![img](../images/$%7Bfiilename%7D/c29b26741a4a44049b3fc370d0a19310.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑



图片来源：https://zhuanlan.zhihu.com/p/115912936

如上图，应用告知内核启动某个操作，并让内核在整个操作完成之后，通知应用，这种模型与信号驱动模型的主要区别在于，信号驱动IO只是由内核通知我们合适可以开始下一个IO操作，而异步IO模型是由内核通知我们操作什么时候完成。

### **1.5 总结（**什么是阻塞/非阻塞？什么是同步/异步？**）**

异步IO的优化思路是为了提升应用程序的效率。在传统的IO模式下，应用程序通常需要经历两个阶段：

1. **发送询问请求**：应用程序先向内核询问是否有数据可用，这个步骤是阻塞的，程序会等待内核的响应。
2. **发送接收数据请求**：一旦确认有数据，应用程序再向内核请求接收数据。

这样的流程会导致不必要的等待，降低了程序的性能。

而在异步IO模式下，应用程序只需向内核发送一次请求。这个请求同时包含了状态查询和数据拷贝的操作，内核处理完成后会直接通知应用程序。这样，应用程序不必在两个步骤之间来回切换，从而减少了等待时间，提高了整体效率。这种方式特别适合处理大量并发IO操作的场景。

> **什么是阻塞/非阻塞？什么是同步/异步？**
>  阻塞的意思是，当发起读取数据请求时，如果数据尚未准备好，程序会等待数据就绪，这种情况称为**阻塞**；相反，如果请求可以立即返回而不等待，就是**非阻塞**。
>  在IO模型中，如果请求方在发起请求到数据完成的整个过程中都需要参与，这称为**同步请求**；如果应用发送完指令后不再参与，只等待最终结果的通知，则为**异步请求**。
>  同步阻塞和同步非阻塞的区别在于发起请求时，一个会阻塞等待，而另一个不会，但二者都需要应用监控数据完成的过程。至于异步模型，只有异步非阻塞，因为在异步模式下，发送请求后程序立即返回，没有后续流程，因此不会出现阻塞。

## **2. IO多路复用**

### **2.1 什么是IO？**

IO (Input/Output) 即数据的读取（接收）或写入（发送）操作，通常用户进程中的一个完整IO分为两阶段：用户进程空间<-->内核空间、内核空间<-->设备空间（磁盘、网络等）。IO有内存IO、网络IO和磁盘IO三种，通常我们说的IO指的是后两者。![img](../images/$%7Bfiilename%7D/7ddf754118e54b5db208e7f82b5c33bc.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑



图片来源：https://cloud.tencent.com/developer/article/1684951

LINUX中进程无法直接操作I/O设备，其必须通过系统调用请求kernel来协助完成I/O动作；内核会为每个I/O设备维护一个缓冲区。对于一个输入操作来说，进程IO系统调用后，内核会先看缓冲区中有没有相应的缓存数据，没有的话再到设备中读取，因为设备IO一般速度较慢，需要等待；内核缓冲区有数据则直接复制到进程空间。

所以，对于一个网络输入操作通常包括两个不同阶段：

- 等待网络数据到达网卡→读取到内核缓冲区，数据准备好；
- 从内核缓冲区复制数据到进程空间。

### 2.2 什么是 IO多路复用？

路：本意是道路，比如：城市的柏油路，乡村的泥巴路。那么：IO中的路是指什么呢？

在socket 编程中，**[ClientIp, ClientPort, ServerIp, ServerPort, Protocol]** 5元素可以唯一标识一个socket 连接，基于这个前提，同一个服务的某个端口 可以和 n个客户端建立socket连接，可以通过下图来大致描述：![img](../images/$%7Bfiilename%7D/2659ffdba085467689869887eb9d7a1e.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑



图片来源：https://www.51cto.com/article/794412.html

所以，每个客户端和服务器的socket 连接就可以看做“一路”，多个客户端和该服务器的socket连接就是”多路“，从而，**IO多路**就是多个socket连接上的输入输出流，**复用**就是多个socket连接上的输入输出流由一个线程处理。因此 Linux中的 **IO多路复用**可以定义如下：

> **一个线程处理多个IO流，即一个线程监视多个文件句柄；一旦某个文件句柄就绪，就能够通知应用程序进行相应的读写操作；没有文件句柄就绪时会阻塞应用程序，交出cpu。**

IO多路复用主要有select、poll、epoll模型

### **2.3 从阻塞 IO 到 IO 多路复用**

> **阻塞 IO->非阻塞IO->IO多路复用**

- 阻塞 I/O，是指进程发起调用后，会被挂起（阻塞），直到收到数据再返回。如果调用一直不返回，进程就会一直被挂起。因此，当使用阻塞 I/O 时，需要使用多线程来处理多个文件描述符。多线程切换有一定的开销，因此引入非阻塞 I/O。
- 非阻塞 I/O 不会将进程挂起，调用时会立即返回成功或错误，因此可以在一个线程里轮询多个文件描述符是否就绪。但是非阻塞 I/O 的缺点是：每次发起系统调用，只能检查一个文件描述符是否就绪。当文件描述符很多时，系统调用的成本很高。
- 引入I/O多路复用的目的是通过一次系统调用检查多个文件描述符的状态，这是它的主要优点。与非阻塞I/O相比，I/O多路复用可以在处理大量文件描述符时减少用户态和内核态之间的频繁切换，从而降低系统调用的开销。

具体来说，I/O多路复用将“***遍历所有文件描述符并通过非阻塞I/O检查其是否就绪\***”的工作交给内核来完成。程序可以使用 **select**、**poll** 或 **epoll** 等系统调用来实现这一功能。这些调用是**同步阻塞**的：***如果有文件描述符就绪，它们会返回这些描述符；如果都未就绪，调用会阻塞直到某个描述符就绪，或者超过设定的超时时间后返回\***。使用非阻塞I/O内部检查每个描述符的状态。

如果设置的超时参数为NULL，调用会无限阻塞直到有描述符就绪；如果设置为0，则会立即返回而不阻塞。尽管I/O多路复用引入了一些额外的开销，导致性能可能不如单线程直接处理I/O请求，但它允许在一个线程中同时处理多个I/O请求，避免了为每个请求创建新线程的开销。

> **为什么 I/O 多路复用内部需要使用非阻塞 I/O？**

I/O 多路复用内部会遍历集合中的每个文件描述符，判断其是否就绪：

```cpp
for fd in read_set
    if（ readable(fd) ) // 判断 fd 是否就绪
        count++
        FDSET(fd, &res_rset) // 将 fd 添加到就绪集合中
        break
...
return count
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

这里的 readable(fd) 就是一个非阻塞 I/O 调用。试想，如果这里使用阻塞 I/O，那么 fd 未就绪时，select 会阻塞在这个文件描述符上，无法检查下个文件描述符。

> **注意：**这里说的是 I/O 多路复用的内部实现，而不是说，使用 I/O 多路复用就必须使用非阻塞 I/O。在某些情况下，可以使用阻塞I/O与I/O多路复用结合（例如在特定的线程中进行处理）。

这部分内容可以详细看一下这篇文章，讲的非常好，我后面可能会将其翻译出来：

[Blocking I/O, Nonblocking I/O, And Epolleklitzke.org/blocking-io-nonblocking-io-and-epoll![img](../images/$%7Bfiilename%7D/icon-default-1730491750666-5.png)https://link.zhihu.com/?target=https%3A//eklitzke.org/blocking-io-nonblocking-io-and-epoll](https://link.zhihu.com/?target=https%3A//eklitzke.org/blocking-io-nonblocking-io-and-epoll)

### 2.4 文件描述符

启动一个进程就会得到一个对应的虚拟地址空间，这个虚拟地址空间分为两大部分，在内核区有专门用于进程管理的模块。Linux的进程控制块PCB（process control block）本质是一个叫做task_struct的结构体，里边包括管理进程所需的各种信息，其中有一个结构体叫做**file** ，我们将它叫做**文件描述符表**，里边有一个整形索引表,用于存储**文件描述符**。内核为每一个进程维护了一个文件描述符表，索引表中的值都是从0开始的，所以在不同的进程中你会看到相同的文件描述符，但是它们指向的不一定是同一个磁盘文件。

![img](../images/$%7Bfiilename%7D/5393ffe8b9744d28bab5182a3d256c7e.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑



图片来源：https://subingwen.cn/linux/file-descriptor/#2-2-%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6%E8%A1%A8

***2.4.1 文件描述符上限\***

每一个进程对应的文件描述符表能够存储的打开的文件数是有限制的，默认为**1024**个，这个默认值是可以修改的，支持打开的最大文件数据取决于操作系统的硬件配置。

***2.4.2 默认分配的文件描述符\***

当一个进程被启动之后，内核PCB的文件描述符表中就已经分配了三个文件描述符，这三个文件描述符对应的都是当前启动这个进程的终端文件（Linux中一切皆文件，终端就是一个设备文件，在 /dev 目录中）

在 Unix 和类 Unix 操作系统中，文件描述符是常见的 I/O 操作方式。标准文件描述符包括：

- 0: 标准输入（STDIN_FILENO）
- 1: 标准输出（STDOUT_FILENO）
- 2: 标准错误（STDERR_FILENO）

> 这三个默认分配的文件描述符是可以通过close()函数关闭掉，但是关闭之后当前进程也就不能和当前终端进行输入或者输出的信息交互了。

***2.4.3 新打开的文件如何分配文件描述符\***

因为进程启动之后，文件描述符表中的0,1,2就被分配出去了，因此**从3开始分配**。在进程中每打开一个文件，就会给这个文件分配一个新的文件描述符，比如：

- 通过open()函数打开 /hello.txt，文件描述符 3 被分配给了这个文件，**保持这个打开状态**，**再次**通过open()函数打开 /hello.txt，文件描述符 4 被分配给了这个文件，也就是说一个进程中不同的文件描述符打开的磁盘文件可能是同一个。
- 通过open()函数打开 /hello.txt，文件描述符 3 被分配给了这个文件，**将打开的文件关闭**，此时文件描述符**3就被释放了**。再次通过open()函数打开 /hello.txt，文件描述符 3 被分配给了这个文件，也就是说打开的**新文件会关联文件描述符表中最小的没有被占用的文件描述符**。

***2.4.4 socket\***

> **socket也是一种文件描述符：**socket通过系统调用创建后，会返回一个文件描述符，进程可以使用这个描述符进行网络通信

在tcp的服务器端, 有两类文件描述符：

- 监听的文件描述符
  - 只需要有一个
  - 不负责和客户端通信, 负责检测客户端的连接请求, 检测到之后调用accept就可以建立新的连接
- 通信的文件描述符
  - 负责和建立连接的客户端通信
  - 如果有N个客户端和服务器建立了新的连接, 通信的文件描述符就有N个，每个客户端和服务器都对应一个通信的文件描述符

客户端只有一个socket，用于与服务器建立连接和通信。

在我们之前异步服务器的实现中，服务器的监听通过新创建的session中的socket进行，服务器没有创建单独的socket进行监听；在线程池的实现中，服务器有自己单独的socket进行监听。

![img](../images/$%7Bfiilename%7D/43fa38c339624034a42b4c8263dde611.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑



图片来源：https://subingwen.cn/linux/socket/#4-1-%E6%9C%8D%E5%8A%A1%E5%99%A8%E7%AB%AF%E9%80%9A%E4%BF%A1%E6%B5%81%E7%A8%8B

- 文件描述符对应的内存结构

  ： 	

  - 一个文件文件描述符对应两块内存, 一块内存是读缓冲区, 一块内存是写缓冲区 	
    - 读数据: 通过文件描述符将内存中的数据读出, 这块内存称之为读缓冲区
    - 写数据: 通过文件描述符将数据写入到某块内存中, 这块内存称之为写缓冲区

- 监听的文件描述符

  : 

  - 客户端的连接请求会发送到服务器端监听的文件描述符的读缓冲区中
  - 读缓冲区中有数据, 说明有新的客户端连接
  - 调用accept()函数, 这个函数会检测监听文件描述符的读缓冲区

- 通信的文件描述符

  : 

  - 客户端和服务器端都有通信的文件描述符

  - 发送数据

    ：调用函数 write() / send()，数据进入到内核中 	

    - 数据并没有被发送出去, 而是将数据写入到了通信的文件描述符对应的写缓冲区中
    - 内核检测到通信的文件描述符写缓冲区中有数据, 内核会将数据发送到网络中

  - 接收数据

    : 调用的函数 read() / recv(), 从内核读数据 	

    - 数据如何进入到内核我们不需要处理, 数据进入到通信的文件描述符的读缓冲区中
    - 数据进入到内核, 必须使用通信的文件描述符, 将数据从读缓冲区中读出即可

参考：

[Linux 教程subingwen.cn/linux/#%E7%AC%AC1%E7%AB%A0-Linux-%E5%9F%BA%E7%A1%80编辑![img](../images/$%7Bfiilename%7D/icon-default-1730491750666-5.png)https://link.zhihu.com/?target=https%3A//subingwen.cn/linux/%23%25E7%25AC%25AC1%25E7%25AB%25A0-Linux-%25E5%259F%25BA%25E7%25A1%2580](https://link.zhihu.com/?target=https%3A//subingwen.cn/linux/%23%E7%AC%AC1%E7%AB%A0-Linux-%E5%9F%BA%E7%A1%80)

## 3. 基础socket模型

先介绍基础socket模型，然后在介绍IO多路复用模型，形成对比更容易理解。基础socket通信伪代码如下：

```cpp
listenSocket = socket(); //系统调用socket()函数，调用创建一个主动socket
bind(listenSocket);  //给主动socket绑定地址和端口
listen(listenSocket); //将默认的主动socket转换为服务器使用的被动socket(也叫监听socket)
while (true) { //循环监听客户端连接请求
   connSocket = accept(listenSocket); //接受客户端连接，获取已连接socket
   recv(connsocket); //从客户端读取数据，只能同时处理一个客户端
   send(connsocket); //给客户端返回数据，只能同时处理一个客户端
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

![img](../images/$%7Bfiilename%7D/bc38f20f9aec48ee889ed61b66a0fabd.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑

图片来源：https://www.51cto.com/article/794412.html

基础socket模型，能够实现服务器端和客户端之间的通信，但是程序每调用一次 accept 函数，只能处理一个客户端连接，当有大量的客户端连接时，这种模型处理性能比较差。因此 Linux 提供了高性能的IO多路复用机制来解决这种困境。

> 这种基础socket模型看起来和我们之前学习的asio实现的服务器步骤相似，但其实并**不相同**。asio中io_context.run()会根据平台选择相应的IO多路复用模型执行，只不过这些步骤都被隐藏了。asio底层通信可以参考文章：

[爱吃土豆：网络编程（12）——完善粘包处理操作+asio底层通信过程3 赞同 · 0 评论文章![img](../images/$%7Bfiilename%7D/icon-default-1730491750666-5.png)https://zhuanlan.zhihu.com/p/745782212](https://zhuanlan.zhihu.com/p/745782212)

## 4. select 模型（linux/windows）

### 4.1 原理

select使用一个固定大小的**位图**（bitmask）来表示文件描述符集合。每一位对应一个文件描述符，设置为1表示该描述符被监控，设置为0则不监控。由于位图的大小限制，文件描述符的数量通常不超过**1024**。

每次调用 select 实际上都是把Bitmap从用户态拷贝到内核态，然后在内核态里遍历这些文件描述符，检查这些文件描述符是否已经就绪，并统计其中有多少个文件描述符已经准备好，在Bitmap里把就绪的文件描述符位置修改成1，最后把修改好的Bitmap和就绪数量返回给用户态。用户态现在再去遍历Bitmap就可以知道哪些文件描述符已经就绪，就可以去处理那些已经就绪的文件描述符了。

**select执行原理**：

- 将当前进程的所有文件描述符，一次性的从用户态拷贝到内核态；
- 在内核中快速的无差别遍历每个fd，判断是否有数据到达；
- 将所有fd状态，从内核态拷贝到用户态，并返回已就绪fd的个数；
- 在用户态遍历判断具体哪个fd已就绪，然后进行相应的事件处理。

**缺点**：

- 总是需要拷贝位图，而且每次从内核态返回的位图都会覆盖掉之前的位图，再次使用需要初始化
- 位图的大小有限制，只有1024，也就是说，只能监听1024个文件描述符
- 从内核态返回位图后，用户态还是需要遍历位图来判断哪个文件描述符已经准备好了

### 4.2 函数原型

```cpp
#include <sys/select.h>
struct timeval {
    time_t      tv_sec;         /* seconds */
    suseconds_t tv_usec;        /* microseconds */
};

int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval * timeout);
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

1）**nfds**：委托内核检测的这三个集合中**最大的文件描述符（0~1024）** + 1

- 内核需要线性遍历这些集合中的文件描述符，这个值是循环结束的条件
- 在Window中这个参数是无效的，指定为-1即可

2）**readfds、writefds、errorfds** 是三个文件描述符**集合**。select 会遍历每个集合的**前 nfds 个**描述符，分别找到可以读取、可以写入、发生错误的描述符，统称为“**就绪**”的描述符。然后用找到的子集替换参数中的对应集合，返回所有就绪描述符的总数。也就是检测这些文件描述符对应的读写缓冲区的状态：

- **读缓冲区**（readfds）：检测里边有没有数据，如果有数据该缓冲区对应的文件描述符就绪
- **写缓冲区**（writefds**）**：检测写缓冲区是否可以写(有没有容量)，如果有容量可以写，缓冲区对应的文件描述符就绪
- **读写异常**（errorfds**）**：检测读写缓冲区是否有异常，如果有该缓冲区对应的文件描述符就绪

3）**timeout** 参数表示调用 select 时的阻塞时长。如果所有文件描述符都未就绪，就阻塞调用进程，直到某个描述符就绪，或者阻塞超过设置的 timeout 后，返回。如果 timeout 参数设为 NULL，会无限阻塞直到某个描述符就绪；如果 timeout 参数设为 0，会立即返回，不阻塞。

4）**返回值**：

- 大于0：成功，返回集合中已就绪的文件描述符的总个数
- 等于-1：函数调用失败
- 等于0：超时，没有检测到就绪的文件描述符

### 4.3 fd_set操作过程

在select()函数中第2、3、4个参数都是**fd_set**类型，它表示一个文件描述符的**集合**，这个类型的数据有128个字节，也就是1024个标志位，和内核中文件描述符表中的文件描述符个数是一样的。这意味着，这块内存中的每一个bit 和 文件描述符表中的每一个文件描述符是一一对应的关系，这样就可以使用最小的存储空间将要表达的意思描述出来。

下图中的fd_set中存储了**要委托内核检测读缓冲区的文件描述符集合**。

- 如果集合中的标志位为**0**代表不检测这个文件描述符状态
- 如果集合中的标志位为**1**代表检测这个文件描述符状态

首先，将需要内核检测读缓冲区的文件描述符集合 **fd_set \*readfds** 传给内核，然后内核检查传进来的读集合对应的文件描述符，并对 **fd_set** 集合进行修改。

- 图一就是表示传进来的读文件描述符集合，哪些文件描述符需要修改，1表示需要检测这个文件描述符状态，0表示不需要检测，这里需要检测fd3\fd5\fd6\fd8\fd9、fd10\fd1023,并检测对应文件描述符的读缓冲区。
- 图二表示将检测结果重新写入fd_set中，因为有的需要检测的文件描述符中读缓冲区并没有数据，所以应该将fd_set中对应的标志位置为0，select函数结束后，表示新的fd_set也被传出。用户态可根据传出的fd_set进行想要的读操作。



![img](../images/$%7Bfiilename%7D/format,png.png)

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑

图片来源：https://subingwen.cn/linux/select/#2-2-%E7%BB%86%E8%8A%82%E6%8F%8F%E8%BF%B0

内核在遍历这个读集合的过程中，如果被检测的文件描述符对应的**读缓冲区**中没有数据，内核将修改这个文件描述符在**读集合fd_set**中对应的标志位，改为0，如果有数据那么这个标志位的值不变，还是1（标准是判断文件描述符的读缓冲区中是否有数据）。



![img](../images/$%7Bfiilename%7D/format,png-1730491750662-1.png)

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑

图片来源：https://subingwen.cn/linux/select/#2-2-%E7%BB%86%E8%8A%82%E6%8F%8F%E8%BF%B0

当 **select() 函数**解除阻塞之后，被内核修改过的读集合通过参数传出，此时集合中只要标志位的值为1，那么它对应的文件描述符肯定是就绪的，我们就可以基于这个文件描述符和客户端建立新连接或者通信了

同理，写集合fd_set *writefds和异常集合fd_set *exceptfds的过程和读集合fd_set *readfds的操作过程相似。

> **fd_set 的使用涉及以下几个 API**：

```cpp
#include <sys/select.h>
int FD_ZERO(int fd, fd_set *fdset);  // 将 fd_set 所有位置 0
int FD_CLR(int fd, fd_set *fdset);   // 将 fd_set 某一位置 0
int FD_SET(int fd, fd_set *fd_set);  // 将 fd_set 某一位置 1
int FD_ISSET(int fd, fd_set *fdset); // 检测 fd_set 某一位是否为 1
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 4.4 select实现网络通信流程



![img](../images/$%7Bfiilename%7D/format,png-1730491750663-2.png)

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑

图片来源：https://subingwen.cn/linux/select/#3-1-%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B

1~3）前三步是所有服务器都需要进行的步骤，这里就不进行叙述，从第四步开始。

4）创建一个文件描述符集合fd_set，用于存储**需要检测读事件**（不需要检测的不设置）的所有的文件描述符

- 通过 FD_ZERO() 初始化
- 通过 FD_SET() 将监听的文件描述符放入检测的读集合中

5）**循环调用**select()，周期性的对所有的文件描述符轮询检测

- 如果超时，关闭socket；除此之外还可以继续调用select不关闭socket
- 使用FD_ISSET函数检测fd_set，是否存在某个元素标志位为1 
  - 如果该文件描述符是属于监听的文件描述符，调用 accept() 和客户端建立连接。并将得到的新的通信的文件描述符，通过FD_SET() 放入到检测集合中（下一次调用select便会检测该字符描述符）。继续调用select。
  - 如果这个文件描述符是通信的文件描述符，调用通信函数和客户端通信 	
    - 如果客户端和服务器断开了连接，使用FD_CLR()将这个文件描述符从检测集合中删除
    - 如果没有断开连接，正常通信

简化的流程：![img](../images/$%7Bfiilename%7D/1d273b8dac154f1cacc527f83bb75655.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑



### 4.5 select使用示例

***4.5.1 服务器\***

```cpp
int main() {
  /*
   * 这里进行一些初始化的设置，
   * 包括socket建立，地址的设置等,
   */

  fd_set read_fs, write_fs;
  struct timeval timeout;
  int max = 0;  // 用于记录最大的fd，在轮询中时刻更新即可

  // 初始化比特位
  FD_ZERO(&read_fs);
  FD_ZERO(&write_fs);

  int nfds = 0; // 记录就绪的事件，可以减少遍历的次数
  while (1) {
    // 阻塞获取
    // 每次需要把fd从用户态拷贝到内核态
    nfds = select(max + 1, &read_fd, &write_fd, NULL, &timeout);
    // 每次需要遍历所有fd，判断有无读写事件发生
    for (int i = 0; i <= max && nfds; ++i) {
      if (i == listenfd) { // 如果是监听事件的文件描述符
         nfds--;
         // 这里处理accept事件
         FD_SET(i, &read_fd);//将客户端socket加入到集合中
      }
      if (FD_ISSET(i, &read_fd)) { // 检测读事件的文件描述符
        --nfds;
        // 这里处理read事件
      }
      if (FD_ISSET(i, &write_fd)) { // 检测写事件的文件描述符
         --nfds;
        // 这里处理write事件
      }
    }
  }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

***4.5.2 客户端\***

客户端不需要使用IO多路复用，因为客户端和服务器的对应关系是 1：N，也就是说客户端是比较专一的，只能和一个连接成功的服务器通信。

### 4.6 优缺点

**优点：**

- 跨平台，可以在windows和linux中使用
- 简单易懂

**缺点：**

- 首先，select()函数对单个进程能监听的文件描述符数量是有限制的，它能监听的文件描述符个数由 __FD_SETSIZE 决定，默认值是 1024。
- 其次，当 select 函数返回后，需要遍历描述符集合，才能找到就绪的描述符。这个遍历过程会产生一定开销，从而降低程序的性能。

## 5. poll 模型（linux）

poll和select的区别在于，select使用位图来标记想关注的文件描述符，使用三个位图来标记想关注的读事件，写事件，错误事件。poll使用一个结构体pollfd数组来标志想关注的文件描述符和在这个描述符上感兴趣的事件，poll的优点是数组的长度突破了1024的限制，其他的区别不大。

poll和select相比有以下两点不同：

- poll 在用户态通过**数组方式**传递文件描述符，在内核会转为**链表方式**存储，没有最大数量的限制。
- select可以跨平台使用，poll只能在Linux平台使用

### 5.1 函数原型

```cpp
/**
* 参数 *__fds 是 pollfd 结构体数组，pollfd 结构体里包含了要监听的描述符，以及该描述符上要监听的事件类型
* 参数 __nfds 表示的是 *__fds 数组的元素个数
*  __timeout 表示 poll 函数阻塞的超时时间
*/
int poll (struct pollfd *__fds, nfds_t __nfds, int __timeout);

struct pollfd {
  int fd;          //进行监听的文件描述符
  short int events; //要监听的事件类型
  short int revents; //实际发生的事件类型
};
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

- fds

  ： struct pollfd类型的数组（

  也可以是链表

  ，取决于实现方式）, 里边存储了待检测的文件描述符的信息，这个数组中有三个成员： 	

  - fd：委托内核检测的文件描述符
  - events：委托内核检测的fd事件（输入、输出、错误），每一个事件有多个取值
  - revents：这是一个传出参数，数据由内核写入，存储内核检测之后的结果![img](../images/$%7Bfiilename%7D/8ad113ad5c004ed9b7ec6f45b5edc281.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑



图片来源：https://subingwen.cn/linux/poll/#1-poll%E5%87%BD%E6%95%B0

- **nfds**： 这是第一个参数数组中最后一个有效元素的下标 + 1（也可以指定参数1数组的元素总个数）

- timeout

  ：指定poll函数的阻塞时长 

  - -1：一直阻塞，直到检测的集合中有就绪的文件描述符（有事件产生）解除阻塞
  - 0：不阻塞，不管检测集合中有没有已就绪的文件描述符，函数马上返回
  - 大于0：阻塞指定的毫秒（ms）数之后，解除阻塞

- 函数返回值：

  - 失败： 返回-1
  - 成功：返回一个大于0的整数，表示检测的集合中已就绪的文件描述符的总个数

### 5.2 示例

```cpp
int sock_fd,conn_fd; //监听套接字和已连接套接字的变量
sock_fd = socket() //创建套接字
bind(sock_fd)   //绑定套接字
listen(sock_fd) //在套接字上进行监听，将套接字转为监听套接字

//poll函数可以监听的文件描述符数量，可以大于1024
#define MAX_OPEN = 2048

//pollfd结构体数组，对应文件描述符
struct pollfd client[MAX_OPEN];

//将创建的监听套接字加入pollfd数组，并监听其可读事件
client[0].fd = sock_fd;
client[0].events = POLLRDNORM;
maxfd = 0;

//初始化client数组其他元素为-1
for (i = 1; i < MAX_OPEN; i++)
   client[i].fd = -1;

while(1) {
    //调用poll函数，检测client数组里的文件描述符是否有就绪的，返回就绪的文件描述符个数
    n = poll(client, maxfd+1, &timeout);
    //如果监听套件字的文件描述符有可读事件，则进行处理
    if (client[0].revents & POLLRDNORM) {
         //有客户端连接；调用accept函数建立连接
         conn_fd = accept();
    
           //保存已建立连接套接字
           for (i = 1; i < MAX_OPEN; i++){
             if (client[i].fd < 0) {
               client[i].fd = conn_fd; //将已建立连接的文件描述符保存到client数组
               client[i].events = POLLRDNORM; //设置该文件描述符监听可读事件
               break;
              }
           }
           maxfd = i;
    }

    //依次检查已连接套接字的文件描述符
    for (i = 1; i < MAX_OPEN; i++) {
       if (client[i].revents & (POLLRDNORM | POLLERR)) {
         //有数据可读或发生错误，进行读数据处理或错误处理
         sockfd = fds[i].fd
         if ((n = read(sockfd, buf, MAXLINE)) <= 0) {
            // 这里处理read事件
            if (n == 0) {
                close(sockfd);
                fds[i].fd = -1;
            }
         } else {
             // 这里处理write事件     
         }
         if (--nfds <= 0) {
            break;       
         }   
      }
    }
  }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

> poll 的实现和 select 非常相似，只是poll 使用 pollfd 结构，而 select 使用fd_set 结构，poll **解决了**文件描述符**数量限制**的问题，但是同样需要从用户态拷贝所有的 fd 到内核态，也需要线性遍历所有的 fd 集合，所以它和 select 并没有本质上的区别（**仍要消耗大量资源**）。

poll实现网络通信流程如下图所示：



![img](../images/$%7Bfiilename%7D/format,png-1730491750663-3.png)

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑

参考：

[IO多路转接（复用）之pollsubingwen.cn/linux/poll/#1-poll%E5%87%BD%E6%95%B0编辑![img](../images/$%7Bfiilename%7D/icon-default-1730491750666-5.png)https://link.zhihu.com/?target=https%3A//subingwen.cn/linux/poll/%231-poll%25E5%2587%25BD%25E6%2595%25B0](https://link.zhihu.com/?target=https%3A//subingwen.cn/linux/poll/%231-poll%E5%87%BD%E6%95%B0)

## 6. epoll 模型（linux）

epoll是对select和poll的改进，改善了二者“性能开销大”和select“文件描述符数量受限”的两个缺点。相较于这两个前辈，epoll**改进了工作方式**，因此它更加高效。

- 对于待检测集合select和poll是基于**线性方式**（线性表）处理的，epoll是基于**红黑树**来管理待检测集合的

- select和poll每次都会**线性扫描**整个待检测集合，集合越大速度越慢，epoll使用的是**回调机制**，效率高，处理效率也不会随着检测集合的变大而下降

- 对select和poll返回的集合需要进行判断才能知道哪些文件描述符是就绪的，但epoll可以直接得到已就绪的文件描述符集合，**无需再次检测**

- 使用epoll没有最大文件描述符的限制，仅受系统中进程能打开的最大文件数目限制

- select和poll工作过程中存在内核/用户空间数据的

  频繁拷贝问题

  ，但在epoll中拷贝次数明显下降，主要因为： 

  - **事件驱动模型**：epoll采用事件驱动模型，允许用户在注册时指定感兴趣的事件，这样在事件发生时，内核只需**更新内部数据结构，而不需要每次都遍历所有文件描述符**。
  - **内部数据结构**：epoll使用红黑树和链表来管理事件，允许快速插入和删除操作。这意味着在修改和查询事件时，内核只需处理变化的部分，而不是每次都需要将所有文件描述符的信息从用户空间复制到内核空间。
  - **一次性获取多个事件**：epoll_wait函数允许一次性获取多个就绪事件。用户可以在一个调用中获得所有就绪的事件，减少了多次调用造成的拷贝次数。相比之下，select和poll在每次调用时通常需要处理大量文件描述符的状态。
  - **优化的内存管理**：epoll通过减少对用户空间数据的依赖，允许内核保持对监控的文件描述符状态的更有效管理，从而降低了在高并发环境下的拷贝开销

epoll的拷贝过程：使用copy_from_user函数从用户空间将epoll_event结构copy到内核空间，有事件后再通过__put_user来把发生事件和用户传入的数据copy出去

当多路复用文件数量很多、IO流量频繁的时候，一般使用epoll模型而不使用select、poll。epoll是 2.6内核中提出，使用 **epoll_event** 结构体来记录待监听的fd及其监听的事件类型的。

```cpp
typedef union epoll_data {
 	void        *ptr;
	int          fd;	// 记录文件描述符
	uint32_t     u32;
	uint64_t     u64;
} epoll_data_t;

struct epoll_event
{
    uint32_t events;  //epoll监听的事件类型
    epoll_data_t data; //应用程序数据
};
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### **6.1 epoll函数**

select、poll 模型都只使用一个函数，而 epoll 模型使用三个函数：**epoll_create、epoll_ctl 和 epoll_wait。**

```cpp
#include <sys/epoll.h>
// 创建epoll实例，通过一棵红黑树管理待检测集合
int epoll_create(int size);
// 管理红黑树上的文件描述符(添加、修改、删除)
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// 检测epoll树中是否有就绪的文件描述符
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

select/poll低效的原因之一是***将“添加/维护待检测任务”和“阻塞进程/线程”两个步骤合二为一\***。每次调用select都需要这两步操作，然而大多数应用场景中，需要监视的socket个数相对固定，并不需要每次都修改。***epoll将这两个操作分开，先用epoll_ctl()维护等待队列，再调用epoll_wait()阻塞进程（解耦）\***。通过下图的对比显而易见，epoll的效率更高。



![img](../images/$%7Bfiilename%7D/format,png-1730491750663-4.png)

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑

图片来源：https://subingwen.cn/linux/epoll/#2-%E6%93%8D%E4%BD%9C%E5%87%BD%E6%95%B0

***6.1.1 epoll_create()\***

```
int epoll_create(int size);
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

- size：在Linux内核2.6.8版本以后，这个参数是被忽略的，只需要指定一个大于0的数值就可以了。

epoll_create 会创建一个 **epoll 实例**，同时返回一个**引用该实例的文件描述符**。返回的文件描述符仅仅指向对应的 epoll 实例，并不表示真实的磁盘文件节点。其他 API 如 epoll_ctl、epoll_wait 会使用这个文件描述符来操作相应的 epoll 实例（**对底层的红黑树的节点进行添加、删除操作**）。

当创建好 epoll 句柄后，它会占用一个 fd 值，在 linux 下查看 /proc/进程id/fd/，就能够看到这个 fd。所以在使用完 epoll 后，必须调用 **close(epfd)** 关闭对应的文件描述符，否则可能导致 fd 被耗尽。当**指向同一个 epoll 实例的所有文件描述符都被关闭后，操作系统会销毁这个 epoll 实例。**

epoll 实例内部存储：

- 监听列表：所有要监听的文件描述符，使用**红黑树**
- 就绪列表：所有就绪的文件描述符，使用**链表**

***6.1.2 epoll_ctl ()***

该函数用于管理红黑树实例上的节点，可以进行添加、删除、修改操作。epoll_wait方法返回的事件必然是通过 epoll_ctl添加到 epoll中的。

```cpp
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

typedef union epoll_data {
 	void        *ptr;
	int          fd;	// 记录事件属于哪一个文件描述符
	uint32_t     u32;
	uint64_t     u64;
} epoll_data_t;

struct epoll_event
{
    uint32_t events;  //epoll监听的事件类型
    epoll_data_t data; //应用程序数据
};
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

epoll_ctl 会监听文件描述符 fd 上发生的 event 事件。

- **epfd** ：即 epoll_create 返回的文件描述符，指向一个 epoll 实例

- **fd**： 表示要监听的目标文件描述符

- op 

  ：表示要对 fd 执行的操作，有以下几种： 

  - EPOLL_CTL_ADD：在文件描述符epfd所引用的epoll实例上注册目标文件描述符fd，并将事件与链接到fd的内部文件相关联。
  - EPOLL_CTL_MOD：修改与文件描述符fd关联的事件（event 是一结构体变量，这相当于变量 event 本身没变，但是更改了其内部字段的值）
  - EPOLL_CTL_DEL：删除指定 fd 的所有监听事件，这种情况下 event 参数没用，**event = None**；

- event 

  ：epoll事件，用来修饰第三个参数对应的文件描述符，指定检测这个文件描述符的什么事件 

  - events：委托epoll检测的事件 	
    - EPOLLIN：读事件, 接收数据, 检测读缓冲区，如果有数据该文件描述符就绪
    - EPOLLOUT：写事件, 发送数据, 检测写缓冲区，如果有容量、可写，该文件描述符就绪
    - EPOLLERR：异常事件
    - EPOLLET：将epoll设置为边沿触发模式，默认情况是水平触发

- **返回值** 0 或 -1，表示上述操作成功与否。

epoll_ctl 会将文件描述符 fd 添加到 epoll 实例的监听列表里，同时为 fd 设置一个回调函数，并监听事件 event。当 fd 上发生相应事件时，会调用回调函数，将 fd 添加到 epoll 实例的就绪队列上。

***6.1.3 epoll_wait()\***

这是 epoll 模型的主要函数，功能相当于 select，用于检测创建的epoll实例中有没有就绪的文件描述符。

```cpp
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

- **epfd** ：即 epoll_create 返回的文件描述符，指向一个 epoll 实例
- **events**： 是一个event结构体数组，保存就绪状态的文件描述符。events中保存的元素是event，通过查看event结构体的绑定事件和data中的fd，即可知道哪一个fd的某个事件被触发。
- **maxevents** ：指定 events 的容量大小
- **timeout** ：类似于 select 中的 timeout。如果没有文件描述符就绪，即就绪队列为空，则 epoll_wait 会阻塞 timeout 毫秒。如果 timeout 设为 -1，则 epoll_wait 会一直阻塞，直到有文件描述符就绪；如果 timeout 设为 0，则 epoll_wait 会立即返回
- **返回值**：表示 events 中存储的就绪描述符个数，最大不超过 maxevents。

### 6.2 示例

```cpp
#include <stdio.h>
#include <ctype.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/epoll.h>

// server
int main(int argc, const char* argv[])
{
    // 创建监听的套接字
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket error");
        exit(1);
    }

    // 绑定
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(9999);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);  // 本地多有的ＩＰ
    
    // 设置端口复用
    int opt = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 绑定端口
    int ret = bind(lfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    if(ret == -1)
    {
        perror("bind error");
        exit(1);
    }

    // 监听
    ret = listen(lfd, 64);
    if(ret == -1)
    {
        perror("listen error");
        exit(1);
    }

    // 现在只有监听的文件描述符
    // 所有的文件描述符对应读写缓冲区状态都是委托内核进行检测的epoll
    // 创建一个epoll模型
    int epfd = epoll_create(100);
    if(epfd == -1)
    {
        perror("epoll_create");
        exit(0);
    }

    // 往epoll实例中添加需要检测的节点, 现在只有监听的文件描述符
    struct epoll_event ev;
    ev.events = EPOLLIN;    // 检测lfd读读缓冲区是否有数据
    ev.data.fd = lfd;
    // 将用于监听的套接字添加到epoll实例中
    ret = epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);
    if(ret == -1)
    {
        perror("epoll_ctl");
        exit(0);
    }

    struct epoll_event evs[1024];
    # 判断evs最多能容纳多少个epoll_event类型的结构体，不一定能容纳1024个
    int size = sizeof(evs) / sizeof(struct epoll_event);
    // 持续检测
    while(1)
    {
        // 检测添加到epoll实例中的文件描述符是否已就绪，并将这些已就绪的文件描述符进行处理
        int num = epoll_wait(epfd, evs, size, -1);
        for(int i=0; i<num; ++i)
        {
            // 取出当前的文件描述符
            int curfd = evs[i].data.fd;
            // 判断这个文件描述符是不是用于监听的
            if(curfd == lfd)
            {
                // 建立新的连接
                int cfd = accept(curfd, NULL, NULL);
                // 新得到的文件描述符添加到epoll模型中, 下一轮循环的时候就可以被检测了
                ev.events = EPOLLIN;    // 读缓冲区是否有数据
                ev.data.fd = cfd;
                ret = epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);
                if(ret == -1)
                {
                    perror("epoll_ctl-accept");
                    exit(0);
                }
            }
            else
            {
                // 处理通信的文件描述符
                // 接收数据
                char buf[1024];
                memset(buf, 0, sizeof(buf));
                int len = recv(curfd, buf, sizeof(buf), 0);
                if(len == 0)
                {
                    printf("客户端已经断开了连接\n");
                    // 将这个文件描述符从epoll模型中删除
                    epoll_ctl(epfd, EPOLL_CTL_DEL, curfd, NULL);
                    close(curfd);
                }
                else if(len > 0)
                {
                    printf("客户端say: %s\n", buf);
                    send(curfd, buf, len, 0);
                }
                else
                {
                    perror("recv");
                    exit(0);
                } 
            }
        }
    }

    return 0;
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

当在服务器端循环调用epoll_wait()的时候，就会得到一个就绪列表，并通过该函数的第二个参数传出：

```cpp
struct epoll_event evs[1024];
# 判断evs最多能容纳多少个epoll_event类型的结构体，不一定能容纳1024个
int size = sizeof(evs) / sizeof(struct epoll_event); 
# 就绪的元素个数
int num = epoll_wait(epfd, evs, size, -1);
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

每当epoll_wait()函数返回一次，在evs中最多可以存储size个已就绪的文件描述符信息，但是在这个数组中实际存储的有效元素个数为num个，如果在这个epoll实例的红黑树中已就绪的文件描述符很多，并且evs数组无法将这些信息全部传出，那么这些信息会在下一次epoll_wait()函数返回的时候被传出。

通过evs数组被传递出的每一个有效元素里边都包含了已就绪的文件描述符的相关信息（哪个文件符被触发、该文件描述符绑定了什么事件），这些信息并不是凭空得来的，这取决于我们在往epoll实例中添加节点的时候，往节点中初始化了哪些数据：

```cpp
struct epoll_event ev;
// 节点初始化
ev.events = EPOLLIN;    
ev.data.fd = lfd;	// 使用了联合体中 fd 成员
// 添加待检测节点到epoll实例中
int ret = epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

**epoll的工作流程：**

![img](../images/$%7Bfiilename%7D/94b8599c413548bdb654ca21a4506b9f.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑

### 6.3 epoll的工作模式

select 只支持水平触发，epoll 支持水平触发和边缘触发。

- **水平触发**（LT，Level Trigger）：当文件描述符就绪时，会触发通知，如果用户程序没有一次性把数据读/写完或者没有进行操作，下次还会发出可读/可写信号进行通知。
- **边缘触发**（ET，Edge Trigger）：仅当描述符从未就绪变为就绪时，通知一次，之后不会再通知。如果我们对这个文件描述符做IO操作，从而导致它再次变成未就绪，当这个未就绪的文件描述符再次变成就绪状态，内核会再次进行通知，并且还是只通知一次。

**区别**：当一个新的事件到来时，ET模式下可以从 epoll_wait调用中获取到这个事件，可是如果这次没有把这个事件对应的套接字缓冲区处理完，在这个套接字没有新的事件再次到来时，在 ET模式下是无法再次从 epoll_wait调用中获取这个事件的；而 LT模式则相反，***只要一个事件对应的套接字缓冲区还有数据\***，就总能从 epoll_wait中获取这个事件。因此，在 LT模式下开发基于 epoll的应用要简单一些，不太容易出错，而在 ET模式下事件发生时，如果没有彻底地将缓冲区数据处理完，则会导致缓冲区中的用户请求得不到响应。

水平触发、边缘触发的名称来源：数字电路当中的电位水平，高低电平切换瞬间的触发动作叫边缘触发，而处于高电平的触发动作叫做水平触发。

***6.3.1 水平触发\***

- 读事件

  ：

  *只要文件描述符对应的读缓冲区还有数据，读事件就会被触发，epoll_wait()解除阻塞*

  - 当读事件被触发，epoll_wait()解除阻塞，之后就可以接收数据了
  - 如果接收数据的buf很小，不能全部将缓冲区数据读出，那么读事件会继续被触发，直到数据被全部读出，如果接收数据的内存相对较大，读数据的效率也会相对较高（减少了读数据的次数）
  - 因为读数据是被动的，必须要通过读事件才能知道有数据到达了，因此对于读事件的检测是必须的

- 写事件

  ：

  *只要文件描述符对应的写缓冲区可写，写事件就会被触发，epoll_wait()解除阻塞*

  - 当写事件被触发，epoll_wait()解除阻塞，之后就可以将数据写入到写缓冲区了
  - 写事件的触发发生在写数据之前而不是之后，被写入到写缓冲区中的数据是由内核自动发送出去的
  - 如果写缓冲区没有被写满，写事件会一直被触发
  - 因为写数据是主动的，并且写缓冲区一般情况下都是可写的（缓冲区不满），因此对于写事件的检测不是必须的

水平触发代码可参考 **6.2 示例**

***6.3.2 边沿触发\***

- 读事件

  ：

  *当读缓冲区有新的数据进入，读事件被触发一次，没有新数据不会触发该事件*

  - 如果有新数据进入到读缓冲区，读事件被触发，epoll_wait()解除阻塞
  - 读事件被触发，可以通过调用read()/recv()函数将缓冲区数据读出 	
    - 如果数据没有被全部读走，并且没有新数据进入，读事件不会再次触发，只通知一次
    - 如果数据被全部读走或者只读走一部分，此时有新数据进入，读事件被触发，并且只通知一次

- 写事件

  ：

  *当写缓冲区状态可写，写事件只会触发一次*

  - 如果写缓冲区被检测到可写，写事件被触发，epoll_wait()解除阻塞
  - 写事件被触发，就可以通过调用write()/send()函数，将数据写入到写缓冲区中 	
    - 写缓冲区从不满到被写满，期间写事件只会被触发一次
    - 写缓冲区从满到不满，状态变为可写，写事件只会被触发一次

总结：epoll的边沿模式下 epoll_wait()检测到文件描述符有新事件才会通知，如果不是新的事件就不通知，通知的次数比水平模式少，效率比水平模式要高。

*a. 边沿触发如何设置？*

epoll默认是水平触发模式，只需在epoll_ctl ()函数中，将EPOLLET添加到结构体的events成员中即可：

```cpp
struct epoll_event ev;
ev.events = EPOLLIN | EPOLLET;	// 设置边沿模式
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

示例代码：

```cpp
int num = epoll_wait(epfd, evs, size, -1);
for(int i=0; i<num; ++i)
{
    // 取出当前的文件描述符
    int curfd = evs[i].data.fd;
    // 判断这个文件描述符是不是用于监听的
    if(curfd == lfd)
    {
        // 建立新的连接
        int cfd = accept(curfd, NULL, NULL);
        // 新得到的文件描述符添加到epoll模型中, 下一轮循环的时候就可以被检测了
        // 读缓冲区是否有数据, 并且将文件描述符设置为边沿模式
        struct epoll_event ev;
        ev.events = EPOLLIN | EPOLLET;   
        ev.data.fd = cfd;
        ret = epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);
        if(ret == -1)
        {
            perror("epoll_ctl-accept");
            exit(0);
        }
    }
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

主要就是在建立新连接时，将新连接的socket设置为边沿模式。

*b. **边沿模式**必须设置为非阻塞IO*

对于写事件的触发一般情况下是不需要进行检测的，因为写缓冲区大部分情况下都是有足够的空间可以进行数据的写入。对于读事件的触发就必须要检测，因为服务器不知道客户端什么时候发送数据，如果使用epoll的**边沿模式**进行读事件的检测，有新数据达到只会通知一次，那么必须要保证得到通知后将数据全部从读缓冲区中读出。那么，应该如何读这些数据呢？

**方式1**：准备一块特别大的内存，用于存储从读缓冲区中读出的数据，但是这种方式有很大的弊端：

- 内存的大小没有办法界定，太大浪费内存，太小又不够用
- 系统能够分配的最大堆内存也是有上限的，栈内存就更不必多言了

**方式2**：循环接收数据：

```cpp
int len = 0;
while((len = recv(curfd, buf, sizeof(buf), 0)) > 0)
{
    // 数据处理...
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

这样做也是有弊端的，因为***套接字操作默认是阻塞\***的，当读缓冲区数据被读完之后，读操作就阻塞了也就是调用的read()/recv()函数被阻塞了，当前进程/线程被阻塞之后就无法处理其他操作了。

要解决阻塞问题，就需要将套接字默认的阻塞行为修改为非阻塞，需要使用fcntl()函数进行处理：

```cpp
// 设置完成之后, 读写都变成了非阻塞模式
int flag = fcntl(cfd, F_GETFL);
flag |= O_NONBLOCK;     // flag = flag | O_NONBLOCK;                                                  
fcntl(cfd, F_SETFL, flag);
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

fcntl() 是一个变参函数, 并且是多功能函数，可以通过这个函数实现文件描述符的复制和获取/设置已打开的文件属性。该函数的函数原型如下：

```cpp
#include <unistd.h>
#include <fcntl.h>	// 主要的头文件

int fcntl(int fd, int cmd, ... /* arg */ );
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

- fd: 要操作的文件描述符
- cmd: 通过该参数控制函数要实现什么功能

| 参数 cmd 的取值 | 功能描述                     |
| --------------- | ---------------------------- |
| F_DUPFD         | 复制一个已经存在的文件描述符 |
| F_GETFL         | 获取文件的状态标志           |
| F_SETFL         | 设置文件的状态标志           |

- 返回值：函数调用失败返回 -1，调用成功，返回正确的值： 
  - 参数 cmd = F_DUPFD：返回新的被分配的文件描述符
  - 参数 cmd = F_GETFL：返回文件的flag属性信息

| 文件状态标志 | 说明                     |
| ------------ | ------------------------ |
| O_RDONLY     | 只读打开                 |
| O_WRONLY     | 只写打开                 |
| O_RDWR       | 读、写打开               |
| O_APPEND     | 追加写                   |
| O_NONBLOCK   | 非阻塞模式               |
| O_SYNC       | 等待写完成（数据和属性） |
| O_ASYNC      | 异步I/O                  |
| O_RSYNC      | 同步读和写               |

### 6.4 使用边沿非阻塞IO模式下的示例代码

```cpp
#include <stdio.h>
#include <ctype.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <fcntl.h>
#include <errno.h>

// server
int main(int argc, const char* argv[])
{
    // 创建监听的套接字
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket error");
        exit(1);
    }

    // 绑定
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(9999);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);  // 本地多有的ＩＰ
    // 127.0.0.1
    // inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr.s_addr);
    
    // 设置端口复用
    int opt = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 绑定端口
    int ret = bind(lfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    if(ret == -1)
    {
        perror("bind error");
        exit(1);
    }

    // 监听
    ret = listen(lfd, 64);
    if(ret == -1)
    {
        perror("listen error");
        exit(1);
    }

    // 现在只有监听的文件描述符
    // 所有的文件描述符对应读写缓冲区状态都是委托内核进行检测的epoll
    // 创建一个epoll模型
    int epfd = epoll_create(100);
    if(epfd == -1)
    {
        perror("epoll_create");
        exit(0);
    }

    // 往epoll实例中添加需要检测的节点, 现在只有监听的文件描述符
    struct epoll_event ev;
    ev.events = EPOLLIN;    // 检测lfd读读缓冲区是否有数据
    ev.data.fd = lfd;
    ret = epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);
    if(ret == -1)
    {
        perror("epoll_ctl");
        exit(0);
    }


    struct epoll_event evs[1024];
    int size = sizeof(evs) / sizeof(struct epoll_event);
    // 持续检测
    while(1)
    {
        // 调用一次, 检测一次
        int num = epoll_wait(epfd, evs, size, -1);
        printf("==== num: %d\n", num);

        for(int i=0; i<num; ++i)
        {
            // 取出当前的文件描述符
            int curfd = evs[i].data.fd;
            // 判断这个文件描述符是不是用于监听的
            if(curfd == lfd)
            {
                // 建立新的连接
                int cfd = accept(curfd, NULL, NULL);
                // 将文件描述符设置为非阻塞
                // 得到文件描述符的属性
                int flag = fcntl(cfd, F_GETFL);
                flag |= O_NONBLOCK;
                fcntl(cfd, F_SETFL, flag);
                // 新得到的文件描述符添加到epoll模型中, 下一轮循环的时候就可以被检测了
                // 通信的文件描述符检测读缓冲区数据的时候设置为边沿模式
                ev.events = EPOLLIN | EPOLLET;    // 读缓冲区是否有数据
                ev.data.fd = cfd;
                ret = epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);
                if(ret == -1)
                {
                    perror("epoll_ctl-accept");
                    exit(0);
                }
            }
            else
            {
                // 处理通信的文件描述符
                // 接收数据
                char buf[5];
                memset(buf, 0, sizeof(buf));
                // 循环读数据
                while(1)
                {
                    int len = recv(curfd, buf, sizeof(buf), 0);
                    if(len == 0)
                    {
                        // 非阻塞模式下和阻塞模式是一样的 => 判断对方是否断开连接
                        printf("客户端断开了连接...\n");
                        // 将这个文件描述符从epoll模型中删除
                        epoll_ctl(epfd, EPOLL_CTL_DEL, curfd, NULL);
                        close(curfd);
                        break;
                    }
                    else if(len > 0)
                    {
                        // 通信
                        // 接收的数据打印到终端
                        write(STDOUT_FILENO, buf, len);
                        // 发送数据
                        send(curfd, buf, len, 0);
                    }
                    else
                    {
                        // len == -1
                        if(errno == EAGAIN)
                        {
                            printf("数据读完了...\n");
                            break;
                        }
                        else
                        {
                            perror("recv");
                            exit(0);
                        }
                    }
                }
            }
        }
    }

    return 0;
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 6.5 关于边缘模式的问题

***6.5.1 为什么边缘触发必须使用非阻塞 I/O？\***

这部分内容可以回顾一下第一章和第二章关于阻塞IO、非阻塞IO、异步IO、多路复用的知识，并将其串联起来。详细内容可以参考这篇文章，讲的非常详细：

[Blocking I/O, Nonblocking I/O, And Epolleklitzke.org/blocking-io-nonblocking-io-and-epoll![img](../images/$%7Bfiilename%7D/icon-default-1730491750666-5.png)https://link.zhihu.com/?target=https%3A//eklitzke.org/blocking-io-nonblocking-io-and-epoll](https://link.zhihu.com/?target=https%3A//eklitzke.org/blocking-io-nonblocking-io-and-epoll)





> **缓冲区大小限制**：每次通过read系统调用时，最多只能读取与缓冲区大小相等的字节数。如果收到的数据超过了这个大小，就需要多次调用read才能将所有数据读取完。

- 如果使用阻塞模式，假设缓冲区大小为1024字节，但一次性收到 64KB 数据。为处理此请求，我们将调用 select，然后调用 read 64 次，总共进行 128 次系统调用（64次select和64次read），这非常多。
- 如果使用非阻塞模式，我们将进行 66 次系统调用：select 只需调用一次，read 循环调用 64 次，然后再调用一次 read 返回 EWOULDBLOCK。相比阻塞模式，非阻塞模式的系统调用几乎少了一半。
- 非阻塞方法的缺点是，由于新增的循环，至少会多进行一次系统调用，因为 read 会被调用，直到返回 EWOULDBLOCK。假设通常情况下，读取缓冲区足够大，能够在一次调用中读取所有传入的数据。那么在常见情况下，通过这个循环将会有三个系统调用，而不是两个：一个用于等待数据，一个用于实际读取数据，另一个用于处理 EWOULDBLOCK。

**select的工作方式：**

- **阻塞I/O**：select可以使用阻塞I/O。当select检测到某个文件描述符可读后，遍历所有可读的文件描述符并进行read操作。由于文件描述符是可读的，即使使用阻塞I/O，也一定能读取到数据，不会一直阻塞。
- **水平触发模式**：在这种模式下，如果第一次read没有读取完所有数据，下一次调用select时，仍然会返回这个文件描述符。这意味着你可以再次尝试读取数据。
- **非阻塞I/O**：如果使用非阻塞I/O，可以在遍历可读文件描述符时，使用循环不断调用read，直到返回EWOULDBLOCK（表示没有更多数据可读）。这样做虽然增加了一次read调用，但可以减少后续调用select的次数。

**epoll的边缘触发模式：**

- **通知机制**：在边缘触发模式下，只有当文件描述符的状态（可读或可写）发生变化时，才会收到操作系统的通知。这意味着如果在第一次通知时没有一次性读取完所有数据，文件描述符的状态在操作系统看来是没有变化的，后续将不会再发出通知。
- **使用非阻塞I/O**：因此，如果使用epoll的边缘触发模式，必须使用非阻塞I/O，并**循环调用**read或write，直到返回EWOULDBLOCK为止。这可以确保你已经处理完所有数据，然后再调用epoll_wait等待下一个通知。

这种方法的好处是，每次调用epoll_wait都能确保数据被全部读取或写入，这样就不会因为没有数据而浪费调用次数。在水平触发模式下，如果epoll_wait时数据未处理完，会直接返回，导致重复通知。

***6.5.2 为什么 epoll 的边缘触发模式不能使用阻塞 I/O？\***

边缘触发模式需要循环读/写一个文件描述符的所有数据。如果使用阻塞 I/O，那么一定会在***最后一次调用（没有数据可读/写）时阻塞，导致无法正常结束。\***

### 6.6 epoll的优点

> **select只支持水平触发，epoll支持边缘触发，epoll两种方式均支持。**

之前说，epoll 是对 select 和 poll 的改进，避免了“性能开销大”和“文件描述符数量少”两个缺点。

- 对于“文件描述符数量少”，select 使用整型数组存储文件描述符集合，而 epoll 使用红黑树存储，数量较大。
- 对于“性能开销大”，epoll_ctl 中为每个文件描述符指定了回调函数，并在就绪时将其加入到**就绪列表**，因此 epoll 不需要像 select 那样遍历检测每个文件描述符，只需要判断就绪列表是否为空即可。这样，在没有描述符就绪时，epoll 能更早地让出系统资源。相当于时间复杂度从 O(n) 降为 O(1)
- 此外，每次调用 select 时都需要向内核拷贝所有要监听的描述符集合，而 epoll 对于每个描述符，只需要在 epoll_ctl 传递一次，之后 epoll_wait 不需要再次传递。这也大大提高了效率。

使用了红黑树，在下面场景下效率会很高：

- 查找：想注册一个文件描述符的时候要判断这个文件描述符是不是已经存在，所以要查
- 删除：断开连接，不再关注这个文件描述符的时候需要快速找到这个描述符，删掉他

红黑树能够满足在这两个场景下都很快

参考：

[IO多路转接（复用）之epollsubingwen.cn/linux/epoll/#2-%E6%93%8D%E4%BD%9C%E5%87%BD%E6%95%B0编辑![img](../images/$%7Bfiilename%7D/icon-default-1730491750666-5.png)https://link.zhihu.com/?target=https%3A//subingwen.cn/linux/epoll/%232-%25E6%2593%258D%25E4%25BD%259C%25E5%2587%25BD%25E6%2595%25B0](https://link.zhihu.com/?target=https%3A//subingwen.cn/linux/epoll/%232-%E6%93%8D%E4%BD%9C%E5%87%BD%E6%95%B0)

------

|            | select             | poll             | epoll                                             |
| ---------- | ------------------ | ---------------- | ------------------------------------------------- |
| 数据结构   | bitmap             | 数组             | 红黑树                                            |
| 最大连接数 | 1024               | 无上限           | 无上限                                            |
| fd拷贝     | 每次调用select拷贝 | 每次调用poll拷贝 | fd首次调用epoll_ctl拷贝，每次调用epoll_wait不拷贝 |
| 工作效率   | 轮询：O(n)         | 轮询：O(n)       | 回调：O(1)                                        |
