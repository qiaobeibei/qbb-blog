---
title: CMU-15445(9)——PROJECT#3-Query Execution-Task#2 & Task#3
date: 2025-07-23 14:12:31
categories:
- CMU-15445
tags: 
- Fall 2024
typora-root-url: ./..
---

## Task 2 Aggregation & Join Executors

在实现了最基本的 insert、scan、update 等算子之后，我们需要在 task2 中实现三个最核心并且比较复杂的三个算子：Aggregation 、NestedLoopJoin 和 NestedIndexJoin，从而实现聚合和连接操作。

### Aggregation 

Aggregation 采用基于哈希表的分组聚合算法，核心思想是将具有相同分组键的元组映射到哈希表的同一个桶中，并在该桶中维护相应的聚合状态，从而在单次遍历输入数据的情况下完成所有分组的聚合计算。

Aggregation 的实现中定义了一个专门的 SimpleAggregationHashTable 类，该类封装了聚合操作所需的所有核心功能，以此维护一个从  AggregateKey 到 AggregateValue 的映射关系：

```cpp
/** The hash table is just a map from aggregate keys to aggregate values */
std::unordered_map<AggregateKey, AggregateValue> ht_{};
/** The aggregate expressions that we have */
const std::vector<AbstractExpressionRef> &agg_exprs_;
/** The types of aggregations that we have */
const std::vector<AggregationType> &agg_types_;
```

在映射表 ht_ 中，AggregateKey  包含了所有 GROUP BY 的表达式的值，而 AggregateValue  则存储了各个聚合函数的中间结果。光看代码很难立即它的作用，举个例子说明：

假设我们有以下 SQL 查询：

```sql
SELECT department, COUNT(*), SUM(salary), MAX(age)
FROM employees
GROUP BY department
```

对于这个查询， AggregateKey  = {department 的值}，比如{“Engineering”}；AggregateValue = {COUNT(*)、UM(salary)和 MAX(age)的值}，比如{5, 20000, 35}。

假如我们处理的第一条记录为：`("Qiaobeibei", "Engineering", 8000， 25)`，首先提取分组键 department =  “Engineering”，然后创建 AggregateKey{"Engineering"}。因为映射表 ht_ 初始不存在该键对应的值，调用 GenerateInitialAggregateValue() 函数进行初始化（`COUNT(*)` 初始化为0，SUM(salary) 初始为NULL，MAX(age)初始为NULL），然后调用 CombineAggregateValues() 并集（`COUNT(*) = 0 + 1 = 1`，SUM(salary) = NULL + 8000 = 8000, MAX(age) = NULL vs 25 = 25）。映射表的状态如下：

```
{"Engineering"} -> {1, 8000, 25}
```

然后处理第二天记录 `("Beauqiao", "Engineering", 9000, 24)`，同样提取分组键 department = "Engineering"，因为映射表中已经存在了该键，直接合并：COUNT(*) = 1 + 1 =2, SUM(salary) = 8000 + 9000 = 17000, MAX(age) = max(25, 24) = 25。映射表修改为：

```
{"Engineering"} -> {2, 17000, 25}
```

继续处理第三天记录 `("Baby", "Marketing", 7000, 24)`，提取分组键 同样提取分组键 department = "Marketing",因为是新的分组，需要创建新条目：COUNT(*) = 0 + 1 = 1, SUM(salary) = NULL + 7000 = 7000, MAX(age) = NULL vs 25 = 25。映射表最终为：

```
{"Engineering"} -> {2, 17000, 25}
{"Marketing"} -> {1, 7000, 25}
```

在这个过程中，agg_exprs_ 负责存储所有聚合表达式，以此告诉系统对哪些列或表达式进行聚合，而 agg_types_  存储了相应的聚合类型，告诉系统执行怎么样的聚合操作，比如 COUNT、SUM或者MAX。

同样是上面的例子，`("Qiaobeibei", "Engineering", 8000， 25)`，首先提取聚合值：

```
agg_exprs_[0]->Evaluate(tuple) -> 1 (COUNT(*)总是1)
agg_exprs_[1]->Evaluate(tuple) -> 8000 (salary列的值)
agg_exprs_[2]->Evaluate(tuple) -> 25 (age列的值)
```

得到 AggregateValue{1, 8000, 25}，然后合并聚合值：

```cpp
for (uint32_t i = 0; i < agg_exprs_.size(); ++i) {
    switch (agg_types_[i]) {
    	case AggregationType::CountStarAggregate:
            // 处理 COUNT(*): 0 + 1 = 1
    		break;
    	case AggregationType::SumAggregate::
            // 处理 SUM(salary) = NULL + 8000 = 8000
    		break;
    	case AggregationType::MaxAggregate:
            // 处理 MAX(age) = NULL vs 25 = 25
    		break;
    }
}
```

最后将键值插入到映射表 ht_ 中。

简而言之，**`agg_exprs_[i]` 告诉系统如何从输入元组中提取到第 i 个聚合函数所需要的值；`agg_types_[i]` 决定了对第 i 个值执行什么的操作；二者通过索引 i 将表达式和聚合类型关联起来，保证每个聚合函数都能正确输入和处理逻辑；最后按分组键映射所有聚合计算结果到 ht_ 的不同桶中。**

------

了解了 SimpleAggregationHashTable  之后，Aggregation 的实现就比较容易懂。和 update、insert、delete 算子一样，它也是从子执行器获取需要处理的数据，然后从 aht_ 中获取数据**逐行处理**。

在 Init 阶段，Aggregation 需要遍历子执行器产生的所有元组，对每个元组提取分组键和聚合值，然后调用 InsertCombine 方法将其插入到哈希表中（如果分组键已存在，则将当前元组的聚合值与已有值进行合并；否则创建新的聚合条目并初始化为默认值）。

在 Next 阶段，Aggregation 使用迭代器遍历哈希表中的所有聚合结果，每次调用 Next 时，执行器从当前迭代器位置获取一个聚合结果，将分组键和聚合值组合输出元组。特别注意的是，对于没有 GROUP BY 子句的聚合查询，当输入为空时，系统需要返回初始聚合值。例如 COUNT(*) 应该返回 0，而 SUM、MIN、MAX 等函数应该返回 NULL。

实现中通过 switch 对不同聚类类型进行分发处理，每种聚合函数都有专门的合并逻辑。

```cpp
auto AggregationExecutor::Next(Tuple *tuple, RID *rid) -> bool { 
  // 遍历所有聚合结果
  if (aht_iterator_ == aht_.End()) {
    // 处理无 GROUP BY 子句且聚合结果为空（输入数据为空）
    if (plan_->GetGroupBys().empty() && aht_.Begin() == aht_.End()) {
      // 返回初始聚类值（COUNT 为0，MAX等为NULL）
      auto initial_val = aht_.GenerateInitialAggregateValue();
      std::vector<Value> values;
      // 添加 GROUP BY 字段值
      for (const auto& group_by_expr : plan_->GetGroupBys()) {
        (void)group_by_expr;
      }

      // Add aggregate values
      for (const auto& agg_val : initial_val.aggregates_) {
        values.push_back(agg_val);
      }

      *tuple = Tuple{values, &GetOutputSchema()};
      *rid = RID{}; // 聚合结果无实际RID，赋空值

      ++aht_iterator_;
      return true;
    }
    return false;
  }
  // 获取当前聚合结果
  auto agg_key = aht_iterator_.Key();
  auto agg_val = aht_iterator_.Val();

  std::vector<Value> values;

  // 添加 GROUP BY 字段值
  for (const auto& group_by_val : agg_key.group_bys_) {
    values.push_back(group_by_val);
  }

  // 添加聚合值
  for (const auto& agg_val_item : agg_val.aggregates_) {
    values.push_back(agg_val_item);
  }

  *tuple = Tuple{values, &GetOutputSchema()};
  *rid = RID{}; // 聚合结果无实际RID，赋空值

  ++aht_iterator_;
  return true;
 }
```

### NestedLoopJoin 

> src/execution/nested_loop_join_executor.cpp

Join 的本质是将两个或者多个表中的数据按照某种条件关联起来，形成一个新的结果集，基本思想是找到满足连接条件的元组对，然后将这些元组组合成新的元组。连接操作有多种类型：

- 内连接（INNER JOIN）：只返回满足连接条件的元组对
- 左外连接（LEFT JOIN）：返回左表的所有元组，如果右表没有匹配的元组，则右表部分用NULL值填充
- 右外连接（RIGHT JOIN）：返回右表的所有元组，如果右表没有匹配的元组，则右表部分用NULL值填充
- 全外连接（FULL OUTER JOIN）返回两个表的所有元组，不匹配的部分用NULL填充

NestedLoopJoin 是最基础的连接算子，其原理简单直观：**对左表的每个元组，遍历右表的所有元组，检查连接条件是否满足**。虽然算子的时间复杂度为O(m*n)，但在小表连接、缺乏合适索引或连接条件比较复杂的情况下比较适用。

在 NestedLoopJoin  的实现中，我们只需要实现内连接和左外连接即可，右外连接和全外连接可以通过左外连接和输入顺序调整来间接实现（）。NestedLoopJoin 的伪代码如下：

```cpp
for outer_tuple in outer_table:
    for inner_tuple in inner_table:
        if inner_tuple matched outer_tuple:
            emit
```

相应的，在 NestedLoopJoinExecutor 类中声明如下私有成员遍变量：

```cpp
/** The NestedLoopJoin plan node to be executed. */
const NestedLoopJoinPlanNode *plan_;
/** left child executor */
std::unique_ptr<AbstractExecutor> left_executor_;
/** right child executor */
std::unique_ptr<AbstractExecutor> right_executor_;
/** current left tuple */
Tuple left_tuple_{};
/** current left rid */
RID left_rid_{};
/** whether left tuple is available */
bool left_tuple_available_{false};
/** whether current left tuple has been matched */
bool left_matched_{false};
```

对照着伪代码，实现流程如下：

1. `left_executor_` 产生左表元组 -> 存储到 `left_tuple_`
2. 对于这个 `left_tuple_`，`right_executor_` 遍历所有右表元组
3. 如果找到匹配，设置 `left_matched_ = true`
4. 处理完所有右表元组后，检查 `left_matched_` 状态
5. 如果是 `LEFT` JOIN 且 `left_matched_ = false`，输出左表元组 + NULL
6. 移动到下一个左表元组（仅 `left_tuple_available_ = true` 时才可以移动），重置 `left_matched_ = fasle`

NestedLoopJoinExecutor 接收两个子执行器分别产出左、右测输入，并在 Next 中以双层循环驱动连接，。初始化阶段同时调用左右子执行器的Init，随后拉取首个左侧元组并清空匹配状态。

Next 阶段在外层”左元组可用“循环内，对应内层通过右侧子执行器拉去右元组并即时评估连接谓词，若为真则拼接输出；当右侧扫描完毕仍无匹配且为左外连接，则输出左侧元组配合右侧列的NULL值。每处理完一个左元组，重置右侧子执行器并推进到下一个左元组，直至左侧耗尽，实现如下：

```cpp
auto NestedLoopJoinExecutor::Next(Tuple *tuple, RID *rid) -> bool { 
  while (left_tuple_available_) {
    Tuple right_tuple{};
    RID right_rid{};

    // Try to get next right tuple
    while (right_executor_->Next(&right_tuple, &right_rid)) {
      // 检查是否匹配
      bool join_condition = true;
      if (plan_->Predicate() != nullptr) {
        auto value = plan_->Predicate()->EvaluateJoin(&left_tuple_, left_executor_->GetOutputSchema(), 
                                                      &right_tuple, right_executor_->GetOutputSchema());
        join_condition = !value.IsNull() && value.GetAs<bool>();
      }

      if (join_condition) {
        left_matched_ = true;

        // 拼接左表和右表元组的值
        std::vector<Value> values;
        for (uint32_t i = 0; i < left_executor_->GetOutputSchema().GetColumnCount(); i++) {
          values.emplace_back(left_tuple_.GetValue(&left_executor_->GetOutputSchema(), i));
        }
        for (uint32_t i = 0; i < right_executor_->GetOutputSchema().GetColumnCount(); i++) {
          values.emplace_back(right_tuple.GetValue(&right_executor_->GetOutputSchema(), i));
        }

        *tuple = Tuple(values, &GetOutputSchema());
        *rid = RID{};
        return true;
      }
    }

    // 处理左外连接，当前左表未找到匹配的右表元组
    if (plan_->GetJoinType() == JoinType::LEFT && !left_matched_) {
      std::vector<Value> values;
      for (uint32_t i = 0; i < left_executor_->GetOutputSchema().GetColumnCount(); i++) {
        values.emplace_back(left_tuple_.GetValue(&left_executor_->GetOutputSchema(), i));
      }
      // 右表无匹配，用NULL填充
      for (uint32_t i = 0; i < right_executor_->GetOutputSchema().GetColumnCount(); i++) {
        auto &column = right_executor_->GetOutputSchema().GetColumn(i);
        values.emplace_back(ValueFactory::GetNullValueByType(column.GetType()));
      }

      *tuple = Tuple(values, &GetOutputSchema());
      *rid = RID{};

      // 获取下一个左表元组
      left_tuple_available_ = left_executor_->Next(&left_tuple_, &left_rid_);
      left_matched_ = false;
      right_executor_->Init(); // 重置右表执行器,下次从右表开头迭代

      return true;
    }

    // 获取下一个左表元组
    left_tuple_available_ = left_executor_->Next(&left_tuple_, &left_rid_);
    left_matched_ = false;
    right_executor_->Init(); // 重置右表执行器,下次从右表开头迭代
  }

  return false;
 }
```

实现过程中需要注意以下几点：

1. 谓词求值的复杂性体现在对多列表达式与NULL语义的支持，若直接将表达式值视为 bool 而忽略 NULL，将漏匹配。通过 Evaluation 在左右两侧模式之上评估表达式，并在落地时以 `!value.IsNull() && value.GetAs<bool>` 统一为 bool，避免了三值逻辑导致的误判。

2. 左外连接的语义落地依赖于细粒度的匹配状态管理：left_matched_ 在首次匹配时置为真，右侧耗尽后检查其值，若仍为 false则按模式动态构造右侧列对应类型的NULl并输出；这要求从右侧输出模式中逐列提取类型，并用`ValueFactory::GetNullValueBytypes` 构造占位。
3. 处理完一个左元组后，需要**重置右侧执行器**。

可能到这里你还是对 Join 有些难以理解，举个例子来将代码和真实操作关系起来：

假设现在有两个表：

![image-20250811153818988](/images/$%7Bfiilename%7D/image-20250811153818988.png)

现在想要找出每个学生及其选课信息。SQL 如下：

```sql
SELECT s.name, s.major, e.course_id, e.grade
FROM students s
LEFT JOIN enrollments e ON s.student_id = e.student_id;
```

NestedLoopJoin 的流程如下：

**第一步**：处理 student_id = 1001

- 外层循环：获取张三的信息（name=”张三“，major=”计算机“）
- 内层循环：扫描选课表，寻找 student_id = 1001 的元组 
  - 找到元组{CS101， 85} and {CS102, 92}
- 拼接左表和右表元组的值，如下图

![image-20250811154447013](/images/$%7Bfiilename%7D/image-20250811154447013.png)

**第二步**：处理  student_id = 1002

- 外层循环：获取李四的信息（name=”李四“，major=”数学“）
- 内层循环：扫描选课表，寻找 student_id = 1002 的元组 
  - 找到元组{MATH101， 78} 
- 拼接左表和右表元组的值，如下图

![image-20250811154814423](/images/$%7Bfiilename%7D/image-20250811154814423.png)

**第三步**：处理 处理  student_id = 1003，流程如上，最后拼接结果值。

如果找的 student_id  在右表不存在呢，假如右表不存在student_id = 1003这个记录对应的值呢？左表的所有行保留，和右表连接（右表无匹配补NULL），结果如下：

![image-20250811155239076](/images/$%7Bfiilename%7D/image-20250811155239076.png)

### NestedIndexJoin

> src/execution/nested_index_join_executor.cpp
>
> [参考](https://zhuanlan.zhihu.com/p/587566135)：在进行 equi-join 时，如果发现 JOIN ON 右边的字段上建了 index，则 Optimizer 会将 NestedLoopJoin 优化为 NestedIndexJoin。具体实现和 NestedLoopJoin 差不多，只是在尝试匹配右表 tuple 时，会拿 join key 去 B+Tree Index 里进行查询。如果查询到结果，就拿着查到的 RID 去右表获取 tuple 然后装配成结果输出。

NestedIndexJoin 的简要概括如上述引用所述，其核心思想是以外表为驱动，逐行从外表产出元组，用该行通过连接键表达式计算出索引值，直接在内表的索引上做点查找获取候选RID，再回表取出内表元组并与外表元组拼接输出；若为左外连接且无匹配，则以NULL补齐到内表列后输出外表元组。与纯 NLJ 相比，区别只在”匹配右表元组“的环节，不再循环遍历右表，而是借助 ScanKey 在索引中把可能匹配缩小为确切匹配，将内层循环的代价从O(右表的大小)降到O(log N + k)。

Init 阶段完成子执行器启动、外表首行拉去与首轮索引探测：对第一条外表元组，通过计划节点的 KeyPredicate 在外表输出模式上求值得到索引键，按内表索引的 key_schema_ 构造键元组并调用 ScanKey 取回所有匹配的 RID，缓存至 inner_rids_ 供 next 使用：

```cpp
void NestedIndexJoinExecutor::Init() { 
  child_executor_->Init();

  // 获取第一个外表元组
  outer_tuple_available_ = child_executor_->Next(&outer_tuple_, &outer_rid);
  outer_matched_ = false;
  inner_rids_.clear();
  inner_match_idx_ = 0;

  if (outer_tuple_available_) {
    // 获取索引信息
    auto catalog = exec_ctx_->GetCatalog();
    auto index_info = catalog->GetIndex(plan_->GetIndexOid());

    // 从外表元组中提取索引键
    auto key_value = plan_->KeyPredicate()->Evaluate(&outer_tuple_, child_executor_->GetOutputSchema());

    std::vector<Value> key_values{key_value};
    Tuple index_key{key_values, &index_info->key_schema_};

    // 利用索引查找所有匹配该键的内表元组的RID
    index_info->index_->ScanKey(index_key, &inner_rids_, exec_ctx_->GetTransaction());
  }
 }
```

Next 阶段先吐完当前外表行的所有匹配，然后再推进下一外表行。若 inner_rids_ 仍有未消费匹配，则逐个回表取元组，跳过已删除记录，设置匹配标志并与外表行拼接输出；当匹配耗尽且未LEFT JOIN且尚未匹配，则构造内表列对应类型的NULL与外表行拼接输出；否则推进到下一条外表行并立即为其做新一轮索引探测：

```cpp
auto NestedIndexJoinExecutor::Next(Tuple *tuple, RID *rid) -> bool { 
  auto catalog = exec_ctx_->GetCatalog();
  auto inner_table_info = catalog->GetTable(plan_->GetInnerTableOid());
  auto index_info = catalog->GetIndex(plan_->GetIndexOid());

  while (outer_tuple_available_) {
    // 当前外表元组未匹配，且有内表匹配项
    if (inner_match_idx_ < inner_rids_.size()) {
      // 获取当前匹配的内表元组
      auto inner_rid = inner_rids_[inner_match_idx_++];
      auto [tuple_meta, inner_tuple] = inner_table_info->table_->GetTuple(inner_rid);
      if (tuple_meta.is_deleted_) {
        continue;
      }

      outer_matched_ = true;

      // 拼接外表和内表的字段
      std::vector<Value> values;
      for (uint32_t i = 0; i < child_executor_->GetOutputSchema().GetColumnCount(); i++) {
        values.push_back(outer_tuple_.GetValue(&child_executor_->GetOutputSchema(), i));
      }
      for (uint32_t i = 0; i < plan_->InnerTableSchema().GetColumnCount(); i++) {
        values.push_back(inner_tuple.GetValue(&plan_->InnerTableSchema(), i));
      }

      *tuple = Tuple{values, &GetOutputSchema()};
      *rid = RID{};

      return true;
    }

    // 处理左外连接，当前外层元组未找到匹配的内表元组
    if (plan_->GetJoinType() == JoinType::LEFT && !outer_matched_) {
      std::vector<Value> values;
      for (uint32_t i = 0; i < child_executor_->GetOutputSchema().GetColumnCount(); i++) {
        values.push_back(outer_tuple_.GetValue(&child_executor_->GetOutputSchema(), i));
      }
      for (uint32_t i = 0; i < plan_->InnerTableSchema().GetColumnCount(); i++) {
        // 内表无匹配，用NULL填充
        auto& column = plan_->InnerTableSchema().GetColumn(i);
        values.push_back(ValueFactory::GetNullValueByType(column.GetType()));
      }
      *tuple = Tuple{values, &GetOutputSchema()};
      *rid = RID{};

      // 准备处理下一个外表元组
      outer_tuple_available_ = child_executor_->Next(&outer_tuple_, &outer_rid);
      outer_matched_ = false;
      inner_rids_.clear();
      inner_match_idx_= 0;

      return true;
    }

    // 处理完当前外层元组的所有匹配项，准备处理下一个外表元组
    outer_tuple_available_ = child_executor_->Next(&outer_tuple_, &outer_rid);
    if (!outer_tuple_available_) {
      break;
    }

    outer_matched_ = false;
    inner_rids_.clear();
    inner_match_idx_= 0;

    // 从新外层元组中提取索引key
    auto key_value = plan_->KeyPredicate()->Evaluate(&outer_tuple_, child_executor_->GetOutputSchema());
    std::vector<Value> key_values{key_value};
    Tuple index_key{key_values, &index_info->key_schema_};

    // 利用索引查找所有匹配该键的内表元组的RID
    index_info->index_->ScanKey(index_key, &inner_rids_, exec_ctx_->GetTransaction());
  }

  // 没有更多的外表元组
  return false;
 }
```

同样，举个例子形象说明。

假设有以下两个表：

![image-20250811170818165](/images/$%7Bfiilename%7D/image-20250811170818165.png)

假设成绩表的student_id字段上有索引。SQL如下：

```sql
SELECT students.id, students.name, scores.score
FROM students
JOIN scores ON students.id = scores.student_id;
```

第一步：从学生表（外表）取出第一行，比如 {1, ”张三“};

第二步：用 id = 1 去成绩表（内表）的 student_id 索引里查找所有 student_id = 1 的行，查找两行 {90， 88}

第三步：组合成结果：{1，”张三“，90} {1，“张三”，88}

第四步：换下一行，{2，“李四”}

第五步：同理，直到 students 表遍历完

> NLJ 和 NIJ 的区别

1. NLJ 实现简单，适用于任何连接条件；但性能差，尤其当内表很大时效率极低
2. 利用索引查询内表匹配行的速度很快，效率高于NLJ；但如果外表很大时，每次都有索引键，索引查找的开销也会积累。
