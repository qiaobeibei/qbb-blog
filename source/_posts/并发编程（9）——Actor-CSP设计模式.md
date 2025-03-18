---
title: 并发编程（9）——Actor/CSP设计模式
date: 2024-11-11 11:29:19
categories:
- C++
- 并发编程
tags: 
- 设计模式
typora-root-url: ./.. 
---

# 九、day9

在并发编程中，多个线程可能需要同时访问相同的内存资源。为了防止不同线程之间的资源冲突，传统并发设计方法通常使用共享内存和加锁机制来确保线程安全。例如，当一个线程在修改共享数据时，其他线程会被“锁住”，无法同时访问该数据。但是传统并发设计方法在频繁加锁的情况下会带来性能开销，**降低系统的执行效率**；并且共享内存加锁方式要求线程之间对共享数据有很强的依赖关系，这种**依赖增加了代码的复杂性和耦合度**，使代码难以维护。

**新的设计模式**：

- **Actor模式**：Actor模式通过消息传递的方式来实现线程间通信。每个Actor都有自己的状态和行为，它们通过发送消息来完成交互，而不需要共享内存。这种方式避免了加锁的复杂性和性能损耗。
- **CSP（Communicating Sequential Processes）模式**：CSP模式也是通过消息传递进行通信，但它强调线程（或进程）之间的严格隔离。各个线程通过通道（Channel）来传递消息，而不直接共享状态，避免了竞争条件和加锁问题。

参考：

[博主恋恋风辰的个人博客](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/2WNBz17Q9PNiHXiLygjre8TdhlL)

------

# 1. Actor设计模式

Actor模型的设计模式有以下几个核心要素：

1. **独立的Actor**：每个Actor是独立的个体，拥有自己的状态和行为。Actor之间不共享状态，从而消除了并发编程中的数据共享问题。
2. **异步消息传递**：Actor之间不直接调用方法，而是通过消息传递来通信。消息传递是**异步非阻塞**的，发送方不需要等待接收方完成处理，而是立即继续执行自己的任务，避免了阻塞等待带来的性能问题。
3. **顺序消息处理**：每个Actor都有一个“邮箱”或“消息队列”，它接收来自其他Actor的消息，并按照顺序缓存起来。Actor处理消息时会从邮箱中取出一条消息并执行相应的操作，并且**每个Actor一次只同步能处理一个消息**（处理过程中，除了可以接受消息外，不能左任何其他无关操作），保证了消息处理的原子性。Actor在处理消息时不会被其他消息打断，也不会与其他Actor竞争资源，从而减少了并发问题。
4. **无共享状态**：由于Actor之间不共享数据，只能通过消息传递来交互，因此大大降低了并发编程中的数据竞争和死锁等复杂性。
5. **单线程**：每个Actor在独立的线程中运行，Actor之间通过消息队列来通信。例如，Actor1向Actor2发送消息时，消息会投递至Actor2的队列中，Actor2从队列中取出消息并进行处理。这种设计就像邮件通信一样，一个Actor向另一个Actor“投递”一条消息，接收的Actor从“邮箱”中取出消息进行处理。

如下图所示：

![1731313547270](/../../images/$%7Bfiilename%7D/1731313547270.jpg)

<center>Actor2邮箱处理过程</center>

因为Actor之间不共享数据，只能通过消息传递来交互，因此大大降低了并发编程中的数据竞争和死锁等复杂性。我们需要维护的只有每个Actor接受消息的消息队列，只有**保证是线程安全的消息队列即可**。

之前不管是在网络编程逻辑层消息队列（多个服务线程向逻辑线程消息队列投递消息）设计中，还是在并发编程学习关于条件变量相关知识中，都实现了线程安全的消息队列，可参考：

1. [网络编程（14）——基于单例模板实现的逻辑层 | 爱吃土豆的个人博客](https://www.aichitudou.cn/2024/11/03/网络编程（14）——基于单例模板实现的逻辑层/)
2. [并发编程（5）——条件变量、线程安全队列 | 爱吃土豆的个人博客](https://www.aichitudou.cn/2024/11/03/并发编程（5）——条件变量、线程安全的队列/)

# 2. CSP设计模式

**CSP**（Communicating Sequential Processes，**通信顺序进程**）由英国计算机科学家 Tony Hoare 在1978年提出，用于描述两个独立的并发实体通过共享的通讯 `channel`(管道)进行通信的并发模型。

**CSP（通信顺序进程）**模式将`channel`视为一等公民（第一类对象），其主要关注点在于通信的通道，而非发送或接收消息的实体。与Actor模型关注"谁在发送或接收消息"不同，CSP更侧重于"通过什么渠道传递消息"，即强调通信本身而非通信双方的具体实现。

在CSP中，**channel**被用作不同进程之间的通信媒介。进程通过channel发送和接收消息来进行同步和数据传递，channel承担了中介的角色。与Actor模型中由每个Actor维护自己的邮箱不同，CSP模型允许进程通过共享的channel进行直接通信（**CSP将消息投递给channel，至于谁从channel中取数据，谁从channel中发数据，发送的一方和接收的一方是不关注的**）。CSP模型不关心消息从哪来或到哪去，只关心消息是通过哪个channel传递的。这种设计实现了进程的解耦，并且允许进程通过**共享channel**来实现安全的**同步通信**。

> 简单来说，Actor在发送消息前必须知道接收方是谁，而接受方收到消息后也需知道发送方是谁，更像是邮件的通信模式。而csp是完全解耦合的，不关心消息从哪来或到哪去，只关心消息是通过哪个channel传递的。

## 2.1 使用C++实现CSP

```c++
#include <iostream>
#include <queue>
#include <mutex>
#include <condition_variable>

template <typename T>
class Channel {
private:
    std::queue<T> queue_;
    std::mutex mtx_;
    std::condition_variable cv_producer_;
    std::condition_variable cv_consumer_;
    size_t capacity_;
    bool closed_;
public:
    Channel(size_t capacity = 0) : capacity_(capacity), closed_(false) {}
    bool send(T value) {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_producer_.wait(lock, [this]() {
            // 对于无缓冲的channel，我们应该等待直到有消费者准备好
            return (capacity_ == 0 && queue_.empty()) || queue_.size() < capacity_ || closed_;
            });
        if (closed_) {
            return false;
        }
        queue_.push(value);
        cv_consumer_.notify_one();
        return true;
    }
    bool receive(T& value) {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_consumer_.wait(lock, [this]() { return !queue_.empty() || closed_; });
        if (closed_ && queue_.empty()) {
            return false;
        }
        value = queue_.front();
        queue_.pop();
        cv_producer_.notify_one();
        return true;
    }
    void close() {
        std::unique_lock<std::mutex> lock(mtx_);
        closed_ = true;
        cv_producer_.notify_all();
        cv_consumer_.notify_all();
    }
};
```

1. 类的成员变量

   - `std::queue<T> queue_`: 存储发送到 channel 的消息的队列。
   - `std::mutex mtx_`: 互斥锁，用于保护共享资源（消息队列）避免多线程的竞争。
   - `std::condition_variable cv_producer_`: 用于控制生产者（`send`操作）线程的条件变量。
   - `std::condition_variable cv_consumer_`: 用于控制消费者（`receive`操作）线程的条件变量。
   - `size_t capacity_`: 表示 channel 的容量。如果 `capacity_` 为 0，则表示这是一个无缓冲的 channel，如果没有缓冲，那就相当于同步应用，生产者放入数据后生产者马上就会取。
   - `bool closed_`: 表示 channel 是否已关闭。`true` 表示 channel 已关闭，无法再发送消息。

2. `send` 方法：向 channel 发送消息

   ```c++
   bool send(T value) {
       std::unique_lock<std::mutex> lock(mtx_);
       cv_producer_.wait(lock, [this]() {
           return (capacity_ == 0 && queue_.empty()) || queue_.size() < capacity_ || closed_;
       });
       if (closed_) {
           return false;
       }
       queue_.push(value);
       cv_consumer_.notify_one();
       return true;
   }
   ```

   主要步骤是通过条件变量挂起当前线程，若当前线程被唤醒，并且满足判断条件，则继续执行下面的代码，过程如下

   使用 `cv_producer_.wait` 等待以下条件之一：

   - **无缓冲 channel 且队列为空**：只有当无缓冲 channel 且没有未消费的消息时，才能继续执行 `send`的代码。
   - **缓冲 channel 且队列未满**：缓冲 channel 时，只有队列中消息数量小于 `capacity_` 时才能继续 `send`。
   - **channel 已关闭**：如果 `closed_` 为 `true`，不再等待，直接返回。

   将数据`push`至消息队列，并唤醒消费者进行消费

3. `receive` 方法：从 channel 接收消息

   ```c++
   bool receive(T& value) {
       std::unique_lock<std::mutex> lock(mtx_);
       cv_consumer_.wait(lock, [this]() { return !queue_.empty() || closed_; });
       if (closed_ && queue_.empty()) {
           return false;
       }
       value = queue_.front();
       queue_.pop();
       cv_producer_.notify_one();
       return true;
   }
   ```

   若当前线程被唤醒，并且满足队列不为空或者closed_为true任意之一时，继续执行`receive`下面的代码；如果不满足，线程继续挂起，等待被再次唤醒；

   消息从队列弹出后，唤醒生产者线程生产；

   注意，这里从消息队列中`receice`相当于之前通过条件变量实现的线程安全的消息队列中的`pop`函数，这个对引用进行赋值，必须先将消息队列中的值赋值给引用对象，然后才能从消息队列中弹出。这是为了防止内存爆炸造成的数据丢失，具体分析可以参考我之前写的文章：[条件变量实现线程安全的消息队列](https://www.aichitudou.cn/2024/11/03/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%EF%BC%885%EF%BC%89%E2%80%94%E2%80%94%E5%88%A9%E7%94%A8%E6%9D%A1%E4%BB%B6%E5%8F%98%E9%87%8F%E5%AE%9E%E7%8E%B0%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E9%98%9F%E5%88%97/#2-%E4%BD%BF%E7%94%A8%E6%9D%A1%E4%BB%B6%E5%8F%98%E9%87%8F%E5%AE%9E%E7%8E%B0%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E7%9A%84%E9%98%9F%E5%88%97)

4. `close` 方法：用于关闭 channel

   ```c++
   void close() {
       std::unique_lock<std::mutex> lock(mtx_);
       closed_ = true;
       cv_producer_.notify_all();
       cv_consumer_.notify_all();
   }
   ```

   将判断变量`closed_`置为`true`，表示直接退出，然后使用`notify_all()`唤醒全部工作线程。

5. 测试

   ```c++
   int main() {
       Channel<int> ch(10);  // 10缓冲的channel
       std::thread producer([&]() {
           for (int i = 0; i < 5; ++i) {
               ch.send(i);
               std::cout << "Sent: " << i << std::endl;
           }
           ch.close();
           });
       
       std::thread consumer([&]() {
           std::this_thread::sleep_for(std::chrono::milliseconds(500)); // 故意延迟消费者开始消费
           int val;
           while (ch.receive(val)) {
               std::cout << "Received: " << val << std::endl;
           }
           });
       
       producer.join();
       consumer.join();
       return 0;
   }
   ```

   创建一个有 10 个缓冲的 `Channel<int>` 类型的对象 `ch`：

   - 最多可以存储 10 个未被消费的数据。
   - 生产者可以最多连续发送 10 条数据，而无需等待消费者立即接收（消费者可以等一会儿再继续消费）。
   - 缓冲区满后，生产者会暂停（若缓冲区满后，除了将`closed_`置为`true`外，不满足任何判断条件可以退出线程等待），直到消费者消费一些数据，从而腾出空间。

   创建并启动生产者线程：

   - 在循环中，将值 `i` 发送到 `ch`，并在控制台输出 "Sent: `i`"。
   - 生产者线程发送 5 个值（0 到 4），每发送一个值后会打印一次。
   - 发送完成后，调用 `ch.close()`，通知 channel 已关闭，表示生产者不再发送数据。

   创建并启动消费者线程：

   - 线程开始后延迟 500 毫秒，模拟消费者稍后开始消费的情况。
   - 进入循环，尝试从 `ch` 接收数据（因为在生产者从队列插入数据后，会唤醒消费者线程进行消费）。若 `receive` 成功接收到值，则输出 "Received: `val`"。
   - 当 channel 关闭并且所有消息都已被消费后，`receive` 将返回 `false`，使得循环结束。

   > 这里有一个问题需要注意：消费者线程是在生产者线程结束之后才运行的（消费者线程延迟500ms），那么就没有代码唤醒消费者线程，为什么消费者线程可以正常运行？

   在生产者线程调用 `ch.send(i)` 时，`send` 方法的最后一行是 `cv_consumer_.notify_one();`，它会在成功推送数据到 `queue_` 后通知等待的消费者线程。这一行为虽然发生在消费者进入等待状态（500 毫秒后）之前，但 `notify_one()` 的通知**会被记录下来**。

   这种行为在 `condition_variable` 的底层实现中是合理的：在消费者延迟执行后调用 `cv_consumer_.wait()` 时，条件变量会再次检查 `predicate`。如果 `queue_` 已有数据（即 `!queue_.empty()`），则判断条件 `predicate` 为 `true`，消费者线程就会立即获取消息并继续执行，而不会阻塞。

   这是因为当消费者线程进入 `wait` 时，它会**首先检查判断条件**是否满足，如果条件已经为 `true`，消费者不再等待，而是直接继续执行。**并不是唤醒线程后才会检查，而是调用wait函数后会立即检查判断条件，然后每次被唤醒再次检查。**

   > [测试代码](https://godbolt.org/)

   输出结果为：

   ```
   Sent: 0
   Sent: 1
   Sent: 2
   Sent: 3
   Sent: 4
   Received: 0
   Received: 1
   Received: 2
   Received: 3
   Received: 4
   ```

   如果将缓冲`capacity_`设为0的话，就相当于同步应用，生产者放入数据后生产者马上就会取。

   > [测试代码](https://godbolt.org/)

   输出结果为：

   ```
   Sent: 0
   Received: 0
   Sent: 1
   Received: 1
   Sent: 2
   Received: 2
   Sent: 3
   Received: 3
   Sent: 4
   Received: 4
   ```

## 2.2 通过CSP实现ATM取款逻辑

![1731329355357](/../../images/$%7Bfiilename%7D/1731329355357.jpg)

整体逻辑如上图所示。

代码可参考：[代码](https://gitcode.com/m0_63086198/cpp/overview)
