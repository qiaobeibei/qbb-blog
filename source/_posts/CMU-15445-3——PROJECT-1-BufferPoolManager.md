---
title: CMU-15445(3)——PROJECT#1-BufferPoolManager-Task#1
date: 2025-05-05 16:51:51
categories:
- CMU-15445
tags: 
- Fall 2024
typora-root-url: ./..
---

# PROJECT#1-BufferPoolManager

在完成了前面基础的PROJECT#0后，从本节开始才正式进入了CMU-15445的学习，最终目的是构建一个面向磁盘的数据库管理系统。

`PROJECT#1` 的主要任务是实现数据库管理系统的**缓冲池管理器**，缓冲池负责在主存缓冲区与持久化存储（硬盘）之间来回移动数据的物理页（虚拟内存通过内存交换实现运行内存超过物理内存的大小），同时充当缓存 —— 将频繁使用的页面保留在内存中以加快访问速度，并将未使用或不活跃的页面**淘汰**回存储设备。

在操作系统中，如果虚拟内存采取内存分页的方式进行管理，那么通常会将整个物理内存通过单位页进行划分，单位页的大小是 4KB，因此本节的缓冲池管理器也同样以 4KB  为单位管理数据。由于 BusTub 中的页大小固定，缓冲池管理器将这些页存储在称为**帧**的固定大小缓冲区中。

- **页**是 4 KB 的**逻辑（虚拟）数据**，可存储在内存、磁盘或同时存在于两者中；
- **帧**是固定长度的 4 KB 内存块（即指向该内存的指针），用于存储单个页的数据，只能存在于内存中。

二者的关系可类比为将（逻辑）页存储在（物理）固定大小的帧中。缓冲池管理器通过帧来管理内存中的页 —— 当需要访问磁盘上的页时，会将页的数据加载到某个空闲的帧中（类似把书从仓库搬到书桌的格子里），方便快速访问。

> 举个例子：
>
> - 当数据库需要处理一个页时（比如查询某条数据），缓冲池管理器会先检查该页是否已在某个帧中（即是否在内存里）：
>   - 如果在，直接使用帧中的数据（快速访问）；
>   - 如果不在，从磁盘读取该页的数据，放入一个空闲的帧中（类似从仓库搬书到书桌的格子）。
> - 当内存不够时，缓冲池会根据策略（如 LRU）淘汰某些帧中的页（把书从格子里放回仓库），腾出空间给新的页。

除了能作为缓存，缓冲池管理器使数据库管理系统能够支持容量超过系统可用内存的数据库，主要就是利用虚拟内存的思想。

> 操作系统本身就有缓存机制，为什么DBMS还需要使用独立的 Buffer Pool？

形象点来说，数据库的缓冲池就像书店专属的高效展示区，能精准管理热销书、记录修改细节、优化取书流程，比商场共用区域更懂书店的生意（数据库的特殊需求），所以必须自己建而不是 “蹭” 公共区域。

缓冲池其实就是为了减少OS从磁盘进行IO的次数，当 DBMS 请求一个页时，该页的副本会被放入缓冲池的某个帧中，此后，当再次请求该页时，系统会先搜索缓冲池：若页存在于缓冲池中，则直接使用；若不存在，则从磁盘读取副本。如下图：

![image-20250505172515214](/images/$%7Bfiilename%7D/image-20250505172515214.png)

以上的介绍便是PROJECT#1的主要任务，主要通过以下三个TASK实现：

![image-20250505173643766](/images/$%7Bfiilename%7D/image-20250505173643766.png)

- LRU-K Replacement Policy
- Disk Scheduler
- Buffer Pool Manager

上图中的 `Page Table`（页表）通过哈希表实现，用于跟踪当前存在于内存中的页，将页 ID 映射到缓冲池中的帧位置（其实和虚拟内存中的页表一个思路，缓冲池的页表不应与页目录混淆，后者是页 ID 到数据库文件中页位置的映射）。`LRU-K Replacement Policy` 其实就是实现如何通过页表跟踪缓冲池并且记录页面的使用情况进行相应的淘汰过程。

`Disk Scheduler` 负责缓冲池和硬盘之间的读写，因为页表还维护了每页的额外元数据、`dirty-flag` 和 `pin/reference counter`，每当线程修改页面时，`dirty-flag` 都会由线程设置；且线程在访问页面之前必须递增计数器，如果页面的计数大于零，则不允许存储管理器从内存中淘汰该页面。这些步骤都是需要我们在 `Disk Scheduler` 实现的，并且需要保证**线程安全**。

上面两个 TASK 是主要内容的实现，`Buffer Pool Manager` 则是通过 `LRU-K Replacement Policy` 和 Disk Scheduler 从磁盘获取数据库页面，并将它们存储在内存中，必要时也可以安排将脏页写回磁盘。

每一个TASK在具体实现的时候介绍。

> 注意：缓冲池是多线程并发的组件，因此必须要保证线程安全问题，这里可以考虑使用互斥或者原子操作搭配内存序实现。

该课程的讲义ppt可通过以下链接获取：

[Kangyupl/CMU15445-slide-and-note - 码云 - 开源中国](https://gitee.com/kangyupl/cmu15445-slide-and-note)

## Task #1 - LRU-K Replacement Policy

LRU-K Replacement Policy 的任务总结起来就是负责跟踪缓冲池中页面的使用情况，以便在需要为新页面腾出内存空间时，确定应该将哪些页面 / 帧从内存中淘汰并写回到磁盘。

实现该 TASK 需要使用到以下文件：

- `src/include/buffer/lru_k_replacer.h`
- `src/buffer/lru_k_replacer.cpp`

注意到文件 buffer 下有除 `lru_k_replacer.cpp` 外名为 `lru_replacer.cpp` 和 `clock_replacer.cpp` 的文件，很明显，缓冲区除了`LRU-K` 外还有 `LRU`、`LFU`以及`clock` 等 Replacement Policy，那为什么这里使用 `LRU-K` 取代 `LRU`和`CLOCK`呢？

LRU 和 LFU 其实是数据库常使用的一种策略，前者基于 “最近最少使用的页面在未来一段时间内也不太可能被使用” 这一假设，它会维护一个页面访问顺序的列表，每当一个页面被访问时，就将其移动到列表头部。当需要淘汰页面时，选择列表尾部（即最近最少使用）的页面进行淘汰。可以参考力扣的题解进行理解：[【图解】一张图秒懂 LRU！](https://leetcode.cn/problems/lru-cache/solutions/2456294/tu-jie-yi-zhang-tu-miao-dong-lrupythonja-czgt/?envType=study-plan-v2&envId=top-100-liked)

Clock 策略是 LRU 的一种近似算法，该策略为每个页面设置一个引用位，初始值为 0。当页面被访问时，引用位被置为 1。系统使用一个类似时钟指针的机制在页面列表中循环扫描，当扫描到一个页面时，如果其引用位为 1，则将其置为 0 并继续扫描；如果引用位为 0，则淘汰该页面。如下图所示：

![image-20250505181355462](/images/$%7Bfiilename%7D/image-20250505181355462.png)

尽管这两种方式都有现有的实现方式，但二者均存在许多问题。比如二者很容易受到 `sequential flooding` 的影响，比如当进行一次大规模的顺序扫描时，会将大量近期不会再使用的页面加载到缓存中，挤掉原本可能会频繁使用的页面，导致缓存命中率下降。

举例说明：

1. 执行Q1，当id=0时，读取了page0，将page0换入到缓冲池：

   ![img](/images/$%7Bfiilename%7D/word-image-50.png)

2. 接下来执行Q2，以此读取page1、page2、page3、page … …等等后面的page。

   ![img](/images/$%7Bfiilename%7D/word-image.gif)

   但是在想换入page3的时候，缓冲池空间不够了。如果使用的是LRU算法则会把page0换出。但是我们还要用page0，没有办法，page0只能不断的被换入换出，这样就降低了效率。

可通过以下三种方式解决该问题：

1. `LRU-K`，也就是在该PROJECT中实现的算法，其中 K 是针对单个页面（page）所对应的缓存数据，需要对其访问次数进行计数（若 K 取值为 2，那就意味着要统计每个页面的最近两次访问情况；若 K 为 3，就是统计最近三次的访问情况）。**核心思路**是将最近使用过1次的判断标准扩展为最近使用过 K 次，也就是说当某个页面的访问次数还没有达到 K 次时，该页面的访问记录不会被无限地记录下去，把这部分数据存放在一个 “历史队列” 中，临时存放访问次数较少、还不能确定其是否经常被使用的数据；一旦某个页面的访问次数达到了 K 次，就表明这个页面是比较频繁被使用的，是 “热点数据”，此时，会把该数据的索引从 “历史队列” 移动到 “缓存队列”。

   “缓存队列” 里存放的都是那些经常被访问的数据，当需要淘汰数据时，会优先从 “历史队列” 中选择数据进行淘汰（LRU），当“历史队列”中的数据淘汰完后，通过**倒数第 K 次**的访问时间与当前时间的距离作为其距离（替换权重）来选择性的将“缓存队列”中的数据淘汰。

   举个例子来说，K=2，5个块的访问时间历史如下，时间从1开始，当前时间为12：

   ```
   块1：1
   块2：2
   块3：3、6、9、11
   块4：4、7、
   块5：5、8、10
   ```

   由于访问次数不足K，块1与块2的LRU-K距离为正无穷（+inf）；块 3的距离为12-9=3；块4的距离为12-4=8；块5的距离为：12-8=4。如果五个块均可以被替换，那么根据LRU-K算法，块1将最先被替换出去，接着被替换出去的是块2，然后依次是4、5、3。

2. 多缓冲区

3. 优先级

### lru_k_replacer.h

头文件定义了 `LRUKNode` 和 `LRUKReplacer` 两个类，前者表示一个页面节点，存储页面的访问历史等信息；后者需要我们实现 `LRU-K Replacement Policy`，其实就是实现“历史队列”，存放那些使用频率不高的页用于淘汰。源代码如下:

```cpp
namespace bustub {

enum class AccessType { Unknown = 0, Lookup, Scan, Index };

class LRUKNode {
 private:
  /** History of last seen K timestamps of this page. Least recent timestamp stored in front. */
  // Remove maybe_unused if you start using them. Feel free to change the member variables as you want.

  [[maybe_unused]] std::list<size_t> history_;
  [[maybe_unused]] size_t k_;
  [[maybe_unused]] frame_id_t fid_;
  [[maybe_unused]] bool is_evictable_{false};
};

class LRUKReplacer {
 public:
  explicit LRUKReplacer(size_t num_frames, size_t k);

  DISALLOW_COPY_AND_MOVE(LRUKReplacer);

  /**
   * TODO(P1): Add implementation
   *
   * @brief Destroys the LRUReplacer.
   */
  ~LRUKReplacer() = default;

  auto Evict() -> std::optional<frame_id_t>;

  void RecordAccess(frame_id_t frame_id, AccessType access_type = AccessType::Unknown);

  void SetEvictable(frame_id_t frame_id, bool set_evictable);

  void Remove(frame_id_t frame_id);

  auto Size() -> size_t;

 private:
  // TODO(student): implement me! You can replace these member variables as you like.
  // Remove maybe_unused if you start using them.
  [[maybe_unused]] std::unordered_map<frame_id_t, LRUKNode> node_store_;
  [[maybe_unused]] size_t current_timestamp_{0};
  [[maybe_unused]] size_t curr_size_{0};
  [[maybe_unused]] size_t replacer_size_;
  [[maybe_unused]] size_t k_;
  [[maybe_unused]] std::mutex latch_;
};

}  // namespace bustub
```

`LRUKNode` 类代表一个页面节点，存储页面的访问历史和状态信息。简单分析 `LRUKNode`  的成员信息：

```cpp
class LRUKNode {
 private:
  [[maybe_unused]] std::list<size_t> history_;
  [[maybe_unused]] size_t k_;
  [[maybe_unused]] frame_id_t fid_;
  [[maybe_unused]] bool is_evictable_{false};
};
```

- `history_` 记录该页面最近 K 次访问的时间戳，最旧的时间戳存于列表头部，新时间戳通过 `push_back` 存于列表尾部
- `k_` 明显是 K 值
- `fid_` 表示缓冲区中帧的 id
- `is_evictable_` 表示该帧是否可被淘汰

> 如果我们想要使用该类的某些成员，需要将 `[[maybe_unused]]` 删除

虽然存在其他的策略，比如LRU和时钟，但我们这里仅需要实现LRU-K.同样分析一下 `LRUKReplacer` 的功能和成员：

```cpp
[[maybe_unused]] std::unordered_map<frame_id_t, LRUKNode> node_store_;
[[maybe_unused]] size_t current_timestamp_{0};
[[maybe_unused]] size_t curr_size_{0};
[[maybe_unused]] size_t replacer_size_;
[[maybe_unused]] size_t k_;
[[maybe_unused]] std::mutex latch_;
```

私有成员如上，后续我们需要根据相应情况进行使用和增加。

- `node_store_` 是帧 ID 到 `LRUKNode` 的映射，其实就是页表`Page Table`，用于跟踪当前存在于内存中的页，将页映射到缓冲池中的帧位置。
- `current_timestamp_` 当前的时间戳
- `curr_size_` 当前可淘汰页的个数，也是 `size()` 的返回值，`curr_size_` 在多线程情况下修改可能会造成资源竞争的问题，若使用互斥保护锁粒度过于大，**这里可将其类型修改为 `std::atomic<size_t>`，避免资源竞争。**
- `replacer_size_` 表示缓冲池帧的的总上限

分析一下公有函数：

1. `explicit LRUKReplacer(size_t num_frames, size_t k)`：构造函数，以缓冲池的帧数 `num_frames` 和 `K` 值 `k` 作为参数，并且使用宏 `DISALLOW_COPY_AND_MOVE` 禁止了 `LRUKReplacer` 的拷贝和移动，详细定义如下：

   ```cpp
   #define DISALLOW_COPY(cname)                                    \
     cname(const cname &) = delete;                   /* NOLINT */ \
     auto operator=(const cname &)->cname & = delete; /* NOLINT */
   
   #define DISALLOW_MOVE(cname)                               \
     cname(cname &&) = delete;                   /* NOLINT */ \
     auto operator=(cname &&)->cname & = delete; /* NOLINT */
   
   #define DISALLOW_COPY_AND_MOVE(cname) \
     DISALLOW_COPY(cname);               \
     DISALLOW_MOVE(cname);
   ```

   `DISALLOW_COPY_AND_MOVE` 中结合了 `DISALLOW_COPY` 和 `DISALLOW_MOVE`，将给定类的拷贝和移动构造函数和运算符主动 `delete`，仅可以通过公有的构造函数在定义。

   > 因为我们使用的是单缓冲池，我们可以使用单例模式优化，将构造函数私有化，避免资源浪费；但是在多缓冲池下就不要使用单例模式了。

2. `Evict() -> std::optional<frame_id_t>`：淘汰与 `LRUKReplacer` 所追踪的其他所有可淘汰帧相比，反向 k 距离最大的帧。若没有可淘汰的帧，则返回 `std::nullopt`。**其实就是将“历史队列”中最久未被使用的页淘汰出去**。

3. `RecordAccess(frame_id_t frame_id)`：记录给定的帧在当前时间戳已被访问。在缓冲池管理器中固定一个页面后，应调用此方法。**其实就是将当前的时间戳存储到给定帧id对应页的 `history_` 中，从而记录每次访问的时间戳，进而辅助计算反向 K 距离**。

4. `Remove(frame_id_t frame_id)`：清除与一个帧相关的所有访问历史。**其实就是当一个页不再需要时，删除与之管理的访问信息，从 `node_store_` 删除与 `frame_id` 对应的条目，避免无效的访问历史记录干扰 LRU - K 替换策略的决策**。

   > Evict 和 Remove 看起来作用很相似，但有很大差别：
   >
   > - `Remove` ：只有在缓冲池管理器中删除一个页面时才会被调用。删除页面的原因可能有很多，比如用户显式删除了某个数据，或者系统进行了一些清理操作等
   > - `Evict` ：当缓冲池已满，需要加载新的页面但没有可用的空闲帧时会被调用。此时，`Evict` 方法会根据 LRU-K 选择一个最合适的帧将其中的页数据**淘汰到硬盘**，以腾出空间来加载新的页面
   >
   > 很明显，前者就是真实删除，该页的数据不再被需要，不仅需要从缓冲区的帧中将该页删除，同时也需要将 `LRUKReplacer` 中记录的信息删除，维护 `LRUKReplacer` 中数据的一致性和有效性，避免无效的访问历史记录干扰 `LRU - K` 替换策略的决策。
   >
   > 而后者的目的是解决缓冲池空间不足的问题，将一个不常用的页面淘汰到磁盘，为新的页面腾出内存空间，保证系统的正常运行，该页的数据并没有被删除，而是暂时移动到了硬盘中。
   >
   > 此外，`Evict()` 是将反向K距离最大的页淘汰，而`Remove`是将给定帧id对应的页淘汰，无论它的反向K距离是多少。

5. `SetEvictable(frame_id_t frame_id, bool set_evictable)`：用于控制一个帧是否可被淘汰，同时也会控制 `LRUKReplacer` 的大小。**其实就是当一个页面的固定计数变为 0 时，将其对应的帧应标记为可淘汰。**

6. `Size() -> size_t`：返回当前 `LRUKReplacer` 中可淘汰帧的数量，大小是动态的，其返回值等于当前所有 `is_evictable_ = true` 的帧的数量，只有可淘汰的帧才会被纳入 “淘汰候选集”，不可淘汰的帧（如正在被使用的帧）不会被 `Evict ()` 方法考虑。

`LRUKReplacer` 的最大容量与缓冲池的大小相同，包含了缓冲池管理器中所有帧的占位符，无论该帧当前是否可被淘汰，`LRUKReplacer`都需要跟踪它的访问历史和状态。

`LRUKReplacer` 的大小由可淘汰帧的数量来表示。`LRUKReplacer` 初始时不包含任何帧（`size()` 为0或者说`curr_size_`是0，即使 `node_store_` 中已经为所有帧创建了占位符），只有当一个帧被标记为可淘汰时，`LRUKReplacer`的大小才会增加。同样，当一个帧被固定或未被使用时，替换器的大小会减小。

当一个帧被用户线程引用的次数变为 0 时（意味着该帧当前未被使用），缓冲池管理器会调用 `SetEvictable(frame_id, true)`，将该帧标记为可淘汰，`LRUKReplacer`的大小（即可淘汰帧的数量）会增加 1。当帧被重新固定（即被用户线程引用）时，缓冲池管理器会调用 `SetEvictable(frame_id, false)`，将该帧标记为不可淘汰，`LRUKReplacer`的大小减少 1。

### lru_k_replacer.cpp

代码由于课程要求不会公开，这里说一下我实现的思路。

1. 首先要在`LRUKReplacer`中调用`LRUKNode`的私有变量，要么对于`LRUKNode`，实现其构造函数，然后实现一下辅助函数用于设置和返回私有变量：

   ```cpp
   auto GetHistory() -> std::list<size_t> & ;
     auto GetK() -> size_t ;
     auto GetFid() -> frame_id_t;
     auto SetEvictable(bool flag) -> void;
     auto GetEvictable() -> bool;
   ```

   要么将`LRUKReplacer` 设为 `LRUKNode` 的友元类

   ```cpp
   friend class LRUKReplacer;
   ```

   此外，`LRUKNode` 除 `LRUKNode(size_t k, frame_id_t fid)` 外还需要指定默认构造函数 `LRUKNode() = default`，因为在 `LRUKReplacer` 使用`std::unordered_map` 的 `operator[]` 时，若指定的键不存在于映射中，会首先调用 `LRUKNode` 的默认构造，然后才会报错。要么定义`LRUKNode`的默认构造，要么使用`emplace`插入。

2. `LRUKReplacer`构造函数很简单，将 num_frames 和 k 赋值给对应变量即可。

   > `num_frames`  需要通过 `static_cast<frame_id_t>` 转换为 `frame_id_t`。

3. 实现 `Evict() -> std::optional<frame_id_t>` 之前，需要先实现一个辅助函数 `CalculateBackwardKDistance(const LRUKNode& node)` 计算给定帧id对应页的反向K距离：如果给定`LRUKNode`的访问记录次数小于`K`，则返回`inf`，反之找到倒数第`K`次的访问时间，然后返回当前时间戳与倒数第`K`次的访问时间的差。

   在 `Evict()` 中定义一个比较函数 `cmp`，比较传入两个 `frame_id_t` 对应页的优先级关系（反向K距离越大优先级越大，若反向K距离相同则比较最早访问时间，越早访问优先级越高），然后定义一个优先队列 `std::priority_queue<frame_id_t, std::vector<frame_id_t>, decltype(cmp)>`，底层容器使用 `std::vector` 方便随机访问，比较函数使用我们定义的 `cmp`，优先级从小到大依次排列。

   > `cmp` 接受两个元素作为参数，并返回一个布尔值。如果比较函数返回 `true`，则第一个元素的优先级低于第二个元素；如果返回 `false`，则第一个元素的优先级高于第二个元素。

   然后加锁遍历`node_store`_，若`node`的标记`is_evictable_`为`true`，则将该`node`对应的`frame_id_t`加入到优先队列中进行排序；

   如果优先队列不为空，则删除 `node_store_` 中优先队列队首元素代表的 `LRUKNode` （经`cmp`排序后，优先级最大的帧在队首），并修改 `curr_size_`，反之返回 `std::nullopt`。

   > `node_store_` 和 `curr_size`_是需要保护的共享资源，在使用和修改的时候需要注意进行加锁，如果 `curr_size_` 的类型被修改为了 `std::atomic<size_t>`则只需要考虑保护`node_store_` 。

   其实优先队列最好分成“历史队列”和“缓存队列”，前者存放历史记录不满 k 的帧，后者存放历史记录满 k 的帧。

   ```cpp
   // 历史队列，存放历史记录不满 k 的帧
   using HistoryQueueEntry = std::pair<size_t, frame_id_t>;
   std::priority_queue<HistoryQueueEntry, std::vector<HistoryQueueEntry>, std::greater<>> history_queue_;
   
   // 缓存队列，存放历史记录满 k 的帧
   using CacheQueueEntry = std::pair<size_t, frame_id_t>;
   std::priority_queue<CacheQueueEntry, std::vector<CacheQueueEntry>, std::greater<>> cache_queue_;
   ```

4. `RecordAccess(frame_id_t frame_id, [[maybe_unused]] AccessType access_type)` 主要用于记录给定 `frame_id` 对应的页在当前时间戳被访问，同时需要保证 `frame_id` 范围在 `[0, num_frames - 1]` 中，否则利用已经定义好的宏`BUSTUB_ASSERT`断言。

   需要注意，因为有可能在调用 `RecordAccess` 之前系统调用了 `Evict()` ，因此即使构造函数中**所有可能的帧 ID** 预先创建了对应的 `LRUKNode` 对象，但有可能在 `Evict()` 中将某一个帧id对应的`LRUKNode` 删除，因此我们必须判断`node_store_`是否存在给定帧id对应的`LRUKNode`，若没有则创建一个。

   在 `LRUKNode` 的历史记录中加当前时间戳后（`current_timestamp_++`），**仅需要保存 `history_` 中的后 k 个记录，剩余部分需要 pop以节约空间。**

   > 验证帧id有效性时，需要将 replacer_size_ 的类型强制转换 `static_cast<frame_id_t>`

5. `Remove(frame_id_t frame_id)->void`需要经过三次验证检查，第一次验证 `frame_id` 是否有效（在`[0, num_frames - 1]` 中），第二次需要验证`frame_id`对应的`LRUKNode`是否存在，第三次验证帧是否可淘汰，即只有在“历史队列”中的帧才可以淘汰。经过三次验证检查后，**帧及其访问历史**才会被移除。

   > 第一次和第三次检查时通过 `BUSTUB_ASSERT` 来终止程序，而第二次检查不成功会直接 return，不会终止。

6. `SetEvictable()` 和 `Size()` 比较简单，前者检查帧id有效性和其对应的页是否存在，并根据标志位相应的增加或删除 `curr_size_`；后者直接返回`curr_size_`即可。

### test

先将 `./test/buffer/lru_k_replacer_test.cpp` 下第一个测试函数第二个形参的前缀 DISABLE_ 删除；

然后从根目录cd至build，运行：

```
make lru_k_replacer_test -j `nproc`
./test/lru_k_replacer_test
```

![image-20250508164855326](/images/$%7Bfiilename%7D/image-20250508164855326.png)

参考：

[CMU15-445-P1全局思路及详细实现过程（超超超超详细，我奶都能看懂！！！）-CSDN博客](https://blog.csdn.net/weixin_49614712/article/details/142899300?spm=1001.2014.3001.5502)

[CMU15-445数据库系统：缓存池 - 高志远的个人主页](https://gaozhiyuan.net/database/cmu-database-systems-buffer-pools.html)

[CMU15445 2024Spring 课程作业_cmu15445 gradescope-CSDN博客](https://blog.csdn.net/qq_28625359/article/details/137108932?spm=1001.2014.3001.5502)
