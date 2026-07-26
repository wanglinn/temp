让我先查看 PostgreSQL ECPG 源码中 `ecpg.trailer` 文件的相关内容。
根据对 PostgreSQL ECPG 预处理器源码的分析，问题根源在于：当解析结构体成员时，若遇到未识别的类型名（如未定义的 `typedef`），`var_type` 规则会调用 `get_typedef(name, false)`，而 `get_typedef` 在找不到类型时会调用 **`mmfatal`** 直接 `exit` 退出，导致整个预处理过程终止。

下面给出完整的修改方案，通过引入 `gIgnoreError` 标志，实现在结构体解析遇到类型错误时跳过该结构体、继续后续处理。

---

## 一、问题定位

在 `ecpg.trailer` 的 `var_type` 规则中，以下三处调用 `get_typedef(..., false)` 会在类型未找到时触发 `mmfatal` 退出：

| 位置 | 代码 |
|------|------|
| `STRING_P` 分支 | `struct typedefs *this = get_typedef("string", false);` |
| `IDENT ecpg_interval` 最后的 `else` | `struct typedefs *this = get_typedef(@1, false);` |
| `s_struct_union_symbol` | `this = get_typedef(name, false);` |

`get_typedef` 实现（`variable.c`）：
```c
if (!noerror)
    mmfatal(PARSE_ERROR, "unrecognized data type name \"%s\"", name);
```

`mmfatal` 实现（`util.c`）会直接 `exit(error_code)` 并删除输出文件。

---

## 二、修改方案

涉及 **3 个文件**：`preproc_extern.h`、`ecpg.c`（或新增全局定义文件）、`ecpg.trailer`。

### 1. `preproc_extern.h` — 添加全局标志声明

在文件适当位置（与其他 `extern` 变量声明一起）添加：

```c
/* 结构体解析错误忽略标志：为 true 时，结构体成员类型不识别则跳过该结构体 */
extern bool gIgnoreError;
/* 标记当前结构体解析是否已遇到错误 */
extern bool struct_parse_error;
```

### 2. `ecpg.c` — 定义全局标志

在文件顶部（与其他全局变量定义一起）添加：

```c
bool gIgnoreError = false;       /* 默认关闭：遇到类型错误直接退出 */
bool struct_parse_error = false; /* 标记当前结构体是否解析出错 */
```

> 如需通过命令行控制，可在 `main()` 的 `getopt_long` 循环中新增一个选项（如 `--ignore-struct-error`）来设置 `gIgnoreError = true;`。

### 3. `ecpg.trailer` — 核心语法规则修改

#### (1) 修改 `var_type` 规则中的 `get_typedef` 调用

将三处 `get_typedef(..., false)` 改为根据 `gIgnoreError && struct_level > 0` 决定是否静默返回 NULL，并在返回 NULL 时填充 dummy 类型、标记错误。

**① `STRING_P` 分支修改：**

```yacc
| STRING_P
{
    if (INFORMIX_MODE)
    {
        $$.type_enum = ECPGt_string;
        $$.type_str = "char";
        $$.type_dimension = "-1";
        $$.type_index = "-1";
        $$.type_sizeof = NULL;
    }
    else
    {
        struct typedefs *this = get_typedef("string", gIgnoreError && struct_level > 0);
        if (this == NULL)  /* gIgnoreError 模式下类型未找到 */
        {
            struct_parse_error = true;
            $$.type_str = mm_strdup("string");
            $$.type_enum = ECPGt_long;  /* dummy */
            $$.type_dimension = mm_strdup("-1");
            $$.type_index = mm_strdup("-1");
            $$.type_sizeof = mm_strdup("");
            ECPGfree_struct_member(struct_member_list[struct_level]);
            struct_member_list[struct_level] = NULL;
        }
        else
        {
            $$.type_str = (this->type->type_enum == ECPGt_varchar || this->type->type_enum == ECPGt_bytea) ? mm_strdup("") : mm_strdup(this->name);
            $$.type_enum = this->type->type_enum;
            $$.type_dimension = this->type->type_dimension;
            $$.type_index = this->type->type_index;
            if (this->type->type_sizeof && strlen(this->type->type_sizeof) != 0)
                $$.type_sizeof = this->type->type_sizeof;
            else
                $$.type_sizeof = cat_str(3, "sizeof(", this->name, ")");
            ECPGfree_struct_member(struct_member_list[struct_level]);
            struct_member_list[struct_level] = ECPGstruct_member_dup(this->struct_member_list);
        }
    }
}
```

**② `IDENT ecpg_interval` 最后的 `else` 修改：**

```yacc
else
{
    struct typedefs *this = get_typedef(@1, gIgnoreError && struct_level > 0);
    if (this == NULL)  /* gIgnoreError 模式下类型未找到 */
    {
        struct_parse_error = true;
        $$.type_str = mm_strdup(@1);
        $$.type_enum = ECPGt_long;  /* dummy */
        $$.type_dimension = mm_strdup("-1");
        $$.type_index = mm_strdup("-1");
        $$.type_sizeof = mm_strdup("");
        ECPGfree_struct_member(struct_member_list[struct_level]);
        struct_member_list[struct_level] = NULL;
    }
    else
    {
        $$.type_str = (this->type->type_enum == ECPGt_varchar || this->type->type_enum == ECPGt_bytea) ? "" : this->name;
        $$.type_enum = this->type->type_enum;
        $$.type_dimension = this->type->type_dimension;
        $$.type_index = this->type->type_index;
        if (this->type->type_sizeof && strlen(this->type->type_sizeof) != 0)
            $$.type_sizeof = this->type->type_sizeof;
        else
            $$.type_sizeof = cat_str(3, "sizeof(", this->name, ")");
        ECPGfree_struct_member(struct_member_list[struct_level]);
        struct_member_list[struct_level] = ECPGstruct_member_dup(this->struct_member_list);
    }
}
```

**③ `s_struct_union_symbol` 修改：**

```yacc
| s_struct_union_symbol
{
    char *name;
    struct typedefs *this;
    bool forward = (forward_name != NULL && strcmp($1.symbol, forward_name) == 0 && strcmp($1.su, "struct") == 0);

    name = cat2_str($1.su, $1.symbol);
    if (!forward)
    {
        this = get_typedef(name, gIgnoreError && struct_level > 0);
        if (this == NULL)  /* gIgnoreError 模式下类型未找到 */
        {
            struct_parse_error = true;
            $$.type_str = mm_strdup(name);
            $$.type_enum = ECPGt_long;  /* dummy */
            $$.type_dimension = mm_strdup("-1");
            $$.type_index = mm_strdup("-1");
            $$.type_sizeof = mm_strdup("");
            ECPGfree_struct_member(struct_member_list[struct_level]);
            struct_member_list[struct_level] = NULL;
        }
        else
        {
            $$.type_str = this->name;
            $$.type_enum = this->type->type_enum;
            $$.type_dimension = this->type->type_dimension;
            $$.type_index = this->type->type_index;
            $$.type_sizeof = this->type->type_sizeof;
            ECPGfree_struct_member(struct_member_list[struct_level]);
            struct_member_list[struct_level] = ECPGstruct_member_dup(this->struct_member_list);
        }
    }
    else
    {
        $$.type_str = name;
        $$.type_enum = ECPGt_long;
        $$.type_dimension = "-1";
        $$.type_index = "-1";
        $$.type_sizeof = "";
        ECPGfree_struct_member(struct_member_list[struct_level]);
        struct_member_list[struct_level] = NULL;
    }
}
```

#### (2) 修改 `struct_union_type_with_symbol` 规则

在进入结构体定义时重置错误标志；在结束时检查是否出错，若出错则跳过注册 typedef、清理成员列表、直接输出原始文本。

```yacc
struct_union_type_with_symbol: s_struct_union_symbol
{
    ECPGfree_struct_member(struct_member_list[struct_level]);
    struct_member_list[struct_level++] = NULL;
    if (struct_level >= STRUCT_DEPTH)
        mmerror(PARSE_ERROR, ET_ERROR, "too many levels in nested structure/union definition");
    forward_name = mm_strdup($1.symbol);
    struct_parse_error = false;  /* 重置错误标志 */
}
'{' variable_declarations '}'
{
    struct typedefs *ptr, *this;
    struct this_type su_type;

    if (gIgnoreError && struct_parse_error)
    {
        /* 跳过该结构体：不注册 typedef，清理成员列表，输出原始代码 */
        ECPGfree_struct_member(struct_member_list[struct_level]);
        struct_member_list[struct_level] = NULL;
        struct_level--;
        free(forward_name);
        forward_name = NULL;
        struct_parse_error = false;

        @$ = cat_str(4, cat2_str($1.su, $1.symbol), "{", @4, "}");
    }
    else
    {
        /* 原有正常逻辑 */
        ECPGfree_struct_member(struct_member_list[struct_level]);
        struct_member_list[struct_level] = NULL;
        struct_level--;
        if (strcmp($1.su, "struct") == 0)
            su_type.type_enum = ECPGt_struct;
        else
            su_type.type_enum = ECPGt_union;
        su_type.type_str = cat2_str($1.su, $1.symbol);
        free(forward_name);
        forward_name = NULL;

        for (ptr = types; ptr != NULL; ptr = ptr->next)
        {
            if (strcmp(su_type.type_str, ptr->name) == 0)
                mmerror(PARSE_ERROR, ET_ERROR, "type \"%s\" is already defined", su_type.type_str);
        }

        this = (struct typedefs *) mm_alloc(sizeof(struct typedefs));
        this->next = types;
        this->name = mm_strdup(su_type.type_str);
        this->brace_level = braces_open;
        this->type = (struct this_type *) mm_alloc(sizeof(struct this_type));
        this->type->type_storage = NULL;
        this->type->type_enum = su_type.type_enum;
        this->type->type_str = mm_strdup(su_type.type_str);
        this->type->type_dimension = mm_strdup("-1");
        this->type->type_index = mm_strdup("-1");
        this->type->type_sizeof = ECPGstruct_sizeof ? mm_strdup(ECPGstruct_sizeof) : NULL;
        this->struct_member_list = ECPGstruct_member_dup(struct_member_list[struct_level]);
        types = this;
        @$ = cat_str(4, su_type.type_str, "{", @4, "}");
    }
}
;
```

#### (3) 修改 `struct_union_type` 规则（匿名结构体）

```yacc
struct_union_type: struct_union_type_with_symbol
| s_struct_union
{
    ECPGfree_struct_member(struct_member_list[struct_level]);
    struct_member_list[struct_level++] = NULL;
    if (struct_level >= STRUCT_DEPTH)
        mmerror(PARSE_ERROR, ET_ERROR, "too many levels in nested structure/union definition");
    struct_parse_error = false;  /* 重置错误标志 */
}
'{' variable_declarations '}'
{
    if (gIgnoreError && struct_parse_error)
    {
        /* 跳过该匿名结构体 */
        ECPGfree_struct_member(struct_member_list[struct_level]);
        struct_member_list[struct_level] = NULL;
        struct_level--;
        struct_parse_error = false;
        @$ = cat_str(4, @1, "{", @4, "}");
    }
    else
    {
        ECPGfree_struct_member(struct_member_list[struct_level]);
        struct_member_list[struct_level] = NULL;
        struct_level--;
        @$ = cat_str(4, @1, "{", @4, "}");
    }
}
;
```

---

## 三、方案说明

| 场景 | 行为 |
|------|------|
| `gIgnoreError = false`（默认） | 保持原有行为：结构体成员类型不识别时，`get_typedef` 调用 `mmfatal` 直接退出。 |
| `gIgnoreError = true` 且正在解析结构体（`struct_level > 0`） | 类型不识别时，`get_typedef` 静默返回 `NULL`，设置 `struct_parse_error = true`，填充 dummy 类型让解析继续；在结构体定义结束时检测到错误标志，跳过 typedef 注册、清理成员列表、直接输出原始结构体代码到生成的 `.c` 文件中，然后继续解析后续内容。 |
| `gIgnoreError = true` 但不在结构体中（`struct_level == 0`） | 非结构体上下文中的类型错误仍走原有逻辑（调用 `mmfatal` 退出），因为需求仅针对"解析结构体"场景。 |

---

## 四、编译与使用

修改后需要重新生成解析器并编译：

```bash
cd src/interfaces/ecpg/preproc
# 若修改了 ecpg.trailer，需要重新生成 preproc.y 并编译
make clean && make
```

在代码中启用跳过模式（例如在 `main()` 中根据条件设置）：

```c
gIgnoreError = true;  /* 或通过命令行参数传入 */
```

这样，当 ECPG 预处理遇到结构体中包含不认识的类型（如第三方库定义的类型尚未被 ECPG 识别）时，会跳过该结构体定义、保留原始 C 代码、继续处理文件剩余部分，而不是直接终止。
