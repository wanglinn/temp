# Oracle 各种tree 

Oracle 优化器内部涉及多种树形数据结构，除了 Parse Tree 和 Plan Tree，还有以下几种重要的 Tree：

---

## 一、Oracle 优化器中的各类 Tree

### 1. **Parse Tree（解析树）**
- 由 SQL Parser 生成，反映 SQL 的语法结构
- 节点类型：SELECT_STATEMENT、FROM_CLAUSE、WHERE_CLAUSE 等
- 是语法层面的抽象，尚未涉及物理执行

### 2. **Query Block Tree（查询块树）**
- Parse Tree 经过语义分析后，被拆分为多个 **Query Block（查询块）**
- 每个子查询、视图、内联视图对应一个独立的 Query Block
- 优化器自底向上逐个优化 Query Block

```
Query Block Tree 结构示例：
SELECT * FROM t1 WHERE c1 IN (SELECT c1 FROM t2)
├── QB1 (主查询块): SELECT * FROM t1 WHERE c1 IN ...
│   └── 包含子查询引用
└── QB2 (子查询块): SELECT c1 FROM t2
```

### 3. **Transformed Query Tree（转换后的查询树）**
- Query Transformation 阶段输出
- 经过 View Merging、Subquery Unnesting 等转换后的等价查询树
- 仍是逻辑层面的树，但比 Parse Tree 更利于优化

### 4. **Plan Tree / Execution Plan Tree（执行计划树）**
- Row Source Generation 阶段输出
- 由 **Row Source** 节点组成，采用 **火山模型（Iterator Model）**
- 每个节点实现 `open/next/close` 接口
- 是物理执行层面的树

### 5. **Join Tree（连接树）**
- 优化器在枚举连接顺序时生成的树
- 描述多表之间的连接关系
- 类型：
  - **Left-Deep Tree（左深树）**：Oracle 默认考虑
  - **Bushy Tree（茂密树）**：Oracle 通常不考虑（搜索空间太大）

```
Left-Deep Tree:          Bushy Tree:
      JOIN                 JOIN
     /    \               /    \
   JOIN    t3           JOIN   JOIN
  /    \               /   \   /   \
t1     t2             t1   t2 t3   t4
```

### 6. **DFO Tree（Data Flow Operation Tree）**
- 并行执行时使用
- 描述并行执行中的数据流操作
- 包含 PX Coordinator、PX Server Set、Table Queue 等节点

### 7. **Expression Tree（表达式树）**
- 表示 WHERE 子句、SELECT 列表中的表达式
- 例如：`a + b * c` 的表达式树：
```
    +
   / \
  a   *
     / \
    b   c
```

### 8. **Predicate Tree（谓词树）**
- 表示 WHERE/HAVING 子句中的谓词条件
- 用于谓词推导、谓词下推等优化

### 9. **Cost Tree（成本树）**
- 优化器内部评估时，为每个候选计划计算成本
- 形成一棵成本估算树，用于比较不同计划的优劣

---

## 二、完整关系图

```
SQL 文本
    ↓
[Parse Tree] ──→ 语法分析
    ↓
[Query Block Tree] ──→ 语义分析，拆分为查询块
    ↓
[Transformed Query Tree] ──→ Query Transformation（视图合并、子查询展开等）
    ↓
    ├── [Expression Tree] ──→ 表达式优化
    ├── [Predicate Tree] ──→ 谓词推导、下推
    ↓
[Join Tree] ──→ 连接顺序枚举（左深树/茂密树）
    ↓
[Plan Tree / Execution Plan Tree] ──→ Row Source Generation（火山模型）
    ↓
    ├── 串行执行
    └── [DFO Tree] ──→ 并行执行（PX Coordinator + PX Servers）
    ↓
执行
```

---

## 三、其他相关结构

| 结构名 | 说明 |
|--------|------|
| **AST（Abstract Syntax Tree）** | 通用术语，Parse Tree 就是一种 AST |
| **CBO Search Tree** | 优化器搜索候选计划时的树形结构 |
| **Bitmap Tree** | 位图索引运算时生成的树（Bitmap AND/OR） |
| **Partition Tree** | 分区表的分区层次结构 |
| **Index Tree** | B-Tree 索引的物理树结构 |

---

## 四、核心总结

Oracle 中从 SQL 到执行，主要涉及 **4 棵核心树**：

| 树 | 阶段 | 性质 |
|----|------|------|
| **Parse Tree** | Parse 阶段 | 语法树 |
| **Query Block Tree** | 语义分析 | 逻辑块树 |
| **Transformed Query Tree** | Query Transformation | 转换后的逻辑树 |
| **Plan Tree（Row Source Tree）** | Row Source Generation | 物理执行树 |

其中 **Plan Tree** 是最关键的，它直接决定了 SQL 的执行方式。而 **DFO Tree** 是并行执行时对 Plan Tree 的扩展。
