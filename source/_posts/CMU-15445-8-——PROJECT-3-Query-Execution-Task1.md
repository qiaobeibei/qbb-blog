---
title: CMU-15445(8)——PROJECT#3-Query Execution-Task#1
date: 2025-07-21 14:12:31
categories:
- CMU-15445
tags: 
- Fall 2024
typora-root-url: ./..
---

# PROJECT#3 - Query Execution

到目前为止，我们已经实现了 lab1 和 lab2。其中，lab1 的 bmp 像 “中转站”，负责内存与磁盘数据的合理调度和一致性维护，实现高效存储；lab2实现了并发 B + 树索引，为快速查询提供支持。在 lab3中，我们需要基于火山模型实现查询执行器并整合前两者功能，通过 “流水线” 接口将 SQL 查询转为扫描、关联等操作步骤，完成从存储到返回结果的全流程。

bustub 提供了 [BusTub Shell](https://15445.courses.cs.cmu.edu/fall2024/bustub/)，我们可以通过网页运行 SQL或debug。

## Task#0 Read the Source Code

在完成 lab3 之前，我们需要了解一条 SQL 是如何被数据库执行的？

![img](https://15445.courses.cs.cmu.edu/fall2024/project3/img/project-structure.svg)

官网给出了一个详细的示例图。

自上而下，SQL语句首先进入 Parser 生成抽象语法树，随后由 Binder 做名称解析与类型补全（比如把 `SELECT *` 展开成具体列、把表名解析成系统目录里的表OID和每列的序号），后面所有步骤都不再用字符串而是用这些编号来精确定位。接着 Planner 把中间表示排成可执行步骤，产出一颗初始的物理计划树；Optimizer 的作用相当于编译优化器，在不改变语义的前提下换更快的执行路线，比如把等值内连接从“双重循环”的Nested loop Join 更换为先建哈希表再探测的 Hash Join，后面需要实现的 NLJ->Hash Join规则主要负责的就是这一步。

完成优化后，Executors 就像运行时代码按流水线把数据一条条推上去：

- Scan/Values/Insert/Update/Delete 负责真正读写表数据
- Join/Aggregation/Projection/Filter 负责把上一环节吐出的元组做连接、分组、投影和过滤，然后把结果继续往上游返回。

Executors 每次需要读写数据时，会通过 Transaction Manager 发起访问，相当于带锁的文件系统接口，负责并发控制与隔离级别（这部分在 lab4 中实现），保证多会话下读写一致、崩溃可恢复。

真正的数据落在两条存储路径上：要么直接走 Table Heap（2023 lab2 的实现），要么走 Index（2024 fall lab2 中实现的并发B+树索引）。这两条路径最终都会下沉到bpm（页缓存，尽量命中内存、减少磁盘IO） 和 disk manager（真正读写磁盘页），类似于OS的页缓存和块设备驱动。

根据这个图可以清晰的串联四个 lab：lab1 是缓冲池和磁盘这个底层地基；lab2 是索引，提供高效定位；lab3 是从计划、优化到执行器把查询真正跑起来；lab4 是事务与并发把实现的一切放在真实多会话环境中仍能正常运行。

简而言之，整个图表达的就是SQL自上而下的数据路径：SQL被解析、绑定、规划、优化，然后由执行器在事务保护下访问索引或表堆，经由bpm落到磁盘，再把结果一路向上返回。配合 EXPLAN 可以在 [BusTub Shell](https://15445.courses.cs.cmu.edu/fall2024/bustub/) 中直接看到 Planner/Optimizer -> Executors 的对应关系与变化（[课程页](https://15445.courses.cs.cmu.edu/fall2024/project3/)有示例）。

这个图各阶段更详细的介绍可以参考这篇文章：[做个数据库：2022 CMU15-445 Project3 Query Execution - 知乎](https://zhuanlan.zhihu.com/p/587566135)

## Task#1 Access Method Executors

Task#1 要把“访问方法”这一层打通：让计划树最底层能真正读写表与索引，从而支撑后面 Join/Agg 等算子的工作。

执行器都遵循同一个生命周期：**Init 只做一次初始化，随后重复调用 Next 逐条产生或消费元组，直到返回 false**。它们通过 Catalog 找到目标表/索引，通过 Transaction 贯穿整个操作（真正的锁/恢复放在 lab4），底层读写依赖 Table Heap与B+树索引，缓冲命中与落盘由bpm 负责。

Task#1 包含 5 个算子，SeqScan、Insert、Update、Delete 和 IndexScan，均是**按行处理数据**。

所有算子均基于火山模型设计：

```cpp
class AbstractExecutor {
 public:
  virtual void Init() = 0;
  virtual auto Next(Tuple *tuple, RID *rid) -> bool = 0;
  virtual auto GetOutputSchema() const -> const Schema & = 0;
  auto GetExecutorContext() -> ExecutorContext * { return exec_ctx_; }
};
```

每个算子都有 `Init()` 和 `Next()` 两个方法。`Init()` 对算子进行初始化工作。`Next()` 则是向下层算子请求下一条数据。当 `Next()` 返回 false 时，则代表下层算子已经没有剩余数据，迭代结束。

火山模型的好处就是支持流水线处理，不需要将所有数据加载到内存中，而是按需求逐个处理元组。当上层执行器调用下层执行器的 next 方法时，数据就像火山喷发一样从底层向上流动。尽管火山模型占用内存较小，但函数调用开销大，特别是虚函数调用造成 cache miss 等问题。

### SeqScan

这个执行器对应 `SELECT ...FROM table [WHERE ...]` 的最普通的路径。工作原理是按照物理存储顺序遍历表中的每一个页面和每一个元组，当查询引擎需要执行一个没有索引可用的查询时，就会使用SeqScan。在Init 阶段通过 `Catalog::GetTable(table_oid)` 取到table heap与表的Schema，并创建一个表迭代器指向表的起始页。

Next 反复前进迭代器，逐页逐槽读取 tuple 与它的 RID；若计划节点附带谓词，就用表达式求值判真后再产出；产出的列布局以计划节点的输出 Schema 为准（通常是表列的子集或同序拷贝）。由于 bustub 的列引用都在 Binder 阶段固化成 “#child_idx.col_idx”，谓词与投影的 Evaluate 不需要字符串查找，直接按位置取值，这也解释了为什么 Binder 会把名字解析为 OID、列序列号与 tuple_idx。

[文章](https://zhuanlan.zhihu.com/p/587566135)给出了 Bustub 中 table 的结构：

![img](/images/$%7Bfiilename%7D/v2-9bc6214441f8ca37004ff1389114a692_1440w.jpg)

> 首先，Bustub 有一个 Catalog。Catalog 提供了一系列 API，例如 `CreateTable()`、`GetTable()` 等等。Catalog 维护了几张 hashmap，保存了 table id 和 table name 到 table info 的映射关系。table id 由 Catalog 在新建 table 时自动分配，table name 则由用户指定。
>
> 这里的 table info 包含了一张 table 的 metadata，有 schema、name、id 和指向 table heap 的指针。系统的其他部分想要访问一张 table 时，先使用 name 或 id 从 Catalog 得到 table info，再访问 table info 中的 table heap 。
>
> table heap 是管理 table 数据的结构，包含 `InsertTuple()`、`MarkDelete()` 一系列 table 相关操作。table heap 本身并不直接存储 tuple 数据，tuple 数据都存放在 table page 中。table heap 可能由多个 table page 组成，仅保存其第一个 table page 的 page id。需要访问某个 table page 时，通过 page id 经由 buffer pool 访问。
>
> table page 是实际存储 table 数据的结构，父类是 page。相较于 page，table page 多了一些新的方法。table page 在 data 的开头存放了 next page id、prev page id 等信息，将多个 table page 连成一个双向链表，便于整张 table 的遍历操作。当需要新增 tuple 时，table heap 会找到当前属于自己的最后一张 table page，尝试插入，若最后一张 table page 已满，则新建一张 table page 插入 tuple。table page 低地址存放 header，tuple 从高地址也就是 table page 尾部开始插入。
>
> tuple 对应数据表中的一行数据。每个 tuple 都由 RID 唯一标识。RID 由 page id + slot num 构成。tuple 由 value 组成，value 的个数和类型由 table info 中的 schema 指定。
>
> value 则是某个字段具体的值，value 本身还保存了类型信息。
>
> 注意，executor 并不内置查询计划，而是通过其plan成员确定执行逻辑；比如顺序/区间访问可由indexScan依据计划节点在叶链上迭代完成。执行过程中所需的共享资源比如catalog、bpm等仍统一由 `ExecutorContext` 提供。

------

> 修改文件：src/execution/seq_scan_executor.cpp

在 seq_scan_executor.h 中，我们需要重写 AbstractExecutor 的 Init()、Next()和 GetOutputSchema()。

首先声明一个 table iterator 作为私有成员变量 table_iter_ ，然后在初始化阶段从 catalog 获取表信息，并初始化表迭代器：

```cpp
// 从 catalog 获取表信息
auto table_info = exec_ctx_->GetCatalog()->GetTable(plan_->GetTableOid());
BUSTUB_ASSERT(table_info != nullptr, "Table not found");
// 初始化 table iterator
table_iter_ = std::make_unique<TableIterator>(table_info->table_->MakeEagerIterator());
```

exec_ctx_是各种算子的上下文，提供 transaction、catalog、bpm等公共资源，简单理解成一个资源提供方。

plan_  存储了seqScan 的执行计划节点，包含了扫描所需的所有信息，比如 table OID、outputSchema和过滤谓词。

plan_  有很多定义其实放在 SeqScanExecutor 中会更好理解，因为 Init() 从 plan_   获取的 table OID、 过滤谓词，以及 GetOutputSchema中获取的outputSchema，其实都可以当作 SeqScanExecutor  的私有成员变量。但是考虑到代码模块化，因此将定义和执行解耦，方便维护。

所以，SeqScanPlanNode 相当于 SeqScanExecutor  的定义类。

```cpp
auto SeqScanExecutor::Next(Tuple *tuple, RID *rid) -> bool { 
  // 从 catalog 获取表信息
  auto table_info = exec_ctx_->GetCatalog()->GetTable(plan_->GetTableOid());
  BUSTUB_ASSERT(table_info != nullptr, "Table not found");
  // 遍历表，直到找到有效的 tuple 或 end
  while (!table_iter_->IsEnd()) {
    auto [tuple_meta, table_tuple] = table_iter_->GetTuple();
    *rid = table_iter_->GetRID();
    // 移动到下一个 tuple
    ++(*table_iter_);
    // 跳过已删除的元组
    if (tuple_meta.is_deleted_) {
      continue;
    }
    // 是否需要过滤
    if (plan_->filter_predicate_ != nullptr) {
      auto value = plan_->filter_predicate_->Evaluate(&table_tuple, table_info->schema_);
      if (value.IsNull() || !value.GetAs<bool>()) { // 过滤不满足条件的 tuple
        continue;
      }
    }
    // 对元组投影，以匹配输出 schema
    std::vector<Value> values;
    values.reserve(GetOutputSchema().GetColumnCount());

    for (uint32_t i = 0; i < GetOutputSchema().GetColumnCount(); ++i) {
      values.push_back(table_tuple.GetValue(&table_info->schema_, i));
    }
    *tuple = Tuple(values, &GetOutputSchema());
    return true;
  }
  return false;
}
```

在 Next() 的实现中，执行器首先检查元组的元数据，跳过已经被标记为删除的元组（为了支持事务的并发执行，采用的逻辑删除方式，元组被删除时并不会立即从磁盘中移除，而是在元组的元数据中标记为已删除）。

然后执行查询优化中的谓词下推。如果查询包含 WHERE 子句，执行器会在数据源层面就应用这些过滤条件，而不是将所有数据传递给上层之后再进行过滤。这样的好处就是可以尽早减少需要处理的数据流，提高整体查询性能。这里通过调用过滤谓词的 Evaluate 方法判断当前元组是否满足查询条件，不满足跳过。

最后执行投影，即只返回查询所需要的列。数据库表的物理存储通常包含所有列的数据，但查询可能只需要其中的一部分列，执行器根据输出模式的定义，从完整的元组中提取出所需要的列值，构造一个新的元组返回给上层执行器。

执行器每次只处理一个元组，处理完后立即返回给上层，而不需要等待所有数据都处理完毕。

### Insert 

> 修改文件：src/execution/insert_executor.cpp

insert/delete/update 算子和其他算子有一个本质区别：它们不是流式处理数据，而是批量处理所有数据（仍然是逐行处理，只不过结果汇总返回，而不是**一次性**批量处理所有数据），然后返回一个汇总结果。

首先额外声明三个私有变量：

```cpp
/** the child executor from which tuples are pulled */
std::unique_ptr<AbstractExecutor> child_executor_;
/** number of tuples inserted */
int inserted_count_;
/** whether the insert has finished */
bool finished_{false};
```

插入操作的执行流程始于**从子执行器获取要插入的数据**。在数据库的查询执行树中，插入执行器通常位于树的根部，而其子执行器负责产生要插入的元组数据（语句通常带`WHERE`条件，子执行器的作用就是筛选出符合条件的元组），比如在 `INSERT BALUES(1, 'hello'), (2, 'world')` 语句中，子执行器是 ValueExecutor，它会逐个产生这些要插入的元组。而在 `INSERT INTO table1 SELECT * FROM table2` 语句中，子执行器可能是查询执行器，负责从另一个表中读取数据。

插入执行器通过调用子执行器的 next 方法来逐个获取要插入的元组。

```cpp
auto InsertExecutor::Next([[maybe_unused]] Tuple *tuple, RID *rid) -> bool {
  if (finished_) {
    return false;
  }
  auto table_info = exec_ctx_->GetCatalog()->GetTable(plan_->GetTableOid());
  BUSTUB_ASSERT(table_info != nullptr, "Table not found");
  // 获得该表全部索引
  auto indexes = exec_ctx_->GetCatalog()->GetTableIndexes(table_info->name_);
  Tuple child_tuple{};
  RID child_rid{};
  // 插入子执行器提供的所有元组
  while (child_executor_->Next(&child_tuple, &child_rid)) {
    // 创建表元数据
    TupleMeta tuple_meta{0, false};
    // 插入元组
    auto insert_rid = table_info->table_->InsertTuple(tuple_meta, child_tuple,
                                                       exec_ctx_->GetLockManager(),
                                                       exec_ctx_->GetTransaction(),
                                                       plan_->GetTableOid());
    if (insert_rid.has_value()) {
      inserted_count_++;
      // 更新所有索引
      for (auto &index_info : indexes) {
        // 从元组中生成索引键
        auto key_tuple = child_tuple.KeyFromTuple(table_info->schema_, index_info->key_schema_,
                                                   index_info->index_->GetKeyAttrs());
        index_info->index_->InsertEntry(key_tuple, insert_rid.value(), exec_ctx_->GetTransaction());
      }
    }
  }
  // 返回插入的行数
  std::vector<Value> values;
  values.push_back(Value(TypeId::INTEGER, inserted_count_));
  *tuple = Tuple{values, &GetOutputSchema()};

  finished_ = true;
  return true;
}
```

实际的数据插入流程和数据delete比较相似，这里先说前者。插入执行器调用表的 InsertTuple 方法，将元组及其元数据写入到表的存储结构中。这个过程涉及到缓冲池管理、页面分配、以及可能的页面分裂等底层存储操作，如果插入成功，系统会返回新元组的RID。

数据库系统的一个关键特性是索引与表数据的一致性维护，当一个新元组被插入到表中后，系统必须同时更新表上的所有索引。插入执行器通过系统目录获取该表的所有索引信息，然后为每个索引提取相应的键值。这个过程需要根据索引的定义，从完整的元组中提取出构成索引键的那些列的值，然后将这个索引键和新元组的RID一起插入到索引结构中。

整个插入过程必须在事务的上下文中执行，以确保操作的原子性。如果插入过程中发生任何错误，比如违反了唯一性约束或者系统资源不足，整个插入操作都会被回滚，保证数据库的一致性状态。

插入操作的另一个特点是采用批量处理的方式，与SeqScan 每次返回一个数据元组不同，插入执行器会处理所有来自子执行器的元组**（当然还是逐行处理，一次次的将结果汇总，然后返回汇总数据，而不是一次性插入然后返回）**，然后返回一个包含插入行数的结果元组。

### Update

> 修改文件：src/execution/update_executor.cpp

Update 和 Insert 类似，依赖于“子执行器产出待更新的数据->用 SET 表达式重新计算新值->在表堆中写入新版本并标删旧版本->维护相关索引” 这样的一条流水线。

Planner 会把 SQL 的 UPDATE 转成一个 UpdatePlanNode; 真正决定哪些行需要更新的是它的子执行器（多数情况下是 SeqScan/IndexScan + Filter 的组合）。因此，一个 UPDATE 的执行过程通常为：

1. 从子执行器不断取出旧元组以及旧RID；
2. 对每条旧元组按 SET 子句对应的表达式逐项 Evaluate 得到新值；
3. 构造新元组并插入到 table heap；
4. 把旧元组标记删除；
5. 然后对每个关联索引做 “删除旧键，插入新键”，确保二级索引与基表一致。

等全部处理完，Update 执行器返回受影响的行数。

```cpp
auto UpdateExecutor::Next([[maybe_unused]] Tuple *tuple, RID *rid) -> bool {
  if (finished_) {
    return false;
  }
  // 获取该表的全部索引
  auto indexes = exec_ctx_->GetCatalog()->GetTableIndexes(table_info_->name_);
  Tuple child_tuple{};
  RID child_rid{};
  // 更新子执行器提供的所有元组
  while (child_executor_->Next(&child_tuple, &child_rid)) {
    // 获取旧元组
    auto [old_meta, old_tuple] = table_info_->table_->GetTuple(child_rid);
    // 该元组是否被标删
    if (old_meta.is_deleted_) {
      continue;
    }
    // 计算新值
    std::vector<Value> new_values{};
    new_values.reserve(plan_->target_expressions_.size());
    for (const auto &expr : plan_->target_expressions_) {
      new_values.push_back(expr->Evaluate(&child_tuple, child_executor_->GetOutputSchema()));
    }
    // 创建新元组
    Tuple new_tuple{new_values, &table_info_->schema_};
    // 更新表
    TupleMeta new_meta{0, false};
    auto new_rid = table_info_->table_->InsertTuple(new_meta, new_tuple,
    exec_ctx_->GetLockManager(),
    exec_ctx_->GetTransaction(),
    plan_->GetTableOid());
    if (new_rid.has_value()) {
      // 将旧元组标删
      TupleMeta delete_meta{0, true};
      table_info_->table_->UpdateTupleMeta(delete_meta, child_rid);
      ++updated_count_;
      // 更新全部索引
      for (auto& index_info : indexes) {
        // 从旧元组中提取索引键
         auto old_key = old_tuple.KeyFromTuple(
                        table_info_->schema_,        // 表结构
                        index_info->key_schema_,     // 索引键的结构
                        index_info->index_->GetKeyAttrs()  // 索引依赖的字段（比如“年龄”字段）
                      );
        // 删除旧索引条目
        index_info->index_->DeleteEntry(old_key, child_rid, exec_ctx_->GetTransaction());

        // 从新元组中提取索引键
        auto new_key = new_tuple.KeyFromTuple(table_info_->schema_, index_info->key_schema_, index_info->index_->GetKeyAttrs());
        // 插入新索引条目
        index_info->index_->InsertEntry(new_key, new_rid.value(), exec_ctx_->GetTransaction());
      }
    }
  }
  // 返回更新行数
  std::vector<Value> values;
  values.push_back(Value(TypeId::INTEGER, updated_count_));
  *tuple = Tuple{values, &GetOutputSchema()};
  finished_ = true;
  return true;
}
```

UpdateExecutor 完整的执行了上面提到的“子执行器产出待更新的数据->用 SET 表达式重新计算新值->在表堆中写入新版本并标删旧版本->维护相关索引” 这条范式。构造时先通过 Catalog 用表 OID 拿到 table_info；初始化时启动子执行器、清零计数器并复位finished_。在 Next() 中进入循环，从子执行器不断获取（tuple, rid)，再通过 table->GetTuple(rid) 取到”真实旧值“和元数据，并进行标删判断。

新值的计算依赖于 Planner 填入的 target_expressions 与子执行器输出 schema 做列绑定，逐一 Evaluate 生成 Value 数组，再用表的 schema 构造新元组。

> 例如 `UPDATE students SET age=20, name='Alice' WHERE id=1`，`SET` 后面的每个赋值（`age=20`、`name='Alice'`）都会被解析为一个 `target_expression`，存储在 `plan_->target_expressions_` 中，`target_expressions_` 的长度等于需要更新的字段数量。

为了保证索引和数据的一致性（比如 “年龄” 索引会映射年龄到行位置，当数据更新后，索引必须同步修改，否则查询会出错），这里采用”插入新版本+标删旧版本“的方法。先 InsertTuple 拿到新的 RID，再把旧版本的 UpdateTupleMeta 标记 is_delete_ = true。虽然这种”写时复制“的方法会改变 RID，但可以将数据与索引维护的逻辑简化：对每个索引，分别用 old_tuple/new_tuple 与索引的 key_schema、key_attrs 组合生成键，执行 DeleteEntry(child_rid) 和 InsertEntry(new_rid)，从而保证所有二级索引在更新后可精确定位新版本。

所有行处理后，返回一个仅有一列的元组，表示更新的行数，并设置 finished_，确保 Update 执行器只返回一次有效结果。

### Delete

> src/execution/delete_executor.cpp

删除流程和Insert/update基本相同，不做分析

### IndexScan

> src/execution/index_scan_executor.cpp

IndexScan 主要用于通过表上的索引快速定位并获取满足条件的元组。使用我们在 Project 2 中实现的 B+Tree Index Iterator，遍历 B+ 树叶子节点。由于我们实现的是非聚簇索引，在叶子节点只能获取到 RID，需要拿着 RID 去 table 查询对应的 tuple。流程大概为：

先用 B+ 树按键值把候选行的 RID 快速定位出来，再回表取出完整元组、做必要的剩余过滤和投影，按输出模式流式返回结果。

在 Planner 阶段，IndexScanPlanNode 会指明要用的索引、目标表，以及能从谓词里提取的索引键，同时保留没被索引完全覆盖的那部分谓词作为残余过滤。

```cpp
void IndexScanExecutor::Init() { 
  auto index_info = exec_ctx_->GetCatalog()->GetIndex(plan_->GetIndexOid());
  BUSTUB_ASSERT(index_info != nullptr, "Index not found");

  auto table_info = exec_ctx_->GetCatalog()->GetTable(plan_->table_oid_);
  BUSTUB_ASSERT(table_info != nullptr, "Table not found");

  // 若存在谓词键，使用它们进行点查询
  if (!plan_->pred_keys_.empty()) {
    // 计算谓词键获取搜索键
    std::vector<Value> key_values;
    for (const auto& expr : plan_->pred_keys_) {
      // 常量表达式可使用空元组计算
      key_values.push_back(expr->Evaluate(nullptr, index_info->key_schema_));
    }

    Tuple search_key{key_values, &index_info->key_schema_};

    // 扫描匹配的RID
    index_info->index_->ScanKey(search_key, &result_rids_, exec_ctx_->GetTransaction());
  } else {
    // 对于全索引扫描，B+树
    use_iterator_ = true;
    it_ = tree_->GetBeginIterator();
    end_it_ = tree_->GetEndIterator();
    // 使用迭代器不需要result_rids
    result_rids_.clear();
 
  }

  current_idx_ = 0;
 }
```

执行初始化时，执行器通过系统目录拿到 indexInfo（包括B+树对象、key_schema、key_attrs）和 table_info。若 plan_ 携带等值键（比如WHERE k = 42 这种能被索引精确匹配的条件），就把这些键表达式求值成符合索引 key_schema 的搜索键，用索引的点查接口直把匹配到的RID批量取出来，存到本地列表中；若没有等值键，完整实现使用索引迭代器做范围或全索引扫描，最小实现也可以先只支持点查。

索引和表的元数据拿到，并且后续RID按键检索出后，接下来Next阶段只需要逐个消费这些 RID：回表根据RID取出元组和meta，跳过已经标删的元组；如果plan_里还有残余谓词，就在回表后对元组求值，不满足的继续丢弃；满足的再按输出schema投影列并返回。

```cpp
auto IndexScanExecutor::Next(Tuple *tuple, RID *rid) -> bool { 
  auto table_info = exec_ctx_->GetCatalog()->GetTable(plan_->table_oid_);
  BUSTUB_ASSERT(table_info != nullptr, "Table not found");

  if (use_iterator_) {
    while(!it_.IsEnd()) {
      auto [key, value] = *it_;
      RID current_rid = value;
      ++it_;

      auto [tuple_meta, table_tuple] = table_info->table_->GetTuple(current_rid);
      if (tuple_meta.is_deleted_) {
        continue;
      }

      // 过滤
      if (plan_->filter_predicate_ != nullptr) {
        auto filter_value = plan_->filter_predicate_->Evaluate(&table_tuple, table_info->schema_);
        if (filter_value.IsNull() || !filter_value.GetAs<bool>()) {
          continue;
        }
      }

      // 对元组投影，使其匹配输出 schema（只保留需要的字段）
      std::vector<Value> values;
      values.reserve(GetOutputSchema().GetColumnCount());
      // 遍历输出schema的每个字段，从原始元组中提取对应的值
      for (uint32_t i = 0; i <GetOutputSchema().GetColumnCount(); ++i) {
        values.push_back(table_tuple.GetValue(&table_info->schema_, i));
      }

      *tuple = Tuple{values, &GetOutputSchema()};
      *rid = current_rid;
      return true;
    }
  } else {
    while (current_idx_ < result_rids_.size()) {
        auto current_rid = result_rids_[current_idx_++];
        // 使用索引返回的RID从表中获取元组
        auto [tuple_meta, table_tuple] = table_info->table_->GetTuple(current_rid);

        if (tuple_meta.is_deleted_) {
          continue;
        }

        // 过滤
        if (plan_->filter_predicate_ != nullptr) {
          auto value = plan_->filter_predicate_->Evaluate(&table_tuple, table_info->schema_);
          if (value.IsNull() || !value.GetAs<bool>()) {
            continue;
          }
        }

        // 对元组投影，使其匹配输出 schema（只保留需要的字段）
        std::vector<Value> values;
        values.reserve(GetOutputSchema().GetColumnCount());
        // 遍历输出schema的每个字段，从原始元组中提取对应的值
        for (uint32_t i = 0; i <GetOutputSchema().GetColumnCount(); ++i) {
          values.push_back(table_tuple.GetValue(&table_info->schema_, i));
        }

        *tuple = Tuple{values, &GetOutputSchema()};
        *rid = current_rid;
        return true;
      }
    }
    return false;
  }
```

IndexScan 是一个迭代产出的算子，next 的每一次调用，就尽力返回一条满足条件的记录；当没有可返回的记录时，才给出false结束。intit 阶段已经把能用索引定位到的候选RID准备好了，Next 的工作就是沿着这些候选逐个验证并产出。

具体到流程，Next 会从内部游标到当前位置取一个RID，回表把这条记录读出来，同时拿到它的元数据，并判断标删。如果plan_中还有剩余谓词（索引没有完全覆盖的那部分条件），就用这条回表得到的元组去求值。通过筛选后，再按`plan_`给定的输出schema做投影，把这个元组和对应的RID设置到出参里，立刻返回true。下次再调用next，会从内部游标的下一个候选继续相同过程，直至候选耗尽，函数返回false表示扫描结束。

Next 能保证只返回当前可见且满足条件的行，并且一次只给一条（处理完一条后就return true，等待下一次调用next直至返回false表示无候选数据）。

> **IndexScan 和 SeqScan 的区别**

1. 数据访问路径不同：相比于SeqScan，indexScan首先通过索引把搜索空间裁小（即使不存在谓词键pred_keys_使用B+树索引从头遍历，也不像SeqScan一样逐个遍历数据的物理存储，而是遍历数据的索引结构，后者明显快很多），next再做回表与参与判断，比SeqScan要省得多。

2. 功能不同：SeqScan适合全表扫描，但需要读取所有数据页；IndexScan适合选择性查询，只读取符合条件的元组

3. 执行策略不同：SeqScan总是线性扫描，IndexScan支持点查找（根据精确的键值在索引中查找匹配的记录）和全扫描（使用B+树迭代器遍历索引结构找到符合条件的叶子页）

   1. 简单来说，点查找就是”精确匹配“，直到要找什么，直接去索引中去定位，而不是遍历所有索引结构。比如 `SELECT * FROM students WHERE student_id = 1001;`  student_id = 1001 就是谓词键，可以直接在 students 列中定位  student_id = 1001 的元组，存储起来然后在 Next 阶段处理。

      而 `SELECT * FROM students` 就是全扫描了，直接使用B+树迭代器遍历索引结构找到符合的叶子页（虽然没有谓词键的情况下，IndexScan和SeqScan用的都是全扫描，但是前者扫描的是索引结构，比后者快得多）

# test

bustub 提供了所有测试用例，没有隐藏测试，测试用例位于 test/sql/ 目录下。

先编译

```
make -j$(nproc) sqllogictest
```

基础功能测试：

```
./bin/bustub-sqllogictest ../test/sql/p3.00-primer.slt --verbose
```

![image-20250809150023745](/images/$%7Bfiilename%7D/image-20250809150023745.png)

顺序扫描测试：

```
./bin/bustub-sqllogictest ../test/sql/p3.01-seqscan.slt --verbose
```

![image-20250809150500962](/images/$%7Bfiilename%7D/image-20250809150500962.png)

插入测试：

```
./bin/bustub-sqllogictest ../test/sql/p3.02-insert.slt --verbose
```

![image-20250809150649632](/images/$%7Bfiilename%7D/image-20250809150649632.png)

更新测试：

```
./bin/bustub-sqllogictest ../test/sql/p3.03-update.slt --verbose
```

![image-20250809150823898](/images/$%7Bfiilename%7D/image-20250809150823898.png)

更新会卡住不动，检查发现是 update 的实现会导致插入新元组后，新元组被后续的 update 操作再次处理。修改：

```cpp
// 错误
auto new_rid = table_info_->table_->InsertTuple(new_meta, new_tuple, ...);
if (new_rid.has_value()) {
	TupleMeta delete_meta{0, true};
	table_info_->table_->UpdateTupleMeta(delete_meta, child_rid);
	// ...
}

// 修改为
TupleMeata new_meta{0, false};
tuple_info_->table_->UpdateTupleInPlace(new_meta, new_tuple, child_rid);
```

通过 UpdateTupleInPlace 更新值，而不是删除然后插入。



删除测试：

```
./bin/bustub-sqllogictest ../test/sql/p3.04-delete.slt --verbose
```

![image-20250809150915561](/images/$%7Bfiilename%7D/image-20250809150915561.png)

B+树扫描测试：

```
./bin/bustub-sqllogictest ../test/sql/p3.05-index-scan-btree.slt --verbose
```

![image-20250809151128665](/images/$%7Bfiilename%7D/image-20250809151128665.png)

空表测试：

```
./bin/bustub-sqllogictest ../test/sql/p3.06-empty-table.slt --verbose
```

![image-20250809151217394](/images/$%7Bfiilename%7D/image-20250809151217394.png)

其他测试：

```
./bin/bustub-sqllogictest ../test/sql/intro.slt --verbose
./bin/bustub-sqllogictest ../test/sql/baby_arithmetic.slt --verbose
```

![image-20250809151425825](/images/$%7Bfiilename%7D/image-20250809151425825.png)

![image-20250809151508938](/images/$%7Bfiilename%7D/image-20250809151508938.png)

均通过测试。

下一节尝试实现task2.
