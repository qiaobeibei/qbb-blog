---
title: CMU-15445(6)——PROJECT#2-BPlusTree-Task#1
date: 2025-07-01 17:54:05
categories:
- CMU-15445
tags: 
- Fall 2024
typora-root-url: ./..
---

# PROJECT#2-B+Tree

在 PROJECT#2 中，我们需要实现一个B plus Tree，用过 MySQL 的同学肯定对它不陌生，B+Tree是实现高效数据检索的核心组件，其内部节点的作用是引导搜索过程，而实际的数据项则存于叶子节点中。该索引结构能够实现快速的数据检索，无需对数据库表中的每一行数据进行扫描，可实现快速随机查找以及对有序记录的高效扫描。

举个例子，如果我们要用某个给定的 id 来检索某个货物的记录，没有索引结构的情况下，我们只能从第一条记录开始遍历每个货物的记录，直到找到某个ID和我们给定的ID一致的记录，时间复杂度是O(N)。常见的索引结构有二叉树、红黑树、哈希表和B+Tree，而如果我们维护了一个以ID为KEY的索引结构，我们可以通过这个索引结构查询这个ID对应的货物所在的位置，然后直接从这个位置读取数据，B+Tree能保证 O (log n) 时间复杂度的检索操作。

在着手项目开发前，建议先系统学习 B + 树数据结构的核心原理，预先夯实知识基础。

## B+Tree

DB 的数据一般保存在磁盘上，这样的好处是持久化，但也有很大的缺点：**读写特别慢**。

磁盘读写的最小单位是**扇区**，扇区的大小只有 `512B` 大小，操作系统一次会读写多个扇区，所以操作系统的最小读写单位是**块**(Block)。Linux 中的块（页）大小为 `4KB` ，也就是一次磁盘 I/0 操作会直接读写8个扇区。

由于数据库的索引是保存到磁盘上的，因此当我们通过索引查找某行数据的时候，就需要先**从磁盘读取索引到内存，再通过索引从磁盘中找到某行数据，然后读入到内存**，也就是说查询过程中会发生多次磁盘//O，而磁盘 I/0 次数越多，所消耗的时间也就越大（如果没有索引结构，那么就需要从头开始将每一行数据的索引从磁盘读到内存和目标索引进行对比，I/O次数会特别多）。

所以，我们希望索引的数据结构能在尽可能少的磁盘的 1/0 操作中完成查询工作，因为磁盘 I/0 操作越少，所消耗的时间也就越小。

因此，一个合格的数据结构需要满足以下要求：

- 能在尽可能**少**的磁盘的 /O 操作中完成查询工作；
- 要能高效地查询**某一个记录**，也要能高效地执行**范围查找**;

如果可以将索引按顺序（增序或降序），那么可以通过二分查找来降低查询时间复杂度到O(logN)，进一步优化到二分查找树、自平衡二叉树、B树....，这一部分内容可参考文章：

[>>>为什么 MySQL 采用 B+ 树作为索引？ | 小林coding<<<](https://xiaolincoding.com/mysql/index/why_index_chose_bpuls_tree.html#什么是二分查找)

索引结构的迭代优化总结起来其实就是以下内容：

在数据结构的索引场景中，若将索引存储于数组结构，线性查找的时间复杂度为 O (N)，而采用二分查找可将复杂度降至 O (logN)。尽管数组的线性存储方式实现简单，但插入操作需移动后续所有元素，导致 O (N) 级的时间开销。由此引出二叉树结构，其通过指针链接节点实现非连续存储，兼具二分查找特性，但当二叉搜索树退化为单支树（如全右子树或全左子树）时，时间复杂度会退化为 O (N)。

为解决此类平衡性问题，AVL 树与红黑树等平衡二叉搜索树应运而生。然而，无论何种平衡树结构，其树高仍随数据量呈 O (logN) 增长，这直接导致磁盘 I/O 次数随树高增加而显著上升。B 树的出现通过提升节点扇出能力（单个节点可包含多个子节点）有效降低树高，但传统 B 树节点存储索引与数据记录的复合信息，当用户数据记录尺寸远大于索引键时，会引发两大问题：

1. 非目标节点的数据记录加载会消耗额外 I/O 资源；
2. 无效数据占用内存空间，降低缓存命中率。

B+树其实就是B树的升级，MySQL 中索引的数据结构就是采用了 B+ 树，B+ 树结构如下图：

<div style="text-align: center;">   <img src="/images/$%7Bfiilename%7D/b6678c667053a356f46fc5691d2f5878.png" width="800"> </div>

<center>图片来源：https://xiaolincoding.com/mysql/index/why_index_chose_bpuls_tree.html#%E4%BB%80%E4%B9%88%E6%98%AF-b-%E6%A0%91-2</center>

B+ 树与 B 树差异的点，主要是以下这几点：

- 叶子节点（最底部的节点）才会存放实际数据（索引+记录），非叶子节点只会存放索引；
- 所有索引都会在叶子节点出现，叶子节点之间构成一个有序链表；
- 非叶子节点的索引也会同时存在在子节点中，并且是在子节点中所有索引的最大（或最小）。
- 非叶子节点中有多少个子节点，就有多少个索引；

如下图所示（课程ppt示例图）：

<div style="text-align: center;">   <img src="/images/$%7Bfiilename%7D/image-20250528115452211.png" width="600"> </div>

所有数据均保存至叶子结点 `Leaf Nodes`；内部节点 `Inner Node` 仅仅充当控制查找记录的媒介，并不代表数据本身，所有的内部结点元素都同时存在于子结点中，是子节点元素中是最大（或最小）元素。

如上图，`5` 是其子结点的最大值， `9` 是其子结点的最小值，所有的数据 `[1,3,6,7,9,13...]` 均保存至叶子结点中，而根结点 `[5,9]`均不是数据本身，只是充当控制查找记录的媒介。

总而言之，一个完整的 B+ 树由根节点、内部节点、叶子节点三部分组成。每个节点均有键（key）和值（value）组成，区别在于内部节点的值指向子节点的指针（通常是页 ID，用于导航到下一层节点），而叶节点的值指向实际数据记录的引用（通常是 RID，用于定位存储在磁盘上的完整元组）。每个节点通常对应数据库缓冲池中的一个物理页面，通过唯一的页 ID标识。

内部节点的结构如下图所示：

1. 每个内部节 `Inner Nodes` 点都由 `[P,K]` 组成，P指向子树根结点的指针；`K` 表示关键字值，也就是索引。

2. 内部节点的关键字值有如下规律：
   $$
   K₁<K₂<…<K_{c-1}
   $$

<div style="text-align: center;">   <img src="/images/$%7Bfiilename%7D/image-20250701110918230.png" width="800"> </div>

叶子节点的结构如下图所示：

1. 叶子节点的值指向磁盘中的某块数据地址；
2. 存在指针 `P_next` 指向下一个叶子节点，方便遍历查询。

<div style="text-align: center;">   <img src="/images/${fiilename}/image-20250701113810900.png" width="700"> </div>

关于 B+ 树的插入、删除可参考文章：[B+树看这一篇就够了（B+树查找、插入、删除全上） - 知乎](https://zhuanlan.zhihu.com/p/149287061)

参考：[为什么 MySQL 采用 B+ 树作为索引？ | 小林coding](https://xiaolincoding.com/mysql/index/why_index_chose_bpuls_tree.html#什么是-b-树-2)

## TASK#1-B+Tree Pages

了解完 B+ 树的基础知识后，便可以着手实现 `TASK#1`。

我们现在很清楚一个完整的 B+ 树由根节点、内部节点和叶子节点三部分组成，其实三者都是同一类型，只不过根据位置不同，不同节点存储的数据对象不同。

在 Project 中，B+ 树所有节点都以 `B+Tree Page` 的形式存在，根据节点类型不同，又分为 `B+Tree Internal Page` 和 `B+Tree Leaf Page`，二者均继承自 `B+Tree Page`，区别只在于二者存储的数据类型不同：

1. `B+Tree Internal Page` 的 `Value` 为子树根节点的索引；
2. `B+Tree Leaf Page` 的 Value 为元组 RID。

本节的目的就是实现以下三个页：

- B+Tree Page
- B+Tree Internal Page
- B+Tree Leaf Page

### Base-Tree

需要修改的文件：

- `src/include/storage/page/b_plus_tree_page.h`
- `src/storage/page/b_plus_tree_page.cpp`

文件中的抽象类 `BPlusTreePage` 是B+树所有节点类型的公共抽象，包含了内部节点和叶节点共享的核心属性和方法，我们实现的函数多数是 `Get/Set`，因此比较简单。

因为内部节点和叶子节点都是  `BPlusTreePage` 的派生，因此每个 B + 树页面的起始位置都包含了三元素：`PageType (4) | CurrentSize (4) | MaxSize (4)`，分别代表：

- `PageType`：4 字节，标识页面类型
- `CurrentSize`：4 字节，当前页面中的键值对数量
- `MaxSize`：4 字节，页面可容纳的最大键值对数量

一共 **12B** 的不包含键值对的信息被称为 `HEADER`，分别对应 `BPlusTreePage` 的成员变量：

```cpp
private:
  // Member variables, attributes that both internal and leaf page share
  IndexPageType page_type_ __attribute__((__unused__));
  // Number of key & value pairs in a page
  int size_ __attribute__((__unused__));
  // Max number of key & value pairs in a page
  int max_size_ __attribute__((__unused__));
```

其中，`IndexPageType` 是定义的枚举类：

```cpp
enum class IndexPageType { INVALID_INDEX_PAGE = 0, LEAF_PAGE, INTERNAL_PAGE };
```

- `INVALID_INDEX_PAGE = 0`：无效页面（初始状态或已释放的页面）
- `LEAF_PAGE`：叶子节点页面（存储实际数据记录的引用）
- `INTERNAL_PAGE`：内部节点页面（存储索引键和子节点指针）

此外，尽管这里 B+ 树的所有节点均以 `B+Tree Page` 的形式存在，但和 lab1 的 Page 是有本质上的区别，`BPlusTreePage` 实际对应BPM 中  `Page` 对象的 `data_` 存储区域。因此在读取或写入节点时，首先需要通过 `BPM.CheckedReadPage(page_id)` 获取受保护可读的 `std::optional<ReadPageGuard>` 对象（获取可写对象也如此），再将其 `data_` 部分 `reinterpret_cast` 为 `BPlusTreePage`，最后根据对应的 `page_type_` 强转为 `Internal/Leaf page`，然后在读取或写入后取消固定该页面。具体数据排布如下图所示：

<div style="text-align: center;">   <img src="/images/$%7Bfiilename%7D/image-20250701154206559.png" width="900"> </div>

实现整体比较简单，需要注意的只有`GetMinSize()` ，对于内部节点和叶子节点，最小大小有不同的计算方式。

> 最大键值对数量为 N 时，节点的最少键值对数量存在三种情况：
>
> - 根节点：
>   - 根节点是叶节点时，内部至少需要 1 个键值对，这个很好理解，空树插入首个元素时，根节点必须至少有 1 个键值对（如初始插入场景）
>   - 根节点是内部节点时，内部至少需要 2 个键值对，因为内部节点需指向至少 2 个子节点（左子树和右子树），因此至少需要 1 个有效键（实际用于索引查询的键值对） + 1 个哨兵键（内部节点最左侧的无效键）
> - 内部节点：节点插入数据之后可能溢出，这时需要进行分裂操作，为了简化分裂代码的编写，内部节点和根节点会留出一个键值对的位置作为哨兵，实际最大键值对数量为 N−1，加上最左侧的无效键，最小键值对数量为 ⌈(N−2)/2⌉+1；
>   - (N−2)：减去哨兵键和 1 个分裂键
>   - 加 1 是因为哨兵键必须保留在左子树
> - 叶节点：最小键值对数量为 ⌈(N−1)/2⌉，因为叶节点无需哨兵键，最大有效键值对为 N−1（留出 1 个空位用于分裂）

未完待续。。。
