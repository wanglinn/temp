#1 分析 enterprisedb proc 模式 
https://www.enterprisedb.com/docs/epas/latest/application_programming/ecpgplus_guide/  -- 这个proc 模式实现了宿主变量可以不在声明块中声明。

# 问题 注册的宿主变量的信息中 包含了作用域（也就是函数名这样的信息） 对吧？

# 回复 

**是的，必须包含。**

如果不存作用域信息（比如只存变量名→类型的映射），预编译器根本无法区分不同函数里的同名局部变量，生成的绑定代码就会张冠李戴。

---

## 一、符号表结构对比

### 标准 ECPG / 达梦（扁平表，无作用域）

```c
struct variable {
    char *name;          /* 变量名 */
    enum ECPGttype type; /* 类型 */
    int  brace_level;    /* 可能只有这个，用于 DECLARE SECTION 内的简单嵌套 */
    struct variable *next;
};
```

所有在 `BEGIN/END DECLARE SECTION` 里声明的变量都挂在一个**全局链表**上。查找 `:var` 时就是 `strcmp(name)`，简单粗暴。

### Pro*C / EDB -C PROC 模式（分层符号表）

```c
struct scope {
    char *func_name;          /* 函数名，NULL 表示全局作用域 */
    int   block_depth;        /* 块嵌套深度（if/for/while 里的局部块） */
    struct variable *vars;    /* 该作用域内的变量链表 */
    struct scope *parent;     /* 指向外层作用域 */
    struct scope *next;       /* 同级作用域 */
};

struct variable {
    char *name;
    enum ECPGttype type;
    int   is_parameter;       /* 是否是函数形参 */
    int   line_number;        /* 声明行号，用于报错 */
    struct variable *next;
};
```

查找 `:var` 时，**从当前作用域向内层逐层查找**：

```c
struct variable *find_hostvar(const char *name, struct scope *current_scope)
{
    struct scope *s;
    for (s = current_scope; s; s = s->parent) {
        struct variable *v;
        for (v = s->vars; v; v = v->next) {
            if (strcmp(v->name, name) == 0)
                return v;
        }
    }
    return NULL;  /* 未找到 */
}
```

---

## 二、预编译器怎么知道"当前作用域"

在 `pgc.l` / `preproc.y` 中扫描 C 代码时，需要维护一个**简单的 C 语法状态机**：

| 事件 | 动作 |
|---|---|
| 遇到 `void func_name(` 或 `int func_name(` | 压入新作用域，`func_name` 作为作用域名 |
| 遇到 `{` | `block_depth++`，如果是函数体开头，标记这是函数作用域 |
| 遇到 `}` | `block_depth--`，如果归 0，弹出当前函数作用域 |
| 在作用域内遇到 `int x;` | 注册到**当前作用域**的变量表 |
| 遇到 `EXEC SQL ... :x` | 从**当前作用域**向内查找 `x` |

---

## 三、实际例子

```c
int g_id;                    /* 全局作用域：func=NULL */

void foo(int p_id)           /* 进入 foo 作用域 */
{                            /* block_depth=1 */
    int l_id;                /* 注册到 foo 作用域 */
    
    {                        /* block_depth=2（嵌套块） */
        int l_id;            /* 遮蔽外层 foo 的 l_id */
        EXEC SQL INTO :l_id; /* 绑定到 block_depth=2 的 l_id */
    }
    
    EXEC SQL INTO :l_id;     /* 绑定到 block_depth=1 的 l_id */
    EXEC SQL INTO :g_id;     /* foo 作用域找不到，去全局作用域找 */
}                            /* 弹出 foo 作用域 */
```

---

## 四、对 ECPG 改造的启示

如果你要在改造的 ECPG 中支持"不用 DECLARE SECTION"，有两种实现路径：

### 方案 A：轻量作用域（推荐）

不实现完整的 C 解析器，只识别**函数边界**和**最外层块**：

```c
/* 只区分"全局"和"当前函数"两个层级 */
struct scope global_scope;
struct scope *current_func_scope = &global_scope;

/* 遇到函数定义开头，创建新 scope */
/* 遇到函数结束的 }，销毁 scope，回到 global_scope */
```

这样同名局部变量至少能在不同函数间隔离，实现简单。

### 方案 B：完整作用域（Pro*C 级兼容）

支持块级嵌套（`if`/`for`/`while` 里的局部变量也能遮蔽外层），需要：
- 维护 `block_depth` 栈
- 每个 `{` 压栈，`}` 弹栈
- 查找时从内层到外层

工作量大，但和 Pro*C 行为完全一致。

---

## 一句话

> **是的，必须存作用域信息（至少要有函数名/作用域层级指针），否则不同函数中的同名局部变量会互相冲突。** 标准 ECPG 用扁平链表是因为所有宿主变量都在全局 `DECLARE SECTION` 里；要支持 Pro*C 式的"随处声明"，就必须给符号表加上作用域维度，查找时按 C 语言规则从内层向外层回溯。


# 问题 按 EDB -C PROC   这种方案实现，请列出详细的设计方案。不要修改代码
# 回复

以下是按 **EDB `-C PROC` Pro*C 兼容模式** 设计的完整方案，目标是让 ECPG 支持**全局扫描 C 代码中的变量声明**，使宿主变量**不必强制放在 `BEGIN/END DECLARE SECTION`** 中。

---

## 一、总体架构

引入一个**全局 Pro*C 兼容开关**（由命令行 `-C PROC` 激活）。激活后，预编译器的行为从"仅扫描 DECLARE SECTION"切换为"全文件 C 语法感知模式"。

核心新增模块：
- **C 声明扫描器**：在标准 `<C>` 模式下，识别变量定义并注册到符号表
- **分层符号表**：支持全局、函数、块级作用域
- **作用域状态机**：跟踪 `{` / `}` 嵌套深度和函数边界
- **宿主变量解析器**：`EXEC SQL` 语句中的 `:var` 按 C 作用域链查找

---

## 二、命令行接口

新增预编译选项：

| 选项 | 说明 |
|---|---|
| `-C PROC` | 开启 Pro*C 兼容模式，允许 C 代码中任意位置的变量作为宿主变量 |
| `-C INFORMIX` / `-C ORACLE` | 可作为 `-C PROC` 的别名或子集，保持与 EDB 行为对齐 |

**默认行为**：不开启时，保持现有 ECPG 行为（必须 `BEGIN/END DECLARE SECTION`），确保向后兼容。

---

## 三、词法层扩展（Scanner）

### 1. 状态机增强
现有 `<C>` 模式负责扫描 C 代码。在 `-C PROC` 模式下，`<C>` 模式需要**额外识别变量声明模式**，而非简单地把所有内容当作 `S_ANYTHING` 透传。

### 2. 关键识别点
- **类型关键字**：`int`、`char`、`float`、`double`、`short`、`long`、`unsigned`、`struct`、`typedef` 等
- **存储类修饰符**：`static`、`extern`、`auto`、`register`、`const`、`volatile`
- **函数定义边界**：返回类型 + 标识符 + `(` 参数列表 `)` `{`
- **变量声明 vs 表达式语句**：区分 `int x;`（声明）和 `x = 1;`（赋值）

### 3. 伪类型支持
Pro*C 支持 `VARCHAR` 伪类型（实际展开为 `struct`）。在 `-C PROC` 模式下，`<C>` 模式中遇到 `VARCHAR` 关键字时，需要按宿主变量规则处理，而不是当作普通 C 标识符。

---

## 四、语法层扩展（Parser）

### 1. 声明规则增强
在 `preproc.y` 中，现有 `var_declaration` 规则只处理 `BEGIN DECLARE SECTION` 内的声明。新增一套**全局 C 声明识别规则**，在 `-C PROC` 模式下激活。

### 2. 函数/作用域边界规则
新增规则识别：
- **函数定义头**：返回类型 + 函数名 + 参数列表
- **复合语句块**：`{` 开启新块，`}` 关闭当前块
- **参数声明**：K&R 风格或 ANSI C 风格的函数参数

### 3. 与现有规则的隔离
全局 C 声明规则仅在 `-C PROC` 模式下生效。标准模式下，这些规则不参与归约，避免影响现有 ECPG 的编译速度和稳定性。

---

## 五、符号表与作用域管理

### 1. 符号表结构
采用**树形作用域链**：

- **根节点**：全局作用域（Global Scope）
- **函数节点**：每个函数定义创建一个 Function Scope
- **块节点**：函数内的 `{ }` 块创建 Block Scope，支持嵌套

每个作用域节点维护：
- 作用域类型（Global / Function / Block）
- 父作用域指针（用于向上查找）
- 本作用域内的变量哈希表（按变量名索引）
- 函数名（仅 Function Scope 有）

### 2. 变量条目
每个变量记录：
- 变量名
- C 数据类型（基础类型、指针、数组、结构体）
- 类型修饰符（signed/unsigned、const/volatile）
- 数组维度（如果是数组）
- 结构体/typedef 解析链（如果是复合类型）
- 声明行号（用于错误定位）
- 是否是函数参数
- 是否是外部变量（extern）

### 3. 作用域生命周期
- 遇到函数定义头（参数列表结束后的 `{`）：压入 Function Scope
- 遇到普通 `{`：压入 Block Scope
- 遇到 `}`：弹出当前作用域，回到父作用域
- 文件结束：自动清理所有作用域

---

## 六、类型映射系统

### 1. 基础类型映射
建立 C 类型到 ECPG 内部类型枚举的映射表：

| C 类型 | ECPG 内部类型 |
|---|---|
| `char` / `signed char` | `ECPGt_char` |
| `unsigned char` | `ECPGt_unsigned_char` |
| `short` / `short int` | `ECPGt_short` |
| `unsigned short` | `ECPGt_unsigned_short` |
| `int` | `ECPGt_int` |
| `unsigned int` | `ECPGt_unsigned_int` |
| `long` | `ECPGt_long` |
| `unsigned long` | `ECPGt_unsigned_long` |
| `long long` | `ECPGt_long_long` |
| `float` | `ECPGt_float` |
| `double` | `ECPGt_double` |
| `char[n]` / `char *` | `ECPGt_char` + 长度 |
| `VARCHAR[n]` | 特殊处理，展开为 struct |

### 2. 复杂类型处理
- **指针**：记录指针层级（`*`、`**`），但宿主变量绑定通常只支持一级指针（字符串）或数组
- **数组**：记录维度信息，用于数组绑定（`FOR :n INSERT`）
- **结构体**：支持 `%ROWTYPE` 等效物，或按字段逐个映射
- **Typedef**：维护 typedef 别名到实际类型的解析链

### 3. 指示变量（Indicator Variables）
Pro*C 允许指示变量也随处声明。符号表中需要标记哪些变量被用作指示变量（通常以 `_ind` 结尾或显式配对）。

---

## 七、宿主变量解析流程

当 `EXEC SQL` 语句中出现 `:varname` 时：

### 1. 查找流程
1. 获取**当前解析位置所在的作用域**（由 Parser 维护的作用域栈顶）
2. 在当前作用域的变量表中查找 `varname`
3. 若未找到，沿父作用域链逐层向上查找
4. 若找到，获取其类型信息，生成绑定代码
5. 若全局作用域也未找到，报 `undefined host variable` 错误

### 2. 遮蔽（Shadowing）处理
- 内层作用域的同名变量优先于外层
- 函数参数视为函数作用域内的局部变量
- 不同函数中的同名变量互不干扰

### 3. 与显式 DECLARE SECTION 的交互
- 如果变量同时在 `DECLARE SECTION` 和 C 代码中声明，以**当前作用域最近**的为准
- `DECLARE SECTION` 中的变量注册到全局作用域（或一个特殊的"声明区作用域"），作为兜底查找

---

## 八、与现有 DECLARE SECTION 的兼容策略

### 1. 双模式并存
- **标准模式**（默认）：只识别 `BEGIN/END DECLARE SECTION` 中的变量，行为完全不变
- **PROC 模式**（`-C PROC`）：同时识别全局 C 变量和 DECLARE SECTION 变量

### 2. 优先级规则
当两种声明存在冲突时：
- 如果 `:var` 在当前函数作用域内找到了 C 局部变量，优先使用局部变量
- 如果在当前作用域找不到，再去 DECLARE SECTION 的全局符号表查找
- 这样保证局部变量可以**遮蔽**全局 DECLARE 变量，符合 C 程序员直觉

### 3. 混合使用
允许同一份 `.pc` 文件中：
- 部分变量放在 `DECLARE SECTION`（传统写法）
- 部分变量直接声明在 C 代码中（Pro*C 风格）

---

## 九、边界场景处理

### 1. 函数参数作为宿主变量
Pro*C 允许函数参数直接在 SQL 中使用：
```c
void foo(int empno) {
    EXEC SQL SELECT * INTO ... FROM emp WHERE id = :empno;
}
```
设计要点：函数参数在函数作用域创建时即注册到符号表。

### 2. 结构体成员访问
支持 `:struct_var.member` 和 `:struct_var->member`：
- 符号表中记录结构体类型定义
- 遇到 `:` 后解析完整的成员访问链
- 生成对应偏移量的绑定

### 3. 数组下标
支持 `:arr[i]`：
- 识别数组变量 + 下标表达式
- 生成元素地址绑定（`&arr[i]`）

### 4. 宏定义与条件编译
如果变量声明在 `#ifdef` 块内，预编译器需要：
- 要么要求宏已展开（依赖 C 预处理器先处理）
- 要么在 ECPG 层面也支持简单的条件编译跟踪

### 5. 外部变量（extern）
`extern int x;` 应注册到全局作用域，但标记为 `extern`，避免重复定义检查。

---

## 十、验证与测试策略

### 1. 基础功能测试
- 全局变量作为宿主变量
- 函数参数作为宿主变量
- 局部变量作为宿主变量
- 不同函数同名变量隔离
- 内层块变量遮蔽外层

### 2. 类型覆盖测试
- 所有基础 C 类型（char/short/int/long/float/double 及其 unsigned）
- 数组和指针
- VARCHAR 伪类型
- 结构体和 typedef

### 3. 边界测试
- 未声明变量报错
- 作用域外访问局部变量报错
- 类型不匹配警告（如 `char*` 绑定到 `int` 列）

### 4. 兼容性测试
- 同一文件混合使用 `DECLARE SECTION` 和随处声明
- 关闭 `-C PROC` 时行为与原生 ECPG 完全一致

---

## 十一、实施阶段建议

| 阶段 | 目标 |
|---|---|
| **Phase 1** | 实现 `-C PROC` 开关和全局/函数级作用域符号表，支持基础类型（int/char/float/double） |
| **Phase 2** | 支持数组、指针、结构体成员访问 |
| **Phase 3** | 支持块级嵌套作用域和变量遮蔽 |
| **Phase 4** | 支持 VARCHAR 伪类型、typedef 解析、K&R 函数参数 |

---

## 一句话总结

> **EDB `-C PROC` 模式的核心设计是：在 ECPG 预编译器中嵌入一个轻量级 C 声明解析器，维护分层作用域符号表，使 `EXEC SQL` 语句中的 `:var` 能按 C 语言标准作用域规则查找绑定。** 这需要词法层识别 C 类型关键字、语法层跟踪函数和块边界、符号表支持作用域链和遮蔽语义，同时保持与现有 `BEGIN/END DECLARE SECTION` 机制的向后兼容。



