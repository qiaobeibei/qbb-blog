---
title: CMU-15445(10)——PROJECT#3-Query Execution-Task#3
date: 2025-07-24 14:12:31
categories:
- CMU-15445
tags: 
- Fall 2024
typora-root-url: ./..
---

## Task #3 - HashJoin Executor and Optimization

> src/execution/hash_join_executor.cpp

### HashJoin 

从 Task#2 的实现可知，NLJ 通过双重循环暴力枚举两表所有组合，效率最低；NIJ 则在外表循环的基础上，利用内表的索引加速查找匹配行，适合内表有索引的情况。而 HashJoin 首先将一张表的连接键构建为哈希表，再用另一张表的连接键进行高效查找，适合大表等值连接，通常在无索引或多对多等值连接时性能最优。从效率来排 HashJoin  > NIJ > NLJ，**但 HashJoin  只能用在等值连接（equi-join）中，因为哈希表只能做键是否相等的查找，范围条件（>或者<）或带计算的条件无法直接用哈希匹配**。

> **等值比较**是指，连接条件里两边都是“某个表的一列”，并且用等号比较。比如 a.id = b.student_id。它不是“列=常量（a.id = 10）”，也不是"列和计算"（比如 a.id + 1= b.student_id），更不是大小比较 (a.id > b.student_id)。只能是左表的一列等于右表的一列。

简单来说，hashJoin 就是对每个元组根据分组列和聚合列分别构造出一个数组作为哈希表的Key和Value， 然后针对相同的Key对Value做聚合。

在实现 HashJoinExecutor 之前，先解释一下什么是哈希键的自定义比较和哈希函数。哈希键的自定义比较函数和哈希函数主要是为了让复合数据结构能够作为哈希表的键而必须实现的操作。

**自定义比较函数**主要用来判断什么时候两个键才能被视为相等，比如对于包含多个列的对象，只有当所有列都相等时，整个键才相等，并且要确保包含 NULL 的键不会与任何键相等；在下述实现中，`std::vector<Value>` 用于承载一列或多列的连接键，重载的 `operator==` 逐列使用 `Value::CompareEquals` 比较相等，只有比较结果为 CmpBool::CmpTrue 时判定为相等；一旦列数不一致或任一列比较结果非真（包括 NULL 导致的 CmpNull），就视为不相等，确保让复合键的相等关系和 SQL 等值连接的语义对齐：包含 NULL 的连接键不会彼此相等，因此不会参与等值匹配。示例如下：

```cpp
struct HashJoinKey {
  std::vector<Value> keys_;

  explicit HashJoinKey(std::vector<Value> key) : keys_(std::move(key)) {}

  auto operator==(const HashJoinKey& other) const->bool {
    if (keys_.size() != other.keys_.size()) {
      return false;
    }
    for (size_t i = 0; i < keys_.size(); ++i) {
      if (keys_[i].CompareEquals(other.keys_[i]) != CmpBool::CmpTrue ){
        return false;
      }
    }
    return true;
  }
};
```

而**自定义哈希函数**主要为了将复合键转换为哈希值，它需要遍历键的每一个分量，对每个分裂计算哈希，然后将这些哈希值组成一个整体的哈希值。下述哈希函数的实现中，对每个 Value 调用 HashUtil::HashValue 得到列哈希，再通过 HashUtil::CombineHashes 顺序合并为整体哈希值。该组合方式既保留列的顺序信息，又能在不同列数、不同取值时产生分布较好的散列，降低碰撞概率。实现如下：

```cpp
template <>
struct hash<bustub::HashJoinKey> {
  auto operator()(const bustub::HashJoinKey& key) const-> size_t {
    size_t hash_val = 0;
    for (const auto& val : key.keys_) {
      hash_val = bustub::HashUtil::CombineHashes(hash_val, bustub::HashUtil::HashValue(&val));
    }
    return hash_val;
  }
};
```

`HashJoinKey` 和 `hash<bustub::HashJoinKey>` 在构建端能把相等的多列连接键归并到同一哈希桶并聚集对应左侧（外表）元组列表，在探测端按相同规则计算键后能以均摊O(1)的复杂度查到候选匹配，同时遵循SQL等值连接关于NULL的语义规则。

举个例子说明：假设有一个 join 查询：`SELECT * FROM A JOIN B ON A.id = B.id AND A.name = B.name`, 连接键包含两列：id 和 name

- 表 A 的数据：{1, "A"}、{2, "B"}、{3, NULL}
- 表 B 的数据：{1, "A"}、{2, ”B"}、{3, "C"}

在 HashJoin 的**构建阶段**， HashJoinKey 会为表 A 的每一行创建复合键：

- 键1: [id = 1, name = "A"]
- 键2: [id = 2, name = "B"]
- 键3: [id = 3, name = NULL]

自定义的比较函数确保：

- 键1和键1相等（所有列都相等）
- 键1和键2不相等（name、id列均不同）
- 键3和任何键都不相等（包含NULL）

自定义哈希函数为每个键计算哈希值，相同的键产生相同的哈希值。

在  HashJoin 的**探测阶段**，对于表 B 的每一行，也创建相同的复合键，然后用这个键在哈希表中查找匹配的左侧元组，只有真正匹配的行才会被连接（比如表A和表B的{1, "A"}行），而包含 NULL 的行不会被错误的匹配。

------

上面提到了 HashJoin 的**构建阶段**和**探测阶段**，什么是构建和探测呢？

HashJoin 的构建阶段发生在 Init() 中，目标是建立哈希索引。我们的实现中是从左子执行器（一般选择较小的表作为构建端）逐行读取元组，对每个元组按左连接键表达式求值得到连接键，构造 HashJoinKey 对象，并将该键插入到 hash_table 中。如果哈希表中已存在相同键，则将新的左侧元组追加到对应键的 `std::vector<Tuple>` 中；如果是新键，则创建新的 vector 并插入该元组。构建完之后，哈希表就能形成从**连接键**到**所有具有该键值的左表元组列表**的映射关系。

```cpp
void HashJoinExecutor::Init() { 
  left_executor_->Init();
  right_executor_->Init();

  // 构建阶段：从左表构建hash表（一般从较小的表开始）
  hash_table_.clear();

  Tuple left_tuple{};
  RID left_rid{};

  while (left_executor_->Next(&left_tuple, &left_rid)) {
    // 计算左表元组的 hash key
    std::vector<Value> left_key_values;
    for (const auto& expr : plan_->LeftJoinKeyExpressions()) {
      left_key_values.emplace_back(expr->Evaluate(&left_tuple, left_executor_->GetOutputSchema()));
    }
    // 构建 hash key 并将其与左表元组关联存储在哈希表中
    HashJoinKey left_key{left_key_values};
    hash_table_[left_key].emplace_back(std::move(left_tuple));
  }
  // 初始化探测阶段
  current_matches_.clear();
  current_match_idx_=0;
  right_tuple_available_=false;
  left_matched_=false;
 }
```

探测阶段发生在 Next() 中，目标是利用已构建的哈希表快速找到匹配。我们的实现中是从右子执行器逐行读取元组，对每个元组按右连接键表达式求值得到连接键，构造 HashJoinKey 对象，然后用该键在哈希表中查找。如果找到匹配的键，说明存在连接键值相等的左侧元组，此时将匹配的左侧元组列表缓存到 current_matches_ 中，并逐行产出连接结果；若为左外连接，还需要处理右表元组未匹配的清空，即输出右表元组与左表空值拼接的结果。

```cpp
auto HashJoinExecutor::Next(Tuple *tuple, RID *rid) -> bool { 
  while (true) {
    // 若当前右表元组有未处理的左表匹配，优先返回这些匹配结果
    if (current_match_idx_ < current_matches_.size()) {
      // 获取当前要处理的左表元组
      const auto& left_tuple = current_matches_[current_match_idx_++];

      // 拼接左表和右表值
      std::vector<Value> values;
      for (uint32_t i = 0; i < left_executor_->GetOutputSchema().GetColumnCount(); i++) {
        values.emplace_back(left_tuple.GetValue(&left_executor_->GetOutputSchema(), i));
      }
      for (uint32_t i = 0; i < right_executor_->GetOutputSchema().GetColumnCount(); i++) {
        values.emplace_back(current_right_tuple_.GetValue(&right_executor_->GetOutputSchema(), i));
      }

      *tuple = Tuple{values, &GetOutputSchema()};
      *rid = RID{};
      return true;
    }

    // 左外连接，右表元组未找到匹配的左表元组
    if (plan_->GetJoinType() == JoinType::LEFT && right_tuple_available_ && !left_matched_) {
      std::vector<Value> values;
      // 添加右表元组的所有字段值
      for (uint32_t i = 0; i < right_executor_->GetOutputSchema().GetColumnCount(); i++) {
        values.emplace_back(current_right_tuple_.GetValue(&right_executor_->GetOutputSchema(), i));
      }
      // 左表无匹配，用NULL填充左表字段
      for (uint32_t i = 0; i < left_executor_->GetOutputSchema().GetColumnCount(); i++) {
        auto& column = left_executor_->GetOutputSchema().GetColumn(i);
        values.emplace_back(ValueFactory::GetNullValueByType(column.GetType()));
      }

      *tuple = Tuple{values, &GetOutputSchema()};
      *rid = RID{};
      
      // 准备处理下一个右表元组
      right_tuple_available_ = right_executor_->Next(&current_right_tuple_, &current_right_rid_);
      left_matched_ = false;
      current_match_idx_=0;
      current_matches_.clear();

      return true;
    }

    // 获取下一个右表元组
    if (!right_tuple_available_) {
      right_tuple_available_ = right_executor_->Next(&current_right_tuple_, &current_right_rid_);
    }
    // 若右表已无更多元组，则结束
    if (!right_tuple_available_) {
      return false;
    }

    // 从当前右表元组中计算哈希键
    std::vector<Value> right_key_values;
    for (const auto& expr : plan_->RightJoinKeyExpressions()) {
      right_key_values.emplace_back(expr->Evaluate(&current_right_tuple_, right_executor_->GetOutputSchema()));
    }

    // 构造右表哈希键对象
    HashJoinKey right_key{right_key_values};

    // 探测哈希表，查找与右表连接键匹配的左表元组
    auto it = hash_table_.find(right_key);
    if (it != hash_table_.end()) {
      current_matches_ = it->second;
      current_match_idx_ = 0; 
      left_matched_ = true;
    } else {
      current_matches_.clear();
      current_match_idx_ = 0;
      left_matched_ = false;
    }

    right_tuple_available_ = false;
  }
}
```

可以看到，HashJoin 中左外连接情况下，右表被当作保留测处理，但是在TASK2中实现的NLJ、NIJ 连接算子中，左表（外表）才是保留侧，这是为什么？在不同的连接算子中，哪个表是“保留侧”？

> 这是因为在 hashJoin 中，一般用较小的表作为构建端，我们在实现的时候选择了左表作为构建端，然后用右表进行探测，对右表的每一行在左表形成的构建表中寻找匹配，当右表的某行在左表没有匹配时，输出右表的值和左表的NULL，因此右表作为保留测。但当构建端是右表时，保留测变为了左表。这是个动态的过程。

### Optimizing NestedLoopJoin to HashJoin

> src/optimizer/nlj_as_hash_join.cpp

上面提到了 NLJ 相比 NIJ和hashJoin，NLJ 的效率太差。本小节尝试对 NLJ 优化为 HashJoin（自动将满足固定条件的NLJ转换为HashJoin）。大概思路是递归遍历计划树，识别处等值连接（即连接谓词为等值比较，且左右两侧均为列值表达式），并将其替换为HashJoinPlanNode，简单来说就是：**将连接谓词中的“列 = 列”条件抽取成左右连接键列表，左侧构建哈希表、右侧用键探测**。时间复杂度从(m*n)将为O(m+n)。

> 列= 列的等值比较是指，连接条件里两边都是“某个表的一列”，并且用等号比较。比如 a.id = b.student_id。它不是“列=常量（a.id = 10）”，也不是"列和计算"（比如 a.id + 1= b.student_id），更不是大小比较 (a.id > b.student_id)。只能是左表的一列等于右表的一列。

> 为什么优化为 HashJoin 可以提升连接效率？
>
> HashJoin 的核心思想是“先构建，再探测”。构建阶段，扫描左表，对每一行计算连接键的哈希值，并将该行插入到哈希表对应的桶。在探测阶段，扫描右表，对每一行计算连接键的哈希值，在哈希表中查找匹配的行，从而实现快速等值连接。
>
> 根本原因还是在于，构建阶段只需要扫描一次左表，探测阶段对每个右表行都能在均摊O(1)时间内找到所有匹配的左表行，避免了嵌套循环中重复扫描的问题。而且在左表相对较小时，完全可以将哈希表放在内存，进一步减少IO开销。

触发条件：

- 必须是 equi-join，且两侧键分别来自左右输入各一列；并允许按 AND 连接的多列键
- **仅对 Inner Join 触发规则**，避免外连接语义处理复杂度（外连接涉及保留侧与NULL生成语义，改写很复杂）；
- 若谓词含非等值或混合条件，当前实现选择不该写（保持NLJ）

`Optimizer::OptimizeNLJAsHashJoin`是我们实现的核心，通过自底向上的方式遍历查询计划树，对于每个计划节点，首先递归优化其子节点，确保子节点已经是最优的。然后检查当前节点是否为NLJ 并且是 INNER JOIN，尝试把 NLJ 的谓词拆解出“等值连接键”，成功就使用新的HashJoinPlanNode来替换原有的NLJ，失败就不变：

```cpp
auto Optimizer::OptimizeNLJAsHashJoin(const AbstractPlanNodeRef &plan) -> AbstractPlanNodeRef {
  // 递归优化子节点
  std::vector<AbstractPlanNodeRef> children;
  for (const auto& child : plan->GetChildren()) {
    children.emplace_back(OptimizeNLJAsHashJoin(child));
  }
  auto optimized_plan = plan->CloneWithChildren(std::move(children));

  // 检查是否为 NLJ
  if (optimized_plan->GetType() == PlanType::NestedLoopJoin) {
    const auto& nlj_plan = dynamic_cast<const NestedLoopJoinPlanNode&>(*optimized_plan);

    // 只优化内连接，因为哈希连接对外连接的实现更复杂
    if (nlj_plan.GetJoinType() == JoinType::INNER) {
      std::vector<AbstractExpressionRef> left_keys;
      std::vector<AbstractExpressionRef> right_keys;

      // 尝试从谓词中提取等值连接键
      if (ExtractJoinKeys(nlj_plan.Predicate(), left_keys, right_keys) && !left_keys.empty()) {
        return std::make_shared<HashJoinPlanNode>(nlj_plan.output_schema_, nlj_plan.GetLeftPlan(),
                                                    nlj_plan.GetRightPlan(), std::move(left_keys),
                                                    std::move(right_keys), nlj_plan.GetJoinType());
      }
    }
  }
  return optimized_plan;
}
```

从谓词中提取等值连接键的工作由 IsEquiComparison、IsColumnExpression 和 ExtractJoinKeys 配合完成。IsEquiComparison 只认**“=”**这种等值比较；IsColumnExpression  确保两边都是列；ExtractJoinKeys  负责把复杂谓词拆开（目前只支持AND），并保证一边来自左表，另一边来自右表，这样才能将哈希键的左键/右键一一对应起来。核心逻辑是，如果是AND，就拆两边；如果是“列=列”，就按左右输入的 tuple_idx 把它放进 right_keys  和 left_keys 。实现如下：

> 列绑定里用 tuple_idx 区分左右子：左子为0，右子为1。提取键时需要把任意（左，右）或（右，左）的等值对，规整为：left_keys 对应左子、right_keys 对应右子。

```cpp
auto IsEquiComparison(const AbstractExpressionRef &expr) -> bool {
  const auto *comp_expr = dynamic_cast<const ComparisonExpression *>(expr.get());
  return comp_expr != nullptr && comp_expr->comp_type_ == ComparisonType::Equal;
}

auto IsColumnExpression(const AbstractExpressionRef& expr) -> bool {
  return dynamic_cast<const ColumnValueExpression *>(expr.get()) != nullptr;
}

auto ExtractJoinKeys(const AbstractExpressionRef& predicate, std::vector<AbstractExpressionRef>& left_keys,std::vector<AbstractExpressionRef>& right_keys) -> bool {
  if (predicate == nullptr) {
    return false;
  }

  // ANd 递归拆分
  if (const LogicExpression* logic_expr = dynamic_cast<const LogicExpression*>(predicate.get()); logic_expr != nullptr) {
    if (logic_expr->logic_type_ == LogicType::And) {
      // 递归处理AND的两个子表达式
      return ExtractJoinKeys(logic_expr->GetChildAt(0), left_keys, right_keys) &&
            ExtractJoinKeys(logic_expr->GetChildAt(1), left_keys, right_keys);
    }
    return false;
  }

  // 识别 列=列，且一边来自左表一边来自右表
  if (IsEquiComparison(predicate)) {
    const auto& comp_expr = dynamic_cast<const ComparisonExpression&>(*predicate);
    auto left_expr = comp_expr.GetChildAt(0);
    auto right_expr = comp_expr.GetChildAt(1);

    // 检查两边都是列表达式
    if (IsColumnExpression(left_expr) && IsColumnExpression(right_expr)) {
      const auto& left_col = dynamic_cast<const ColumnValueExpression&>(*left_expr);
      const auto& right_col = dynamic_cast<const ColumnValueExpression&>(*right_expr);

      // 确保一个来自左表（tuple_idex=0），一个来自右表（tuple_idex=1）
      if (left_col.GetTupleIdx() == 0 && right_col.GetTupleIdx() ==1) {
        left_keys.emplace_back(left_expr);
        right_keys.emplace_back(right_expr);
        return true;
      } else if (left_col.GetTupleIdx() == 1 && right_col.GetTupleIdx() ==0) {
        left_keys.emplace_back(right_expr);
        right_keys.emplace_back(left_expr);
        return true;
      }
    }
  }
  return false;
}
```

把这些拼起来看，整体路径就是：遍历计划树，遇到NLJ且是INNER，就去它的谓词里找“列=列”的等值条件，还要确保确实是跨左右输入各取一列；如果找到了，就把这些列表达式成对放进 left_keys/right_keys，并用它们新建一个 HashJoinPLanNode。执行层拿到这个哈希计划节点后，在Init() 先用左输入构建哈希表，再用右输入逐行探测。

> 为什么只处理INNER JOIN，而不是LEFT/RIGHT OUTER？
>
> 因为外连接有“保留测必须产出”的规则，哈希连接需要额外跟踪哪些行没匹配到并补NULL，过程很复杂，目前我只实现了内连接转换。

如果还没有理解，可以看下面这个例子，将原理和实践联系起来：

假如有两个表：

- 表A：id，name，class_id
- 表B：student_id，class_id，score

执行以下SQL：

```sql
SELECT *
FROM A JOIN B ON A.id = B.student_id AND A.age = B.age;
```

**(1) 优化器首先生成一个 NLJ 查询计划**：

```
NestedLoopJoin {
	type: INNER,
	predicate: (A.id = B.student_id AND A.class_id = B.class_id),
	left: Scan(A),
	right: Scan(B)
}
```

**(2) 优化器识别 NLJ 节点**：当优化器 OptimizeNLJAsHashJoin 遍历到这个节点时，发现：这个一个 NestedLoopJoin 节点，连接类型是Inner并且满足优化条件。

**(3) 提取等值连接键**：优化器调用 ExtractJoinKeys 函数分析谓词 `(A.id = B.student_id AND A.class_id= B.class_id)`:

**处理第一个条件** `A.id = B.student_id`：

- 左边 A.id 来自左表 A，tuple_idx = 0
- 右表 B.student_id 来自右表 B，tuple_idx = 1
- 并且是一个有效的等值连接键

输出 `left_keys = [A.id], right_keys = [B.student_id]`

**处理第二各条件** `A.class_id= B.class_id`：

- 左边 A.age来自左表 A，tuple_idx = 0
- 右表 B.age来自右表 B，tuple_idx = 1
- 并且是一个有效的等值连接键

输出 `left_keys = [A.id, A.class_id], right_keys = [B.student_id, B.class_id]`

**(4) 创建 HashJoin 计划**

由于成功提取出了等值连接键，优化器创建新的 HashJoinPlanNode：

```
HashJoinPlanNode {
	type: INNER,
	left_keys = [A.id, A.class_id],
	right_keys = [B.student_id, B.class_id]
	left: Scan(A),
	right: Scan(B)
}
```

（5）执行

```
// 原本的 NLJ 执行方式
for each a in A:
	for each b in B:
		if a.id == b.student_id AND a.class_id == b.class_id:
			output(a, b)
			
// 新的 HashJoin 执行方式
// 构建阶段：扫描表 A
for each a in A:
	key = hash(a,id, a.class_id)
	hash_table[key].emplace_back(a)
	
// 探测阶段：扫描表 B
for each b in B:
	key = hash(b.student_id, b.class_id)
	matches = hash_table[key]
	for each a in matches:
		if a.id == b.student_id AND a.class_id == b.class_id:
			output(a, b)
```

复杂度从O(m*n)降到O(m+n)。
