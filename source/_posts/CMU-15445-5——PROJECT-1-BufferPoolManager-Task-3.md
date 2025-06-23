---
title: CMU-15445(5)——PROJECT#1-BufferPoolManager-Task#3
date: 2025-05-18 10:38:23
categories:
- CMU-15445
tags: 
- Fall 2024
typora-root-url: ./..
---

# PROJECT#1-BufferPoolManager

## Task #3 - Buffer Pool Manager

本节实现 `PROJECT#1` 最后一个 `TASK`，主要内容是使用  `TASK1` 和 `TASK2` 实现的 `LRU-K Replacement Policy` 和 `DiskScheduler`（前者会跟踪页面的访问情况，以便在需要为新页面腾出空间时决定淘汰哪个帧；后者会通过 `DiskManager` 调度磁盘的读写操作），从磁盘获取数据页并将其存储到内存中，当缓冲池管理器被显式要求写出脏页，或者需要淘汰某个页以为新页腾出空间时，也会调度将脏页写回磁盘。整体架构如下图所示：

![image-20250518111117914](/images/$%7Bfiilename%7D/image-20250518111117914.png)

`BusTub` 通过名为 `FrameHeader` 的辅助类实现对内存中帧的管理，所有页面数据的访问均需通过该类进行 —— 其提供的 `GetData` 方法可返回指向帧内存的原始指针，供 `DiskScheduler` 与 `DiskManager` 借此将磁盘物理页内容复制到内存。`Buffer Pool Manager`无需解析页面具体内容，仅需维护页面 ID（page_id_t）及对应的 `FrameHeader` 即可；当页面在磁盘与内存间移动时，系统会重用同一 `FrameHeader` 对象存储不同页面，即在系统生命周期内，每个 `FrameHeader` 可承载多轮不同的页面数据存储任务。源码分析如下：

```cpp
class FrameHeader {
  friend class BufferPoolManager;
  friend class ReadPageGuard;
  friend class WritePageGuard;

 public:
  explicit FrameHeader(frame_id_t frame_id);

 private:
  auto GetData() const -> const char *;
  auto GetDataMut() -> char *;
  void Reset();
    
  const frame_id_t frame_id_;
  std::shared_mutex rwlatch_;
  std::atomic<size_t> pin_count_;
  bool is_dirty_;
  std::vector<char> data_;
};
```

`FrameHeader` 有五个私有成员变量：

- `frame_id_` 作为缓冲区中帧的索引；
- `pin_count_` 是引用计数，类似智能指针的 `ref_count`；
- `is_dirty_` 是脏页标记，表示页是否需要写回磁盘（内存数据页跟磁盘数据页内容不一致的时候，称这个内存页为“脏页”，内存数据写入到磁盘后，内存和磁盘上的数据页的内容就一致了）；
- `rwlatch_` 是读写锁，运行多个读线程并发访问，同时保证写操作的互斥性；
- `data_` 是页面数据存储，初始化大小为 `static constexpr int BUSTUB_PAGE_SIZE = 4096（4KB）`。

并提供了三个私有函数：

- `GetData()`：返回指向`vector`所管理数组**首元素**的指针，其返回的指针内容是不能修改的。
- `GetDataMut()`：同样返回指向`vector`所管理数组**首元素**的指针，只不过其返回的指针是可变的。
- `Reset()`：将 `data_` 清空为零值，重置`pin_count_`和`is_dirty_`

接下来实现 `BufferPoolManager` 和 `ReadPageGuard / WritePageGuard`，包含以下文件：

- `src/include/buffer/buffer_pool_manager.h`
- `src/buffer/buffer_pool_manager.cpp`
- `src/storage/page/page_guard.cpp`
- `src/include/storage/page/page_guard.h`

### buffer_pool_manager.h

首先实现 `BufferPoolManager`，简单看一下 `BusTub` 提供的初始代码：

```cpp
namespace bustub {

class BufferPoolManager;
class ReadPageGuard;
class WritePageGuard;

class FrameHeader {...}

class BufferPoolManager {
 public:
  BufferPoolManager(size_t num_frames, DiskManager *disk_manager, size_t k_dist = LRUK_REPLACER_K,
                    LogManager *log_manager = nullptr);
  ~BufferPoolManager();

  auto Size() const -> size_t;
  auto NewPage() -> page_id_t;
  auto DeletePage(page_id_t page_id) -> bool;
  auto CheckedWritePage(page_id_t page_id, AccessType access_type = AccessType::Unknown)
      -> std::optional<WritePageGuard>;
  auto CheckedReadPage(page_id_t page_id, AccessType access_type = AccessType::Unknown) -> std::optional<ReadPageGuard>;
  auto WritePage(page_id_t page_id, AccessType access_type = AccessType::Unknown) -> WritePageGuard;
  auto ReadPage(page_id_t page_id, AccessType access_type = AccessType::Unknown) -> ReadPageGuard;
  auto FlushPageUnsafe(page_id_t page_id) -> bool;
  auto FlushPage(page_id_t page_id) -> bool;
  void FlushAllPagesUnsafe();
  void FlushAllPages();
  auto GetPinCount(page_id_t page_id) -> std::optional<size_t>;

 private:
  const size_t num_frames_;
  std::atomic<page_id_t> next_page_id_;
  std::shared_ptr<std::mutex> bpm_latch_;
  std::vector<std::shared_ptr<FrameHeader>> frames_;
  std::unordered_map<page_id_t, frame_id_t> page_table_;
  std::list<frame_id_t> free_frames_;
  std::shared_ptr<LRUKReplacer> replacer_;
  std::shared_ptr<DiskScheduler> disk_scheduler_;
  LogManager *log_manager_ __attribute__((__unused__));
};
}  // namespace bustub
```

`FrameHeader` 类在上面分析过了，是一个管理内存帧及其相关的元数据的辅助类。

`BufferPoolManager` 类是本节的重点，负责管理内存中的数据页，包括从磁盘读取数据页、将数据页写入磁盘、缓存常用数据页以及淘汰不常用的数据页等功能。其主要私有成员变量如下：

- `num_frames_`：缓冲池中的帧数，可用于初始化 replacer_；
- `next_page_id_`：下一个要分配的页面 ID；
- `frames_`：存储帧头的`vector`容器，帧头通过智能指针进行管理；
- `page_table_`：页面 ID 到帧 ID 的映射表；
- `free_frames_`：存储空闲帧的 ID；
- `replacer_`：使用 TASK1 实现的 `LRUKReplacer`，选择要淘汰的页面；
- `disk_scheduler_`：使用 TASK2 实现的 `DiskScheduler`，用于磁盘调度；

我们主要负责实现公有函数，这里简单分析一下其功能，为后续实现做铺垫：

- `BufferPoolManager` 的四参数构造函数主要是为了初始化 `replacer_` 、 `disk_scheduler_` 以及 `page_table_`、`frames_`，`replacer_` 的 K 默认是 `LRUK_REPLACER_K（10）`；
- `Size()`：返回缓冲池中的帧数，很简单，直接返回 `num_frames_` 即可；
- `NewPage()`：分配一个新的页面，并返回其页面 ID，大概流程如下：
  1. 从 `free_frames_` 获取可用空闲帧；
  2. 如果 `free_frames_`  为空，则通过 `replacer_` 淘汰页面获取空闲帧，并判断该帧是否为脏页（如果不是脏页，说明内存的数据和磁盘数据相同，可以直接从页表中删除被淘汰的页面，否则得先通过 `disk_scheduler_` 构造写请求将淘汰页的数据写入磁盘更新数据，然后再从页表中删除被淘汰的页面），然后从页表中删除被淘汰的页面；
  3. 给新页分配 id ，并显式增加 `next_page_id_` 的值（使用原子变量的`fetch_add`方法增加）；
  4. 初始化帧并更新页表（绑定新页与帧）；
  5. 通过替换其添加新帧的历史记录时间，并置为非淘汰状态；
  6. 返回新页id
- `DeletePage(page_id_t page_id)`：删除指定页面 ID 的页面，主要流程如下：
  1. 检查页表中是否存在该页面
  2. 如果存在则获取页面所在的帧头，并判断该页是否被其他线程访问（`pin_count_ > 0`）
  3. 如果页面未被其他线程访问，则判断是否为脏页，是否需要写回磁盘
  4. 从页表中删除该页面的映射，并将该帧id添加至`free_frames_`
  5. 通过替换其将该帧设为非淘汰，并通过 `disk_scheduler_` 的 `DeallocatePage` 释放磁盘空间
  6. 最后重置帧状态
- `WritePageGuard` 简单来说就是RAII自动管理锁和pin计数的模块，具体实现可参考 `page_guard.h` 
- `CheckedWritePage(page_id_t page_id, AccessType access_type = AccessType::Unknown)`：获取一个只读的页面保护，确保对页面数据的线程安全访问，注意流程如下：
  1. 检查页面是否已在缓冲池中；若在缓冲池中则获取内存中的页，然后通过 replacer_ 记录访问并置该页为非淘汰状态；
  2. 若页面不在内存中，尝试分配（`free_frames_`）或淘汰帧（调用`replacer_->Evict()`，注意淘汰时需要处理脏页，然后需要从页表中删除被淘汰的页）；
  3. 从磁盘读取目标页面到刚才分配或淘汰的帧中，并更新缓冲池元数据（帧对应的新page_id，引用计数、is_dirty_等）
- `CheckedReadPage(page_id_t page_id, AccessType access_type = AccessType::Unknown)`：以检查模式读取页面，并返回一个 `ReadPageGuard` 对象；操作流程和 CheckedWritePage 相似，区别仅只是返回对象的类型不同
- `FlushPageUnsafe(page_id_t page_id)`：不安全地刷新页面，其实就是只加缓冲池全局锁来确保页表遍历期间的一致性，但是不获取帧锁，因此刷新过程中页面可能被其他线程修改，导致数据不一致，因此在判断该页是脏页后，需要立即将该页置为干净页，以免其他线程同时修改。不过该方式仅能一定程度上防止多线程修改，但是不能保证一定有效，因此存在风险。我这里建议将 `is_dirty_` 的类型修改为原子类型，然后通过 CAS 保证全局一致性，这样既能实现了高效的批量刷新，也能一定程度上保证数据安全。
- `FlushPage(page_id_t page_id)`： 用于将给定 ID 的页面数据安全刷新至磁盘，具体流程为：首先获取缓冲池全局锁以确保线程安全，随后检查该页面是否存在于内存的页表中，若不存在则直接返回；若存在则获取该页面对应的内存帧，通过获取帧的写锁来确保刷新过程中无其他线程访问，接着判断页面是否为脏页，若是脏页则通过 `disk_scheduler_->Schedule` 将数据刷新至磁盘，并在刷新完成后清除脏页标志。
- `FlushAllPagesUnsafe()`：不安全地刷新所有页面，主要思路是先遍历所有页，然后复用`FlushPageUnsafe(page_id_t page_id)`的操作。
- 需要注意，该函数不需要通过 `future` 异步等待结果，将写申请加入请求队列后直接遍历下一个页，无需判断上一个页是否写入磁盘。
- `FlushAllPages()`：将内存中所有页面数据安全刷新到磁盘，主要思路是先遍历所有页，然后复用`FlushPage(page_id_t page_id)`的操作即可；
- `GetPinCount(page_id_t page_id)`：获取指定页面的引用计数，很简单，加锁然后从页表中获取页对应的帧，然后返回帧中的引用计数即可，若页表中不存在该页，返回 `std::nullopt` 即可。

### page_guard.h

`page_guard.h` 主要为了实现 `ReadPageGuard` 和 `WritePageGuard`，其实就是RAII自动管理锁和pin计数的辅助类，主要思想是通过RAII保证：

- 自动加锁 / 解锁：构造函数获取锁，析构函数释放锁
- 引用计数管理：自动维护页面的固定计数（`pin_count_`）
- 线程安全：通过读写锁（`std::shared_mutex`）实现读 - 写分离

协助我们避免大多数复杂重复操作。

`ReadPageGuard` 和 `WritePageGuard`大部分方法都相同除了少部分代码，因此可以考虑设计一个基类 `PageGuard` ，让`ReadPageGuard` 和 `WritePageGuard`继承。基本代码如下：

```cpp
class PageGuard {
 protected:
  page_id_t page_id_;
  std::shared_ptr<FrameHeader> frame_;
  std::shared_ptr<LRUKReplacer> replacer_;
  std::shared_ptr<std::mutex> bpm_latch_;
  std::shared_ptr<DiskScheduler> disk_scheduler_;
  bool is_valid_;

  virtual void AcquireLock() = 0;
  virtual void ReleaseLock() = 0;

 public:
  auto GetPageId() const -> page_id_t;
  auto GetData() const -> const char *;
  void Flush();
  void Drop();
  PageGuard(PageGuard &&that) noexcept;
  auto operator=(PageGuard &&that) noexcept -> PageGuard &;
  virtual ~PageGuard() = default;
};
```

首先看一下保护成员：

- `page_id_` 是准备读/写页面的唯一 id
- `frame_` 是存储待读/写页面数据帧的共享指针
- `replacer_` 用于管理页面替换
- `bpm_latch_` 保护内部数据结构的锁（修改替换器状态（如标记帧可淘汰）或操作帧元数据时，需持有此锁）
- `disk_scheduler_` 调度器，用来管理页面和磁盘直接的数据读写
- `is_valid_` 标记当前 `Guard` 是否持有合法资源，只有当前 `Guard` 持有合法资源时，才可以通过析构或者 `Drop()` 释放资源

然后就是实现 RAII 的重点，我们需要在构造函数中获取读/写锁，然后在析构函数中释放，并且能自动维护页面的引用计数，先从**构造函数**说起：

1. 在 `ReadPageGuard` 的五形参构造函数中，我们需要先将形参接收的资源通过 `std::move` 转移到当前对象中，然后将当前对象的有效标记 `is_valid_` 为 `true` 表示对象持有合法资源。

   然后获取 `frame_` 的**读锁**（调用 `std::shared_mutex::lock_shared` 方法，该方法能获取互斥的共享所有权，若有其他线程以排他性所有权保有互斥，则`lock_shared`的调用者将阻塞执行，直到能取得共享所有权）。

   此外还需要对互斥 `bpm_latch_` 加锁（因为 `frame_` 的**读锁**用于保护帧所存储页面的数据，而`bpm_latch_` 保护当前`Guard`对象的数据结构；），然后通过 `replacer_` 将当前帧存储页置为非淘汰状态，并修改帧的引用计数。

2. 在移动构造函数中，我们将另一个 `Guard` 对象的资源通过 `std::move` 转移后，需要显式将另一个对象的 `is_valid_` 置为 `false`，以免其意外释放资源

3. 在移动赋值运算符中，除了需要重复移动构造函数的操作外，还需要执行 `Drop()` 释放当前对象的资源，因为在调用 `=` 之前，当前对象可能持有其他合法资源，我们需要在获取另一个对象资源前释放当前持有资源。

4. `Drop()` 用于提前手动释放`Guard`持有的资源，实现时需要确保资源被正确释放且不会导致双重释放。大概思路是先分别判断 `is_valid_` 和 `pin_count_--`，**只有**当前者有效时我们才能保证当前 Guard 持有的资源是合法有效的，当后者等于零时（`pin_count_`减一后等于零）才可以将页面标定为淘汰，最后再释放读锁。

5. `Flush()`用于手动**刷新脏页到磁盘**，大概思路是判断当前对象是否有效或其帧对象标记为脏页，二者满足其一则通过 `disk_scheduler_` 构造写请求将当前页刷新到磁盘，最后将帧标记为干净页。

> 我们需要在 `WritePageGuard::GetDataMut()`函数中手动将指定帧的 `is_dirty_` 置为 `true`，以防数据丢失。

### Q&A

1. `CheckedWritePage` 函数中我首先加了全局锁 `std::scoped_lock<std::mutex> lk(*bpm_latch_)`，然后在 `WritePageGuard` 对象时，我又对 `bpm_latch_` 进行加锁，导致死锁，这里其实可以减少锁粒度，`CheckedWritePage` 中的锁在 return  `WritePageGuard`  时就结束，在 `WritePageGuard`  中重新开始加锁。

2. `GetDataMut()` 函数在返回缓存区指针前，必须将 `is_dirty_` 手动设置为 `true`，否则在我们手动执行 `Drop()` 后，该帧的数据被置为可淘汰，如果此时调用了 `NewPage()`，就有可能丢失该页面的数据，我这里在 `DropTest` 测试中就遇到了这个情况。

## test

用到的测试文件如下：

- `PageGuard: test/storage/page_guard_test.cpp` 
- `BufferPoolManager: test/buffer/buffer_pool_manager_test.cpp` 

首先`cd`到`build`目录下，然后将`page_guard_test`和`buffer_pool_manager_test`中测试函数第二个形参的前缀`DISABLE_`去掉，执行命令：

```
make page_guard_test -j `nproc`
./test/page_guard_test

make buffer_pool_manager_test -j `nproc`
./test/buffer_pool_manager_test
```

本地测试全都成功：

![image-20250526113620711](/images/$%7Bfiilename%7D/image-20250526113620711.png)

![image-20250526203530583](/images/$%7Bfiilename%7D/image-20250526203530583.png)

最后测试的时候才发现我实现的是2025 Spring 的TASK，晕头了我，其实2025相比2024只不过多实现了几个函数，差距不大。

## submit

首先在本地进行格式验证，先安装`clang-fromat,clang-tidy`

```
sudo apt-get install clang-format
apt-get install clang-tidy
```

然后在build文件夹下依次执行以下命令：

```
make format
make check-clang-tidy-p1
```

如果报错提示：

![image-20250430193208062](/images/$%7Bfiilename%7D/image-20250430193208062.png)

那么需要我们手动指定 `clang-format` 的路径，再重新进行 CMake 配置和编译：

```
rm -rf CMakeCache.txt CMakeFiles
cmake -DCLANG_FORMAT_BIN=$(which clang-format) ..
```

要是还有错误，可执行下述指令自行调试，**注意修改文件地址**：

```
cd build
clang-tidy -p . \
  /mnt/datab/home/yuanwenzheng/cmu-15445/cmu-15445/src/storage/page/page_guard.cpp \
  /mnt/datab/home/yuanwenzheng/cmu-15445/cmu-15445/src/buffer/buffer_pool_manager.cpp \
  /mnt/datab/home/yuanwenzheng/cmu-15445/cmu-15445/src/storage/disk/disk_scheduler.cpp \
  /mnt/datab/home/yuanwenzheng/cmu-15445/cmu-15445/src/buffer/lru_k_replacer.cpp
```

然后重新执行 `make format`

最后 cd到build下执行：

```
make submit-p1
```

在根目录下生成名为 `project1-submission.zip` 的压缩包，里面主要包含了PROJECT#1的三个TASK的头文件和cpp文件。我们将  `project1-submission.zip`  提交至Gradescope：https://www.gradescope.com/courses/817456即可。

我提交检测的时候，说除了所需的代码文件外，还需要 `GRADESCOPE.md` 签名文件，大概查了下，这是23年开始新加的要求，除了PROJECT#0不需要外，其他项目都需要。运行下面指令生成：

```
cd ..
python3 gradescope_sign.py
```

然后填一下自己的名字、院校和Github ID即可，GRADESCOPE.md 会自行添加至刚才生成的`project1-submission.zip`压缩包中。

### Q&A

> 1. BufferPoolManagerTest.StaitcaseTest 测试失败

![image-20250527174051902](/images/$%7Bfiilename%7D/image-20250527174051902.png)

在调试过程中发现，当调用 FlushPage 函数时存在无法获取 frame->rwlatch 的情况，项目提示需对页面加锁以确保线程安全，但实际添加共享锁或独占锁后均导致 StairCaseTest 因死锁失败。经分析推测，若采用后台多线程机制执行刷新操作，可能需要通过对 page_id 加锁或建立 page_id 与 thread_id 的映射关系来规避冲突；然而当前在 FlushPage 函数中尝试添加锁机制时，无论采用何种锁模式均会触发死锁，该现象表明锁粒度或加锁时序可能存在设计缺陷，需进一步优化多线程环境下的锁管理策略以平衡并发安全性与死锁风险。

Discord 中已经有人给出了回答：

> I think that comment was added later on, and the fall 2024 code looked like this: https://github.com/cmu-db/bustub/blob/01a64ffdbad34b4bf0693096382c44e3107ba690/src/buffer/buffer_pool_manager.cpp 
>
> and the comment about taking the page lock was added in this PR https://github.com/cmu-db/bustub/commit/3e933255eff5600b8d083cca73ad583ec9f6e6a4 
>
> In the PR for that commit, #800, there's a devastating line: Note that previously, the buffer pool manager only had unsafe flush methods. Which imo seems like pretty good confirmation that we're not supposed to lock.

因此，在实现 FlushPage 函数时无需考虑帧锁。



> 2. BufferPoolManagerTest.ConcurrentReaderWriterTest 测试失败

![image-20250527174102347](/images/$%7Bfiilename%7D/image-20250527174102347.png)

CheckedReadPage 和 CheckedWritePage 中的加锁流程有一些问题，应该按这样的顺序操作：加锁 → 获取空闲帧 → 解锁 → 再次加锁（在页面守卫（page guard）构造函数内部，用于处理引用计数等操作）→ 解锁。修复方案是将所有缓冲池管理器（BPM）的操作放在同一个临界区内完成，然后解锁，最后在页面守卫的构造函数中再尝试获取对应的帧锁（frame latch）。

究其原因其实还是在两个bpm加解锁的过程中，其他线程将当前线程处理的帧淘汰了，因为需要将bpm的操作在一个临界区内完成。



> 3. BufferPoolManagerTest.SchedulerTest 测试失败

![image-20250527174438234](/images/$%7Bfiilename%7D/image-20250527174438234.png)

![image-20250527181510423](/images/$%7Bfiilename%7D/image-20250527181510423.png)

这个问题看日志可以得出来是测试预期页面没有被交换到磁盘，考虑到应该是调用 NewPage() 函数时，我判断只有脏页才会 swap 到磁盘，但测试的目的明显是无论是否脏页，都需 swap 到磁盘。修改完提交，测试全部通过：

![image-20250527181426016](/images/$%7Bfiilename%7D/image-20250527181426016.png)

![image-20250527182856290](/images/$%7Bfiilename%7D/image-20250527182856290.png)

排名在中游左右，其实想想还有很多优化的地方：

1. bpm锁粒度过大，我在bpm中有很多函数是直接加锁，并没有区分临界区，导致浪费了资源
2. bpm锁用的是unique_lock，其实可以考虑使用scope_lock或者lock_guard进一步节省开销
3. 目前的 Buffer Pool 是单线程的，可以考虑创建多个Buffer Pools，并使用Hashing进行控制用哪个buffer pool

