好的，基于你提供的 `pgc.l` 完整文件（未包含之前的 `#ifdef`/`#define` patch），我来做**第一步**的修改。

## 第一步目标：支持 `int` 类型变量定义提取（含数组形式，支持 `unsigned`/`signed` 修饰，过滤 `extern`）

### 实现思路

在 `parse_include()` 中，当通过 `#include "xx.h"` 打开头文件时：
1. 先读取头文件内容
2. 逐行扫描，提取 `int` / `unsigned int` / `signed int` 变量定义（含数组形式）
3. 过滤 `extern int ...` 和函数声明
4. 将提取内容写入临时文件 `xx.h.extract`，前后包裹 `EXEC SQL BEGIN/END DECLARE SECTION`
5. 让 ECPG 解析临时文件而非原文件

### 修改内容

需要修改的部分：
1. **新增辅助函数**：`extract_host_vars()` 负责从头文件提取宿主变量
2. **修改 `parse_include()`**：对 `#include "xx.h"` 双引号形式的头文件，先提取再打开临时文件

---

下面是完整 patch：

```diff
--- a/src/interfaces/ecpg/preproc/pgc.l
+++ b/src/interfaces/ecpg/preproc/pgc.l
@@ -79,6 +79,7 @@ static void parse_include(void);
 static bool ecpg_isspace(char ch);
 static bool isdefine(void);
 static bool isinformixdefine(void);
+static bool extract_host_vars(const char *srcfile, const char *dstfile);
 
 char	   *token_start;
 
@@ -2434,6 +2435,206 @@ parse_include(void)
 	BEGIN(C);
 }
 
+/*
+ * extract_host_vars --- 从头文件中提取 int 类型宿主变量定义
+ *
+ * 支持: int, unsigned int, signed int 变量定义（含数组形式）
+ * 过滤: extern, 函数声明, 其他类型
+ *
+ * 返回 true 表示成功提取到内容，false 表示无内容或失败
+ */
+static bool
+extract_host_vars(const char *srcfile, const char *dstfile)
+{
+	FILE *src, *dst;
+	char line[4096];
+	bool has_content = false;
+
+	src = fopen(srcfile, "r");
+	if (!src)
+		return false;
+
+	dst = fopen(dstfile, "w");
+	if (!dst)
+	{
+		fclose(src);
+		return false;
+	}
+
+	/* 写入 BEGIN DECLARE SECTION */
+	fprintf(dst, "EXEC SQL BEGIN DECLARE SECTION;\n");
+
+	while (fgets(line, sizeof(line), src))
+	{
+		char *p = line;
+		char *start;
+		bool is_extern = false;
+		bool is_unsigned = false;
+		bool is_signed = false;
+		bool is_int = false;
+		bool has_paren = false;
+		bool has_bracket = false;
+		int bracket_depth = 0;
+
+		/* 跳过行首空白 */
+		while (*p && (*p == ' ' || *p == '\t'))
+			p++;
+
+		/* 跳过空行 */
+		if (*p == '\n' || *p == '\r' || *p == '\0')
+			continue;
+
+		/* 跳过注释行 // ... */
+		if (p[0] == '/' && p[1] == '/')
+		{
+			/* 保留注释行？不，第一步先跳过 */
+			continue;
+		}
+
+		/* 检查 extern */
+		start = p;
+		if ((p[0] == 'e' || p[0] == 'E') &&
+			(p[1] == 'x' || p[1] == 'X') &&
+			(p[2] == 't' || p[2] == 'T') &&
+			(p[3] == 'e' || p[3] == 'E') &&
+			(p[4] == 'r' || p[4] == 'R') &&
+			(p[5] == 'n' || p[5] == 'N') &&
+			(p[6] == ' ' || p[6] == '\t'))
+		{
+			is_extern = true;
+		}
+
+		/* 检查 unsigned / signed */
+		if (!is_extern)
+		{
+			if ((p[0] == 'u' || p[0] == 'U') &&
+				(p[1] == 'n' || p[1] == 'N') &&
+				(p[2] == 's' || p[2] == 'S') &&
+				(p[3] == 'i' || p[3] == 'I') &&
+				(p[4] == 'g' || p[4] == 'G') &&
+				(p[5] == 'n' || p[5] == 'N') &&
+				(p[6] == 'e' || p[6] == 'E') &&
+				(p[7] == 'd' || p[7] == 'D') &&
+				(p[8] == ' ' || p[8] == '\t'))
+			{
+				is_unsigned = true;
+				p += 9;
+				while (*p && (*p == ' ' || *p == '\t'))
+					p++;
+			}
+			else if ((p[0] == 's' || p[0] == 'S') &&
+					 (p[1] == 'i' || p[1] == 'I') &&
+					 (p[2] == 'g' || p[2] == 'G') &&
+					 (p[3] == 'n' || p[3] == 'N') &&
+					 (p[4] == 'e' || p[4] == 'E') &&
+					 (p[5] == 'd' || p[5] == 'D') &&
+					 (p[6] == ' ' || p[6] == '\t'))
+			{
+				is_signed = true;
+				p += 7;
+				while (*p && (*p == ' ' || *p == '\t'))
+					p++;
+			}
+		}
+
+		/* 检查 int */
+		if (!is_extern)
+		{
+			if ((p[0] == 'i' || p[0] == 'I') &&
+				(p[1] == 'n' || p[1] == 'N') &&
+				(p[2] == 't' || p[2] == 'T') &&
+				(p[3] == ' ' || p[3] == '\t' || p[3] == '\n' || p[3] == '\r' ||
+				 p[3] == ';' || p[3] == '\0'))
+			{
+				is_int = true;
+			}
+		}
+
+		/* 检查是否包含 '(' —— 函数声明的特征 */
+		if (is_int && !is_extern)
+		{
+			char *q = p + 3;
+			while (*q && *q != '\n' && *q != '\r')
+			{
+				if (*q == '(')
+				{
+					has_paren = true;
+					break;
+				}
+				q++;
+			}
+		}
+
+		/* 检查是否包含 '[' —— 数组 */
+		if (is_int && !is_extern && !has_paren)
+		{
+			char *q = p + 3;
+			while (*q && *q != '\n' && *q != '\r')
+			{
+				if (*q == '[')
+				{
+					has_bracket = true;
+					break;
+				}
+				q++;
+			}
+		}
+
+		/* 提取条件：是 int 类型定义，不是 extern，不是函数声明 */
+		if (is_int && !is_extern && !has_paren)
+		{
+			/* 整行写入 */
+			fprintf(dst, "%s", line);
+			has_content = true;
+		}
+	}
+
+	/* 写入 END DECLARE SECTION */
+	fprintf(dst, "EXEC SQL END DECLARE SECTION;\n");
+
+	fclose(src);
+	fclose(dst);
+
+	/* 如果没有提取到任何内容，删除临时文件并返回 false */
+	if (!has_content)
+	{
+		remove(dstfile);
+		return false;
+	}
+
+	return true;
+}
+
 /*
  * ecpg_isspace() --- return true if flex scanner considers char whitespace
  */
```

### 修改 `parse_include()` 函数

在 `parse_include()` 中，当处理双引号包含的头文件时，增加提取逻辑：

```diff
@@ -2360,6 +2561,8 @@ parse_include(void)
 	struct _yy_buffer *yb;
 	struct _include_path *ip;
 	char		inc_file[MAXPGPATH];
+	char		extract_file[MAXPGPATH];
+	bool		use_extract = false;
 	unsigned int i;
 
 	yb = mm_alloc(sizeof(struct _yy_buffer));
@@ -2403,6 +2606,18 @@ parse_include(void)
 		memmove(yytext, yytext + 1, strlen(yytext));
 
 		strlcpy(inc_file, yytext, sizeof(inc_file));
+
+		/* 对双引号包含的头文件尝试提取宿主变量 */
+		snprintf(extract_file, sizeof(extract_file), "%s.extract", inc_file);
+		if (extract_host_vars(inc_file, extract_file))
+		{
+			use_extract = true;
+		}
+		else
+		{
+			/* 提取失败或无内容，使用原文件 */
+			use_extract = false;
+		}
+
 		yyin = fopen(inc_file, "r");
 		if (!yyin)
 		{
@@ -2453,6 +2668,13 @@ parse_include(void)
 	if (!yyin)
 		mmfatal(NO_INCLUDE_FILE, "could not open include file \"%s\" on line %d", yytext, yylineno);
 
+	/* 如果提取成功，切换到临时文件 */
+	if (use_extract)
+	{
+		fclose(yyin);
+		yyin = fopen(extract_file, "r");
+		input_filename = mm_strdup(extract_file);
+	}
+	else
+	{
+		input_filename = mm_strdup(inc_file);
+	}
+
-	input_filename = mm_strdup(inc_file);
 	yy_switch_to_buffer(yy_create_buffer(yyin, YY_BUF_SIZE));
 	yylineno = 1;
 	output_line_number();
```

---

### 需要确认的问题

1. **临时文件删除时机**：当前 patch 中临时文件不会自动删除。需要在 `<<EOF>>` 处理或文件切换时删除。是否需要我加上删除逻辑？

2. **注释处理**：`//` 注释行当前被跳过。`/* ... */` 块注释需要更复杂处理，第一步先跳过是否可接受？

3. **多行声明**：如 `int a,\n    b;` 这种跨行声明，当前逐行处理会漏掉。第一步是否只支持单行声明？

4. **初始化值**：`int a = 10;` 当前会被提取（因为只检查 `int` 开头和 `(` 过滤）。是否需要去掉初始化值？

请确认后我可以调整 patch。

# 2 临时文件处理规则，若全局变量 debug_log = 1 （对应ecpg -D ） 则不删除，否则删除 。 

注释跳过。

当前 只支持单行声明。

初始化值：int a = 10; 当前会被提取。 这个没关系，因为我们只是为了找到行内容 然后当道临时文件中。



另外，临时文件中内容的行号 是否和 原始文件可以对应上， 也就是 从第n行要提取的内容 放到临时文件也是第n行


>
>明白了，需求明确：

1. **临时文件删除规则**：`debug_log = 1`（`ecpg -D` 开启）时不删除，否则删除
2. **注释跳过**：`//` 和 `/* ... */` 都跳过
3. **只支持单行声明**
4. **初始化值保留**：原样提取
5. **行号对齐**：临时文件中第 n 行的内容对应原文件第 n 行

行号对齐的实现方式：在临时文件中，**不提取的行用空行占位**，保持行号一致。

---

下面是完整 patch：

```diff
--- a/src/interfaces/ecpg/preproc/pgc.l
+++ b/src/interfaces/ecpg/preproc/pgc.l
@@ -79,6 +79,8 @@ static void parse_include(void);
 static bool ecpg_isspace(char ch);
 static bool isdefine(void);
 static bool isinformixdefine(void);
+static bool extract_host_vars(const char *srcfile, const char *dstfile);
+static int debug_log = 0;  /* 对应 ecpg -D 调试模式 */
 
 char	   *token_start;
 
@@ -2434,6 +2436,245 @@ parse_include(void)
 	BEGIN(C);
 }
 
+/*
+ * is_int_var_line --- 判断一行是否为 int 类型变量定义（含数组）
+ *
+ * 支持: int, unsigned int, signed int 变量定义（含数组形式）
+ * 过滤: extern, 函数声明(含'('), 预处理指令, 其他类型
+ *
+ * 返回: true 表示需要提取，false 表示跳过
+ */
+static bool
+is_int_var_line(const char *line)
+{
+	const char *p = line;
+	bool is_extern = false;
+	bool is_unsigned = false;
+	bool is_signed = false;
+	bool is_int = false;
+	bool has_paren = false;
+
+	/* 跳过行首空白 */
+	while (*p && (*p == ' ' || *p == '\t'))
+		p++;
+
+	/* 空行 */
+	if (*p == '\n' || *p == '\r' || *p == '\0')
+		return false;
+
+	/* 跳过预处理指令 */
+	if (*p == '#')
+		return false;
+
+	/* 跳过 // 注释 */
+	if (p[0] == '/' && p[1] == '/')
+		return false;
+
+	/* 跳过 /* 块注释开始 */
+	if (p[0] == '/' && p[1] == '*')
+		return false;
+
+	/* 检查 extern */
+	if ((p[0] == 'e' || p[0] == 'E') &&
+		(p[1] == 'x' || p[1] == 'X') &&
+		(p[2] == 't' || p[2] == 'T') &&
+		(p[3] == 'e' || p[3] == 'E') &&
+		(p[4] == 'r' || p[4] == 'R') &&
+		(p[5] == 'n' || p[5] == 'N') &&
+		(p[6] == ' ' || p[6] == '\t'))
+	{
+		is_extern = true;
+	}
+
+	/* 检查 unsigned / signed */
+	if (!is_extern)
+	{
+		if ((p[0] == 'u' || p[0] == 'U') &&
+			(p[1] == 'n' || p[1] == 'N') &&
+			(p[2] == 's' || p[2] == 'S') &&
+			(p[3] == 'i' || p[3] == 'I') &&
+			(p[4] == 'g' || p[4] == 'G') &&
+			(p[5] == 'n' || p[5] == 'N') &&
+			(p[6] == 'e' || p[6] == 'E') &&
+			(p[7] == 'd' || p[7] == 'D') &&
+			(p[8] == ' ' || p[8] == '\t'))
+		{
+			is_unsigned = true;
+			p += 9;
+			while (*p && (*p == ' ' || *p == '\t'))
+				p++;
+		}
+		else if ((p[0] == 's' || p[0] == 'S') &&
+				 (p[1] == 'i' || p[1] == 'I') &&
+				 (p[2] == 'g' || p[2] == 'G') &&
+				 (p[3] == 'n' || p[3] == 'N') &&
+				 (p[4] == 'e' || p[4] == 'E') &&
+				 (p[5] == 'd' || p[5] == 'D') &&
+				 (p[6] == ' ' || p[6] == '\t'))
+		{
+			is_signed = true;
+			p += 7;
+			while (*p && (*p == ' ' || *p == '\t'))
+				p++;
+		}
+	}
+
+	/* 检查 int */
+	if (!is_extern)
+	{
+		if ((p[0] == 'i' || p[0] == 'I') &&
+			(p[1] == 'n' || p[1] == 'N') &&
+			(p[2] == 't' || p[2] == 'T') &&
+			(p[3] == ' ' || p[3] == '\t' || p[3] == '\n' || p[3] == '\r' ||
+			 p[3] == ';' || p[3] == '\0'))
+		{
+			is_int = true;
+		}
+	}
+
+	/* 检查是否包含 '(' —— 函数声明的特征 */
+	if (is_int && !is_extern)
+	{
+		const char *q = p + 3;
+		while (*q && *q != '\n' && *q != '\r')
+		{
+			if (*q == '(')
+			{
+				has_paren = true;
+				break;
+			}
+			q++;
+		}
+	}
+
+	/* 提取条件：是 int 类型定义，不是 extern，不是函数声明 */
+	return (is_int && !is_extern && !has_paren);
+}
+
+/*
+ * extract_host_vars --- 从头文件中提取 int 类型宿主变量定义
+ *
+ * 行号对齐：不提取的行用空行占位，保持与原文件行号一致
+ * 返回 true 表示成功提取到内容，false 表示无内容或失败
+ */
+static bool
+extract_host_vars(const char *srcfile, const char *dstfile)
+{
+	FILE *src, *dst;
+	char line[4096];
+	bool has_content = false;
+	int line_no = 0;
+
+	src = fopen(srcfile, "r");
+	if (!src)
+		return false;
+
+	dst = fopen(dstfile, "w");
+	if (!dst)
+	{
+		fclose(src);
+		return false;
+	}
+
+	/* 写入 BEGIN DECLARE SECTION（占一行，但行号计数从1开始，这里作为第0行前缀） */
+	fprintf(dst, "EXEC SQL BEGIN DECLARE SECTION;\n");
+
+	while (fgets(line, sizeof(line), src))
+	{
+		line_no++;
+
+		if (is_int_var_line(line))
+		{
+			fprintf(dst, "%s", line);
+			has_content = true;
+		}
+		else
+		{
+			/* 不提取的行用空行占位，保持行号对齐 */
+			fprintf(dst, "\n");
+		}
+	}
+
+	/* 写入 END DECLARE SECTION */
+	fprintf(dst, "EXEC SQL END DECLARE SECTION;\n");
+
+	fclose(src);
+	fclose(dst);
+
+	/* 如果没有提取到任何内容，删除临时文件 */
+	if (!has_content)
+	{
+		remove(dstfile);
+		return false;
+	}
+
+	return true;
+}
+
 /*
  * ecpg_isspace() --- return true if flex scanner considers char whitespace
  */
@@ -2360,6 +2601,8 @@ parse_include(void)
 	struct _yy_buffer *yb;
 	struct _include_path *ip;
 	char		inc_file[MAXPGPATH];
+	char		extract_file[MAXPGPATH];
+	bool		use_extract = false;
 	unsigned int i;
 
 	yb = mm_alloc(sizeof(struct _yy_buffer));
@@ -2403,6 +2646,16 @@ parse_include(void)
 		memmove(yytext, yytext + 1, strlen(yytext));
 
 		strlcpy(inc_file, yytext, sizeof(inc_file));
+
+		/* 对双引号包含的头文件尝试提取宿主变量 */
+		snprintf(extract_file, sizeof(extract_file), "%s.extract", inc_file);
+		if (extract_host_vars(inc_file, extract_file))
+		{
+			use_extract = true;
+		}
+		else
+		{
+			use_extract = false;
+		}
+
 		yyin = fopen(inc_file, "r");
 		if (!yyin)
 		{
@@ -2453,6 +2706,21 @@ parse_include(void)
 	if (!yyin)
 		mmfatal(NO_INCLUDE_FILE, "could not open include file \"%s\" on line %d", yytext, yylineno);
 
+	/* 如果提取成功，切换到临时文件 */
+	if (use_extract)
+	{
+		fclose(yyin);
+		yyin = fopen(extract_file, "r");
+		if (!yyin)
+			mmfatal(NO_INCLUDE_FILE, "could not open extract file \"%s\" on line %d", extract_file, yylineno);
+		input_filename = mm_strdup(extract_file);
+	}
+	else
+	{
+		input_filename = mm_strdup(inc_file);
+	}
+
-	input_filename = mm_strdup(inc_file);
 	yy_switch_to_buffer(yy_create_buffer(yyin, YY_BUF_SIZE));
 	yylineno = 1;
 	output_line_number();
@@ -2471,6 +2739,31 @@ parse_include(void)
 	BEGIN(C);
 }
 
+/* 在文件切换或 EOF 时清理临时文件 */
+static void
+cleanup_extract_file(void)
+{
+	/* debug_log = 1 时不删除（对应 ecpg -D 调试模式） */
+	if (debug_log)
+		return;
+
+	/* 清理当前文件的 .extract 临时文件 */
+	if (input_filename != NULL)
+	{
+		size_t len = strlen(input_filename);
+		if (len > 8 && strcmp(input_filename + len - 8, ".extract") == 0)
+		{
+			remove(input_filename);
+		}
+	}
+}
+
 /*
  * ecpg_isspace() --- return true if flex scanner considers char whitespace
  */
@@ -2500,6 +2793,9 @@ isdefine(void)
 			ptr->used == NULL)
 		{
 			/* Save state associated with the current buffer */
+			/* 清理可能存在的临时文件 */
+			cleanup_extract_file();
+
 			struct _yy_buffer *yb;
 
 			yb = mm_alloc(sizeof(struct _yy_buffer));
@@ -2550,6 +2846,9 @@ isinformixdefine(void)
 	if (new)
 	{
+		/* 清理可能存在的临时文件 */
+		cleanup_extract_file();
+
 		struct _yy_buffer *yb;
 
 		yb = mm_alloc(sizeof(struct _yy_buffer));
@@ -2580,6 +2879,9 @@ isinformixdefine(void)
 					{
 						if (yy_buffer == NULL)
 						{
+							/* 退出主文件前清理临时文件 */
+							cleanup_extract_file();
+
 							/* No more input */
 							if (preproc_tos > 0)
 							{
@@ -2590,6 +2892,9 @@ isinformixdefine(void)
 						}
 						else
 						{
+							/* 切换文件前清理临时文件 */
+							cleanup_extract_file();
+
 							/* Revert to previous input source */
 							struct _yy_buffer *yb = yy_buffer;
 							int			i;
```

---

### 说明

1. **`is_int_var_line()`**：判断单行是否为 `int` 类型变量定义，过滤 `extern`、函数声明（含 `(`）、预处理指令、`//` 注释、`/*` 块注释。

2. **`extract_host_vars()`**：
   - 逐行读取头文件
   - 提取的行原样写入临时文件
   - 不提取的行用空行 `\n` 占位，**保持行号对齐**
   - 前后包裹 `EXEC SQL BEGIN/END DECLARE SECTION`

3. **`parse_include()` 修改**：
   - 双引号包含的头文件（`#include "xx.h"`）先调用 `extract_host_vars()`
   - 提取成功则打开 `.extract` 临时文件
   - 提取失败或无内容则使用原文件

4. **`cleanup_extract_file()`**：
   - 在文件切换或 EOF 时调用
   - `debug_log = 1` 时不删除（调试模式保留）
   - 否则删除 `.extract` 临时文件

5. **`debug_log` 初始化**：需要在 `lex_init()` 或其他初始化位置根据 `ecpg -D` 参数设置。当前 patch 中 `debug_log = 0` 为默认值，如果你有现有的命令行参数解析逻辑传入，可以替换。

---

### 验证场景

**h1.h**：
```c
#ifndef H1_H
#define H1_H

// 这是注释
int global_a;
unsigned int global_b;
signed int global_c;
extern int ext_val;          /* 过滤 */
void helper(void);           /* 过滤 */
int arr[10];
int arr2[20][10];
int initialized = 100;       /* 保留 */

#endif
```

**生成的 h1.h.extract**：
```c
EXEC SQL BEGIN DECLARE SECTION;
                              /* 第1行：#ifndef 跳过，空行 */
                              /* 第2行：#define 跳过，空行 */
                              /* 第3行：// 注释跳过，空行 */
int global_a;                 /* 第4行：提取 */
unsigned int global_b;        /* 第5行：提取 */
signed int global_c;          /* 第6行：提取 */
                              /* 第7行：extern 过滤，空行 */
                              /* 第8行：函数声明过滤，空行 */
int arr[10];                  /* 第9行：提取 */
int arr2[20][10];             /* 第10行：提取 */
int initialized = 100;        /* 第11行：提取 */
                              /* 第12行：#endif 跳过，空行 */
EXEC SQL END DECLARE SECTION;
```

行号对齐：原文件第 4 行 `int global_a;` 在临时文件中也是第 4 行（前面 `BEGIN DECLARE SECTION` 占第 1 行，后面空行占位）。

等等，这里有个问题：`BEGIN DECLARE SECTION` 插入后，原文件第 1 行变成了临时文件第 2 行，行号偏移了 1。

你是否需要**绝对行号对齐**（即错误报告指向原文件正确行号）？如果是，需要注入 `#line` 指令。请确认。

# 不需要绝对对齐

》

好的，不需要绝对行号对齐。那当前 patch 的行号方案（`BEGIN DECLARE SECTION` 占第 1 行，原文件内容从第 2 行开始，不提取的行用空行占位）即可满足需求。

不过我需要修正 patch 中的一个问题：`cleanup_extract_file()` 的插入位置有误，会导致编译错误（C 语言不允许在变量声明前插入语句）。让我重新整理一个干净的 patch。

另外，`debug_log` 的初始化需要结合你现有的命令行参数处理。如果你已有 `-D` 参数解析逻辑，请告诉我变量名，我可以替换。

下面是修正后的完整 patch：

```diff
--- a/src/interfaces/ecpg/preproc/pgc.l
+++ b/src/interfaces/ecpg/preproc/pgc.l
@@ -79,6 +79,8 @@ static void parse_include(void);
 static bool ecpg_isspace(char ch);
 static bool isdefine(void);
 static bool isinformixdefine(void);
+static bool extract_host_vars(const char *srcfile, const char *dstfile);
+static int debug_log = 0;  /* 对应 ecpg -D 调试模式，=1 时不删除临时文件 */
 
 char	   *token_start;
 
@@ -2360,6 +2362,8 @@ parse_include(void)
 	struct _yy_buffer *yb;
 	struct _include_path *ip;
 	char		inc_file[MAXPGPATH];
+	char		extract_file[MAXPGPATH];
+	bool		use_extract = false;
 	unsigned int i;
 
 	yb = mm_alloc(sizeof(struct _yy_buffer));
@@ -2403,6 +2407,16 @@ parse_include(void)
 		memmove(yytext, yytext + 1, strlen(yytext));
 
 		strlcpy(inc_file, yytext, sizeof(inc_file));
+
+		/* 对双引号包含的头文件尝试提取宿主变量 */
+		snprintf(extract_file, sizeof(extract_file), "%s.extract", inc_file);
+		if (extract_host_vars(inc_file, extract_file))
+		{
+			use_extract = true;
+		}
+		else
+		{
+			use_extract = false;
+		}
+
 		yyin = fopen(inc_file, "r");
 		if (!yyin)
 		{
@@ -2453,6 +2467,21 @@ parse_include(void)
 	if (!yyin)
 		mmfatal(NO_INCLUDE_FILE, "could not open include file \"%s\" on line %d", yytext, yylineno);
 
+	/* 如果提取成功，切换到临时文件 */
+	if (use_extract)
+	{
+		fclose(yyin);
+		yyin = fopen(extract_file, "r");
+		if (!yyin)
+			mmfatal(NO_INCLUDE_FILE, "could not open extract file \"%s\" on line %d", extract_file, yylineno);
+		input_filename = mm_strdup(extract_file);
+	}
+	else
+	{
+		input_filename = mm_strdup(inc_file);
+	}
+
-	input_filename = mm_strdup(inc_file);
 	yy_switch_to_buffer(yy_create_buffer(yyin, YY_BUF_SIZE));
 	yylineno = 1;
 	output_line_number();
@@ -2471,6 +2500,228 @@ parse_include(void)
 	BEGIN(C);
 }
 
+/*
+ * is_int_var_line --- 判断一行是否为 int 类型变量定义（含数组）
+ *
+ * 支持: int, unsigned int, signed int 变量定义（含数组形式）
+ * 过滤: extern, 函数声明(含'('), 预处理指令, 注释
+ *
+ * 返回: true 表示需要提取，false 表示跳过
+ */
+static bool
+is_int_var_line(const char *line)
+{
+	const char *p = line;
+	bool is_extern = false;
+	bool is_int = false;
+	bool has_paren = false;
+
+	/* 跳过行首空白 */
+	while (*p && (*p == ' ' || *p == '\t'))
+		p++;
+
+	/* 空行 */
+	if (*p == '\n' || *p == '\r' || *p == '\0')
+		return false;
+
+	/* 跳过预处理指令 */
+	if (*p == '#')
+		return false;
+
+	/* 跳过 // 注释 */
+	if (p[0] == '/' && p[1] == '/')
+		return false;
+
+	/* 跳过 /* 块注释开始 */
+	if (p[0] == '/' && p[1] == '*')
+		return false;
+
+	/* 检查 extern */
+	if ((p[0] == 'e' || p[0] == 'E') &&
+		(p[1] == 'x' || p[1] == 'X') &&
+		(p[2] == 't' || p[2] == 'T') &&
+		(p[3] == 'e' || p[3] == 'E') &&
+		(p[4] == 'r' || p[4] == 'R') &&
+		(p[5] == 'n' || p[5] == 'N') &&
+		(p[6] == ' ' || p[6] == '\t'))
+	{
+		is_extern = true;
+	}
+
+	/* 检查 unsigned / signed */
+	if (!is_extern)
+	{
+		if ((p[0] == 'u' || p[0] == 'U') &&
+			(p[1] == 'n' || p[1] == 'N') &&
+			(p[2] == 's' || p[2] == 'S') &&
+			(p[3] == 'i' || p[3] == 'I') &&
+			(p[4] == 'g' || p[4] == 'G') &&
+			(p[5] == 'n' || p[5] == 'N') &&
+			(p[6] == 'e' || p[6] == 'E') &&
+			(p[7] == 'd' || p[7] == 'D') &&
+			(p[8] == ' ' || p[8] == '\t'))
+		{
+			p += 9;
+			while (*p && (*p == ' ' || *p == '\t'))
+				p++;
+		}
+		else if ((p[0] == 's' || p[0] == 'S') &&
+				 (p[1] == 'i' || p[1] == 'I') &&
+				 (p[2] == 'g' || p[2] == 'G') &&
+				 (p[3] == 'n' || p[3] == 'N') &&
+				 (p[4] == 'e' || p[4] == 'E') &&
+				 (p[5] == 'd' || p[5] == 'D') &&
+				 (p[6] == ' ' || p[6] == '\t'))
+		{
+			p += 7;
+			while (*p && (*p == ' ' || *p == '\t'))
+				p++;
+		}
+	}
+
+	/* 检查 int */
+	if (!is_extern)
+	{
+		if ((p[0] == 'i' || p[0] == 'I') &&
+			(p[1] == 'n' || p[1] == 'N') &&
+			(p[2] == 't' || p[2] == 'T') &&
+			(p[3] == ' ' || p[3] == '\t' || p[3] == '\n' || p[3] == '\r' ||
+			 p[3] == ';' || p[3] == '\0'))
+		{
+			is_int = true;
+		}
+	}
+
+	/* 检查是否包含 '(' —— 函数声明的特征 */
+	if (is_int && !is_extern)
+	{
+		const char *q = p + 3;
+		while (*q && *q != '\n' && *q != '\r')
+		{
+			if (*q == '(')
+			{
+				has_paren = true;
+				break;
+			}
+			q++;
+		}
+	}
+
+	/* 提取条件：是 int 类型定义，不是 extern，不是函数声明 */
+	return (is_int && !is_extern && !has_paren);
+}
+
+/*
+ * extract_host_vars --- 从头文件中提取 int 类型宿主变量定义
+ *
+ * 行号对齐：不提取的行用空行占位，保持与原文件行号一致
+ * 返回 true 表示成功提取到内容，false 表示无内容或失败
+ */
+static bool
+extract_host_vars(const char *srcfile, const char *dstfile)
+{
+	FILE *src, *dst;
+	char line[4096];
+	bool has_content = false;
+
+	src = fopen(srcfile, "r");
+	if (!src)
+		return false;
+
+	dst = fopen(dstfile, "w");
+	if (!dst)
+	{
+		fclose(src);
+		return false;
+	}
+
+	/* 写入 BEGIN DECLARE SECTION（占第1行） */
+	fprintf(dst, "EXEC SQL BEGIN DECLARE SECTION;\n");
+
+	while (fgets(line, sizeof(line), src))
+	{
+		if (is_int_var_line(line))
+		{
+			fprintf(dst, "%s", line);
+			has_content = true;
+		}
+		else
+		{
+			/* 不提取的行用空行占位，保持行号对齐 */
+			fprintf(dst, "\n");
+		}
+	}
+
+	/* 写入 END DECLARE SECTION */
+	fprintf(dst, "EXEC SQL END DECLARE SECTION;\n");
+
+	fclose(src);
+	fclose(dst);
+
+	/* 如果没有提取到任何内容，删除临时文件 */
+	if (!has_content)
+	{
+		remove(dstfile);
+		return false;
+	}
+
+	return true;
+}
+
+/*
+ * cleanup_extract_file --- 清理临时 .extract 文件
+ *
+ * debug_log = 1 时不删除（调试模式保留）
+ */
+static void
+cleanup_extract_file(void)
+{
+	if (debug_log)
+		return;
+
+	if (input_filename != NULL)
+	{
+		size_t len = strlen(input_filename);
+		if (len > 8 && strcmp(input_filename + len - 8, ".extract") == 0)
+		{
+			remove(input_filename);
+		}
+	}
+}
+
 /*
  * ecpg_isspace() --- return true if flex scanner considers char whitespace
  */
@@ -2498,6 +2739,9 @@ isdefine(void)
 			ptr->value != NULL &&
 			ptr->used == NULL)
 		{
+			/* 清理可能存在的临时文件 */
+			cleanup_extract_file();
+
 			/* Save state associated with the current buffer */
 			struct _yy_buffer *yb;
 
@@ -2548,6 +2792,9 @@ isinformixdefine(void)
 	if (new)
 	{
+		/* 清理可能存在的临时文件 */
+		cleanup_extract_file();
+
 		/* Save state associated with the current buffer */
 		struct _yy_buffer *yb;
 
@@ -2576,6 +2823,9 @@ isinformixdefine(void)
 					{
 						if (yy_buffer == NULL)
 						{
+							/* 退出主文件前清理临时文件 */
+							cleanup_extract_file();
+
 							/* No more input */
 							if (preproc_tos > 0)
 							{
@@ -2586,6 +2836,9 @@ isinformixdefine(void)
 						}
 						else
 						{
+							/* 切换文件前清理临时文件 */
+							cleanup_extract_file();
+
 							/* Revert to previous input source */
 							struct _yy_buffer *yb = yy_buffer;
 							int			i;
```

---

### 关于 `debug_log` 的初始化

当前 patch 中 `debug_log = 0` 是硬编码。如果你已有 `-D` 参数解析，请找到 ecpg 主入口（如 `main()` 或 `ecpg.c`）中处理 `-D` 的地方，添加：

```c
extern int debug_log;  /* 在 pgc.l 中定义 */

/* 处理 -D 参数时 */
if (strcmp(optarg, "DEBUG") == 0 || strcmp(optarg, "debug") == 0)
    debug_log = 1;
```

或者如果你有现有的全局变量名（如 `verbose` / `debug` 等），告诉我，我可以替换 `debug_log`。

---

### 验证用例

**h1.h**：
```c
#ifndef H1_H
#define H1_H

// global vars
int global_a;
unsigned int global_b;
signed int global_c;

extern int ext_val;          /* 过滤 */
void helper(void);            /* 过滤 */
int arr[10];
int arr2[20][10];
int initialized = 100;        /* 保留 */

#endif
```

**a1.pc**：
```c
#include "h1.h"

int main()
{
    EXEC SQL drop table t1;
    EXEC SQL create table t1(id1 int, id2 varchar(20));
    EXEC SQL insert into t1 values(1, 'a1');
    EXEC SQL select id1 into :global_a from t1;
}
```

**执行**：
```bash
ecpg a1.pc
```

**预期行为**：
- `h1.h.extract` 生成，包含 `BEGIN DECLARE SECTION` + 提取的 `int` 变量 + `END DECLARE SECTION`
- `a1.c` 正常生成，包含对 `global_a` 等变量的宿主变量绑定代码
- 非调试模式下，`ecpg` 完成后 `h1.h.extract` 被删除

请验证后反馈，没问题继续第二步 `char` 类型。


# 生成的临时文件 行内容的缩进 也要和原文件对齐

明白了，临时文件中提取的行内容缩进要和原文件保持一致。

当前 patch 已经是这样做的——`is_int_var_line()` 只判断是否提取，提取时通过 `fprintf(dst, "%s", line)` 原样写入整行（包括行首缩进和尾部换行）。

但有一个问题：**`is_int_var_line()` 在判断时跳过了行首空白**，但写入时保留原行。这没问题，因为判断逻辑修改的是局部指针 `p`，不影响原 `line` 内容。

不过我需要确认一点：当 `unsigned int` 或 `signed int` 前面有缩进时，`is_int_var_line()` 中 `p` 已经前移到 `unsigned`/`signed` 位置，判断逻辑正确。写入时 `line` 仍包含原始缩进。

让我再检查一下 patch 中是否有问题...

实际上，当前 patch 的 `extract_host_vars()` 中：
```c
while (fgets(line, sizeof(line), src))
{
    if (is_int_var_line(line))
    {
        fprintf(dst, "%s", line);  // 原样写入，保留缩进
    }
    else
    {
        fprintf(dst, "\n");  // 空行占位
    }
}
```

这里 `line` 是 `fgets` 读取的原始行（包含 `\n`），`fprintf(dst, "%s", line)` 会原样输出，**缩进已经保留**。

所以当前 patch 已经满足"缩进对齐"的需求，不需要额外修改。

但让我再优化一下 patch，确保 `is_int_var_line()` 中的指针移动不会意外修改原 `line`（当前是 `const char *p = line`，只读，安全）。

下面是优化后的最终 patch（和之前基本一致，确认缩进保留逻辑）：

```diff
--- a/src/interfaces/ecpg/preproc/pgc.l
+++ b/src/interfaces/ecpg/preproc/pgc.l
@@ -79,6 +79,8 @@ static void parse_include(void);
 static bool ecpg_isspace(char ch);
 static bool isdefine(void);
 static bool isinformixdefine(void);
+static bool extract_host_vars(const char *srcfile, const char *dstfile);
+static int debug_log = 0;  /* 对应 ecpg -D 调试模式，=1 时不删除临时文件 */
 
 char	   *token_start;
 
@@ -2360,6 +2362,8 @@ parse_include(void)
 	struct _yy_buffer *yb;
 	struct _include_path *ip;
 	char		inc_file[MAXPGPATH];
+	char		extract_file[MAXPGPATH];
+	bool		use_extract = false;
 	unsigned int i;
 
 	yb = mm_alloc(sizeof(struct _yy_buffer));
@@ -2403,6 +2407,16 @@ parse_include(void)
 		memmove(yytext, yytext + 1, strlen(yytext));
 
 		strlcpy(inc_file, yytext, sizeof(inc_file));
+
+		/* 对双引号包含的头文件尝试提取宿主变量 */
+		snprintf(extract_file, sizeof(extract_file), "%s.extract", inc_file);
+		if (extract_host_vars(inc_file, extract_file))
+		{
+			use_extract = true;
+		}
+		else
+		{
+			use_extract = false;
+		}
+
 		yyin = fopen(inc_file, "r");
 		if (!yyin)
 		{
@@ -2453,6 +2467,21 @@ parse_include(void)
 	if (!yyin)
 		mmfatal(NO_INCLUDE_FILE, "could not open include file \"%s\" on line %d", yytext, yylineno);
 
+	/* 如果提取成功，切换到临时文件 */
+	if (use_extract)
+	{
+		fclose(yyin);
+		yyin = fopen(extract_file, "r");
+		if (!yyin)
+			mmfatal(NO_INCLUDE_FILE, "could not open extract file \"%s\" on line %d", extract_file, yylineno);
+		input_filename = mm_strdup(extract_file);
+	}
+	else
+	{
+		input_filename = mm_strdup(inc_file);
+	}
+
-	input_filename = mm_strdup(inc_file);
 	yy_switch_to_buffer(yy_create_buffer(yyin, YY_BUF_SIZE));
 	yylineno = 1;
 	output_line_number();
@@ -2471,6 +2500,228 @@ parse_include(void)
 	BEGIN(C);
 }
 
+/*
+ * is_int_var_line --- 判断一行是否为 int 类型变量定义（含数组）
+ *
+ * 支持: int, unsigned int, signed int 变量定义（含数组形式）
+ * 过滤: extern, 函数声明(含'('), 预处理指令, 注释
+ *
+ * 返回: true 表示需要提取，false 表示跳过
+ */
+static bool
+is_int_var_line(const char *line)
+{
+	const char *p = line;  /* 只读指针，不修改原 line */
+	bool is_extern = false;
+	bool is_int = false;
+	bool has_paren = false;
+
+	/* 跳过行首空白 */
+	while (*p && (*p == ' ' || *p == '\t'))
+		p++;
+
+	/* 空行 */
+	if (*p == '\n' || *p == '\r' || *p == '\0')
+		return false;
+
+	/* 跳过预处理指令 */
+	if (*p == '#')
+		return false;
+
+	/* 跳过 // 注释 */
+	if (p[0] == '/' && p[1] == '/')
+		return false;
+
+	/* 跳过 /* 块注释开始 */
+	if (p[0] == '/' && p[1] == '*')
+		return false;
+
+	/* 检查 extern */
+	if ((p[0] == 'e' || p[0] == 'E') &&
+		(p[1] == 'x' || p[1] == 'X') &&
+		(p[2] == 't' || p[2] == 'T') &&
+		(p[3] == 'e' || p[3] == 'E') &&
+		(p[4] == 'r' || p[4] == 'R') &&
+		(p[5] == 'n' || p[5] == 'N') &&
+		(p[6] == ' ' || p[6] == '\t'))
+	{
+		is_extern = true;
+	}
+
+	/* 检查 unsigned / signed */
+	if (!is_extern)
+	{
+		if ((p[0] == 'u' || p[0] == 'U') &&
+			(p[1] == 'n' || p[1] == 'N') &&
+			(p[2] == 's' || p[2] == 'S') &&
+			(p[3] == 'i' || p[3] == 'I') &&
+			(p[4] == 'g' || p[4] == 'G') &&
+			(p[5] == 'n' || p[5] == 'N') &&
+			(p[6] == 'e' || p[6] == 'E') &&
+			(p[7] == 'd' || p[7] == 'D') &&
+			(p[8] == ' ' || p[8] == '\t'))
+		{
+			p += 9;
+			while (*p && (*p == ' ' || *p == '\t'))
+				p++;
+		}
+		else if ((p[0] == 's' || p[0] == 'S') &&
+				 (p[1] == 'i' || p[1] == 'I') &&
+				 (p[2] == 'g' || p[2] == 'G') +
+				 (p[3] == 'n' || p[3] == 'N') &&
+				 (p[4] == 'e' || p[4] == 'E') &&
+				 (p[5] == 'd' || p[5] == 'D') &&
+				 (p[6] == ' ' || p[6] == '\t'))
+		{
+			p += 7;
+			while (*p && (*p == ' ' || *p == '\t'))
+				p++;
+		}
+	}
+
+	/* 检查 int */
+	if (!is_extern)
+	{
+		if ((p[0] == 'i' || p[0] == 'I') &&
+			(p[1] == 'n' || p[1] == 'N') &&
+			(p[2] == 't' || p[2] == 'T') &&
+			(p[3] == ' ' || p[3] == '\t' || p[3] == '\n' || p[3] == '\r' ||
+			 p[3] == ';' || p[3] == '\0'))
+		{
+			is_int = true;
+		}
+	}
+
+	/* 检查是否包含 '(' —— 函数声明的特征 */
+	if (is_int && !is_extern)
+	{
+		const char *q = p + 3;
+		while (*q && *q != '\n' && *q != '\r')
+		{
+			if (*q == '(')
+			{
+				has_paren = true;
+				break;
+			}
+			q++;
+		}
+	}
+
+	/* 提取条件：是 int 类型定义，不是 extern，不是函数声明 */
+	return (is_int && !is_extern && !has_paren);
+}
+
+/*
+ * extract_host_vars --- 从头文件中提取 int 类型宿主变量定义
+ *
+ * 行号对齐：不提取的行用空行占位，保持与原文件行号一致
+ * 缩进对齐：提取的行原样输出，保留原始缩进
+ * 返回 true 表示成功提取到内容，false 表示无内容或失败
+ */
+static bool
+extract_host_vars(const char *srcfile, const char *dstfile)
+{
+	FILE *src, *dst;
+	char line[4096];
+	bool has_content = false;
+
+	src = fopen(srcfile, "r");
+	if (!src)
+		return false;
+
+	dst = fopen(dstfile, "w");
+	if (!dst)
+	{
+		fclose(src);
+		return false;
+	}
+
+	/* 写入 BEGIN DECLARE SECTION（占第1行） */
+	fprintf(dst, "EXEC SQL BEGIN DECLARE SECTION;\n");
+
+	while (fgets(line, sizeof(line), src))
+	{
+		if (is_int_var_line(line))
+		{
+			/* 原样输出，保留原始缩进 */
+			fprintf(dst, "%s", line);
+			has_content = true;
+		}
+		else
+		{
+			/* 不提取的行用空行占位，保持行号对齐 */
+			fprintf(dst, "\n");
+		}
+	}
+
+	/* 写入 END DECLARE SECTION */
+	fprintf(dst, "EXEC SQL END DECLARE SECTION;\n");
+
+	fclose(src);
+	fclose(dst);
+
+	/* 如果没有提取到任何内容，删除临时文件 */
+	if (!has_content)
+	{
+		remove(dstfile);
+		return false;
+	}
+
+	return true;
+}
+
+/*
+ * cleanup_extract_file --- 清理临时 .extract 文件
+ *
+ * debug_log = 1 时不删除（调试模式保留）
+ */
+static void
+cleanup_extract_file(void)
+{
+	if (debug_log)
+		return;
+
+	if (input_filename != NULL)
+	{
+		size_t len = strlen(input_filename);
+		if (len > 8 && strcmp(input_filename + len - 8, ".extract") == 0)
+		{
+			remove(input_filename);
+		}
+	}
+}
+
 /*
  * ecpg_isspace() --- return true if flex scanner considers char whitespace
  */
@@ -2498,6 +2739,9 @@ isdefine(void)
 			ptr->value != NULL &&
 			ptr->used == NULL)
 		{
+			/* 清理可能存在的临时文件 */
+			cleanup_extract_file();
+
 			/* Save state associated with the current buffer */
 			struct _yy_buffer *yb;
 
@@ -2548,6 +2792,9 @@ isinformixdefine(void)
 	if (new)
 	{
+		/* 清理可能存在的临时文件 */
+		cleanup_extract_file();
+
 		/* Save state associated with the current buffer */
 		struct _yy_buffer *yb;
 
@@ -2576,6 +2823,9 @@ isinformixdefine(void)
 					{
 						if (yy_buffer == NULL)
 						{
+							/* 退出主文件前清理临时文件 */
+							cleanup_extract_file();
+
 							/* No more input */
 							if (preproc_tos > 0)
 							{
@@ -2586,6 +2836,9 @@ isinformixdefine(void)
 						}
 						else
 						{
+							/* 切换文件前清理临时文件 */
+							cleanup_extract_file();
+
 							/* Revert to previous input source */
 							struct _yy_buffer *yb = yy_buffer;
 							int			i;
```

### 生成的临时文件示例

**h1.h**：
```c
#ifndef H1_H
#define H1_H

    // 带缩进的注释
    int global_a;
    unsigned int global_b;
        signed int global_c;

    extern int ext_val;
    void helper(void);
    int arr[10];
        int arr2[20][10];
    int initialized = 100;

#endif
```

**h1.h.extract**：
```c
EXEC SQL BEGIN DECLARE SECTION;
                              /* 第1行：#ifndef 跳过，空行 */
                              /* 第2行：#define 跳过，空行 */
                              /* 第3行：空行跳过，空行 */
                              /* 第4行：// 注释跳过，空行 */
    int global_a;             /* 第5行：提取，保留4空格缩进 */
    unsigned int global_b;    /* 第6行：提取，保留4空格缩进 */
        signed int global_c;  /* 第7行：提取，保留8空格缩进 */
                              /* 第8行：空行跳过，空行 */
                              /* 第9行：extern 过滤，空行 */
                              /* 第10行：函数声明过滤，空行 */
    int arr[10];              /* 第11行：提取，保留4空格缩进 */
        int arr2[20][10];     /* 第12行：提取，保留8空格缩进 */
    int initialized = 100;    /* 第13行：提取，保留4空格缩进 */
                              /* 第14行：空行跳过，空行 */
                              /* 第15行：#endif 跳过，空行 */
EXEC SQL END DECLARE SECTION;
```

缩进和原文件完全一致。请验证。




# 头文件中有 

int func1()

{

  int x1;

   int x2;

   return x1 +x2;

}

这种会被提取吗？ 按要求应该不会被提取 int 类型变量定义

》
好的，下面是修正后的完整 patch（增加大括号层级判断，只提取全局作用域变量）：

```diff
--- a/src/interfaces/ecpg/preproc/pgc.l
+++ b/src/interfaces/ecpg/preproc/pgc.l
@@ -79,6 +79,8 @@ static void parse_include(void);
 static bool ecpg_isspace(char ch);
 static bool isdefine(void);
 static bool isinformixdefine(void);
+static bool extract_host_vars(const char *srcfile, const char *dstfile);
+static int debug_log = 0;  /* 对应 ecpg -D 调试模式，=1 时不删除临时文件 */
 
 char	   *token_start;
 
@@ -2360,6 +2362,8 @@ parse_include(void)
 	struct _yy_buffer *yb;
 	struct _include_path *ip;
 	char		inc_file[MAXPGPATH];
+	char		extract_file[MAXPGPATH];
+	bool		use_extract = false;
 	unsigned int i;
 
 	yb = mm_alloc(sizeof(struct _yy_buffer));
@@ -2403,6 +2407,16 @@ parse_include(void)
 		memmove(yytext, yytext + 1, strlen(yytext));
 
 		strlcpy(inc_file, yytext, sizeof(inc_file));
+
+		/* 对双引号包含的头文件尝试提取宿主变量 */
+		snprintf(extract_file, sizeof(extract_file), "%s.extract", inc_file);
+		if (extract_host_vars(inc_file, extract_file))
+		{
+			use_extract = true;
+		}
+		else
+		{
+			use_extract = false;
+		}
+
 		yyin = fopen(inc_file, "r");
 		if (!yyin)
 		{
@@ -2453,6 +2467,21 @@ parse_include(void)
 	if (!yyin)
 		mmfatal(NO_INCLUDE_FILE, "could not open include file \"%s\" on line %d", yytext, yylineno);
 
+	/* 如果提取成功，切换到临时文件 */
+	if (use_extract)
+	{
+		fclose(yyin);
+		yyin = fopen(extract_file, "r");
+		if (!yyin)
+			mmfatal(NO_INCLUDE_FILE, "could not open extract file \"%s\" on line %d", extract_file, yylineno);
+		input_filename = mm_strdup(extract_file);
+	}
+	else
+	{
+		input_filename = mm_strdup(inc_file);
+	}
+
-	input_filename = mm_strdup(inc_file);
 	yy_switch_to_buffer(yy_create_buffer(yyin, YY_BUF_SIZE));
 	yylineno = 1;
 	output_line_number();
@@ -2471,6 +2500,240 @@ parse_include(void)
 	BEGIN(C);
 }
 
+/*
+ * is_int_var_line --- 判断一行是否为 int 类型变量定义（含数组）
+ *
+ * 支持: int, unsigned int, signed int 全局变量定义（含数组形式）
+ * 过滤: extern, 函数声明(含'('), 预处理指令, 注释
+ *
+ * 返回: true 表示需要提取，false 表示跳过
+ */
+static bool
+is_int_var_line(const char *line)
+{
+	const char *p = line;  /* 只读指针，不修改原 line */
+	bool is_extern = false;
+	bool is_int = false;
+	bool has_paren = false;
+
+	/* 跳过行首空白 */
+	while (*p && (*p == ' ' || *p == '\t'))
+		p++;
+
+	/* 空行 */
+	if (*p == '\n' || *p == '\r' || *p == '\0')
+		return false;
+
+	/* 跳过预处理指令 */
+	if (*p == '#')
+		return false;
+
+	/* 跳过 // 注释 */
+	if (p[0] == '/' && p[1] == '/')
+		return false;
+
+	/* 跳过 /* 块注释开始 */
+	if (p[0] == '/' && p[1] == '*')
+		return false;
+
+	/* 检查 extern */
+	if ((p[0] == 'e' || p[0] == 'E') &&
+		(p[1] == 'x' || p[1] == 'X') &&
+		(p[2] == 't' || p[2] == 'T') &&
+		(p[3] == 'e' || p[3] == 'E') &&
+		(p[4] == 'r' || p[4] == 'R') &&
+		(p[5] == 'n' || p[5] == 'N') &&
+		(p[6] == ' ' || p[6] == '\t'))
+	{
+		is_extern = true;
+	}
+
+	/* 检查 unsigned / signed */
+	if (!is_extern)
+	{
+		if ((p[0] == 'u' || p[0] == 'U') &&
+			(p[1] == 'n' || p[1] == 'N') &&
+			(p[2] == 's' || p[2] == 'S') &&
+			(p[3] == 'i' || p[3] == 'I') &&
+			(p[4] == 'g' || p[4] == 'G') &&
+			(p[5] == 'n' || p[5] == 'N') &&
+			(p[6] == 'e' || p[6] == 'E') &&
+			(p[7] == 'd' || p[7] == 'D') &&
+			(p[8] == ' ' || p[8] == '\t'))
+		{
+			p += 9;
+			while (*p && (*p == ' ' || *p == '\t'))
+				p++;
+		}
+		else if ((p[0] == 's' || p[0] == 'S') &&
+				 (p[1] == 'i' || p[1] == 'I') &&
+				 (p[2] == 'g' || p[2] == 'G') &&
+				 (p[3] == 'n' || p[3] == 'N') &&
+				 (p[4] == 'e' || p[4] == 'E') &&
+				 (p[5] == 'd' || p[5] == 'D') &&
+				 (p[6] == ' ' || p[6] == '\t'))
+		{
+			p += 7;
+			while (*p && (*p == ' ' || *p == '\t'))
+				p++;
+		}
+	}
+
+	/* 检查 int */
+	if (!is_extern)
+	{
+		if ((p[0] == 'i' || p[0] == 'I') &&
+			(p[1] == 'n' || p[1] == 'N') &&
+			(p[2] == 't' || p[2] == 'T') &&
+			(p[3] == ' ' || p[3] == '\t' || p[3] == '\n' || p[3] == '\r' ||
+			 p[3] == ';' || p[3] == '\0'))
+		{
+			is_int = true;
+		}
+	}
+
+	/* 检查是否包含 '(' —— 函数声明的特征 */
+	if (is_int && !is_extern)
+	{
+		const char *q = p + 3;
+		while (*q && *q != '\n' && *q != '\r')
+		{
+			if (*q == '(')
+			{
+				has_paren = true;
+				break;
+			}
+			q++;
+		}
+	}
+
+	/* 提取条件：是 int 类型定义，不是 extern，不是函数声明 */
+	return (is_int && !is_extern && !has_paren);
+}
+
+/*
+ * extract_host_vars --- 从头文件中提取 int 类型宿主变量定义
+ *
+ * 只提取全局作用域（大括号层级为0）的变量定义
+ * 行号对齐：不提取的行用空行占位，保持与原文件行号一致
+ * 缩进对齐：提取的行原样输出，保留原始缩进
+ * 返回 true 表示成功提取到内容，false 表示无内容或失败
+ */
+static bool
+extract_host_vars(const char *srcfile, const char *dstfile)
+{
+	FILE *src, *dst;
+	char line[4096];
+	bool has_content = false;
+	int brace_level = 0;  /* 大括号层级，只提取层级为0的变量 */
+
+	src = fopen(srcfile, "r");
+	if (!src)
+		return false;
+
+	dst = fopen(dstfile, "w");
+	if (!dst)
+	{
+		fclose(src);
+		return false;
+	}
+
+	/* 写入 BEGIN DECLARE SECTION（占第1行） */
+	fprintf(dst, "EXEC SQL BEGIN DECLARE SECTION;\n");
+
+	while (fgets(line, sizeof(line), src))
+	{
+		char *p;
+		bool is_int_var = false;
+
+		/* 更新大括号层级（在判断 int 变量之前） */
+		for (p = line; *p; p++)
+		{
+			if (*p == '{')
+				brace_level++;
+			else if (*p == '}')
+				brace_level--;
+		}
+
+		/* 只在大括号层级为0时检查是否为 int 变量定义 */
+		if (brace_level == 0 && is_int_var_line(line))
+		{
+			/* 原样输出，保留原始缩进 */
+			fprintf(dst, "%s", line);
+			has_content = true;
+		}
+		else
+		{
+			/* 不提取的行用空行占位，保持行号对齐 */
+			fprintf(dst, "\n");
+		}
+	}
+
+	/* 写入 END DECLARE SECTION */
+	fprintf(dst, "EXEC SQL END DECLARE SECTION;\n");
+
+	fclose(src);
+	fclose(dst);
+
+	/* 如果没有提取到任何内容，删除临时文件 */
+	if (!has_content)
+	{
+		remove(dstfile);
+		return false;
+	}
+
+	return true;
+}
+
+/*
+ * cleanup_extract_file --- 清理临时 .extract 文件
+ *
+ * debug_log = 1 时不删除（调试模式保留）
+ */
+static void
+cleanup_extract_file(void)
+{
+	if (debug_log)
+		return;
+
+	if (input_filename != NULL)
+	{
+		size_t len = strlen(input_filename);
+		if (len > 8 && strcmp(input_filename + len - 8, ".extract") == 0)
+		{
+			remove(input_filename);
+		}
+	}
+}
+
 /*
  * ecpg_isspace() --- return true if flex scanner considers char whitespace
  */
@@ -2498,6 +2759,9 @@ isdefine(void)
 			ptr->value != NULL &&
 			ptr->used == NULL)
 		{
+			/* 清理可能存在的临时文件 */
+			cleanup_extract_file();
+
 			/* Save state associated with the current buffer */
 			struct _yy_buffer *yb;
 
@@ -2548,6 +2812,9 @@ isinformixdefine(void)
 	if (new)
 	{
+		/* 清理可能存在的临时文件 */
+		cleanup_extract_file();
+
 		/* Save state associated with the current buffer */
 		struct _yy_buffer *yb;
 
@@ -2576,6 +2843,9 @@ isinformixdefine(void)
 					{
 						if (yy_buffer == NULL)
 						{
+							/* 退出主文件前清理临时文件 */
+							cleanup_extract_file();
+
 							/* No more input */
 							if (preproc_tos > 0)
 							{
@@ -2586,6 +2856,9 @@ isinformixdefine(void)
 						}
 						else
 						{
+							/* 切换文件前清理临时文件 */
+							cleanup_extract_file();
+
 							/* Revert to previous input source */
 							struct _yy_buffer *yb = yy_buffer;
 							int			i;
```

### 验证场景

**h1.h**：
```c
#ifndef H1_H
#define H1_H

int global_a;
unsigned int global_b;
    signed int global_c;

extern int ext_val;
void helper(void);
int arr[10];

int func1()
{
    int x1;          /* 不提取：在函数体内 */
    int x2;          /* 不提取：在函数体内 */
    return x1 + x2;
}

    int global_d;    /* 提取：函数体外，全局作用域 */

#endif
```

**h1.h.extract**：
```c
EXEC SQL BEGIN DECLARE SECTION;
                              /* 第1行：#ifndef 跳过 */
                              /* 第2行：#define 跳过 */
                              /* 第3行：空行跳过 */
int global_a;                 /* 第4行：提取 */
unsigned int global_b;        /* 第5行：提取 */
    signed int global_c;      /* 第6行：提取，保留缩进 */
                              /* 第7行：空行跳过 */
                              /* 第8行：extern 过滤 */
                              /* 第9行：函数声明过滤 */
int arr[10];                  /* 第10行：提取 */
                              /* 第11行：int func1() 含'(' 过滤 */
{                             /* 第12行：{ 不是 int 定义 */
                              /* 第13行：x1 层级=1 不提取 */
                              /* 第14行：x2 层级=1 不提取 */
                              /* 第15行：return 不是 int 定义 */
}                             /* 第16行：} 不是 int 定义 */
                              /* 第17行：空行跳过 */
    int global_d;             /* 第18行：提取，层级=0 */
                              /* 第19行：#endif 跳过 */
EXEC SQL END DECLARE SECTION;
```

函数体内的 `int x1;` / `int x2;` 因 `brace_level=1` 不提取。请验证。


# 上面改的方案 我觉的有问题。 比如pc 文件中写的 #include "xx1.h" 现在提取信息到 xx1.h.extract , 如果有 #include "xx2.h" 也完全复制过来。现在使用parse_include 处理 xx1.h.extract ， 读取到 #include  “xx2.h”文件内容, 则打开这个xx2.h 文件内容 进行提取， 然后使用 parse_include 处理 xx2.h.extract 。

你看是不是应该这样处理


确认了，方案 B：

- `xx1.h.extract` 包含 `BEGIN/END`，其中 `#include "xx2.h"` 原样保留
- ECPG 处理到 `#include "xx2.h"` 时，`parse_include()` 打开 `xx2.h`
- 生成 `xx2.h.extract`（**不包含** `BEGIN/END`）
- 打开 `xx2.h.extract`

实现方式：`extract_host_vars()` 增加参数 `add_wrapper`，顶层调用为 `true`，嵌套调用为 `false`。

但 `parse_include()` 如何区分顶层和嵌套？

**方案**：`parse_include()` 中，根据当前打开的文件名判断：
- 如果当前文件名以 `.extract` 结尾 → 说明已经在处理 `.extract` 文件，是嵌套调用 → `add_wrapper = false`
- 否则 → 是顶层调用 → `add_wrapper = true`

不对，`parse_include()` 被调用时，当前文件还没打开，是准备打开新文件。

**正确方案**：`parse_include()` 中，生成 `.extract` 文件时，根据**被包含文件的原文件名**判断：
- 如果原文件是 `.pc` 文件直接包含的（即 `yytext` 中的文件名不以 `.extract` 结尾）→ 顶层 → `add_wrapper = true`
- 如果原文件是 `.extract` 文件中的 `#include`（即 `yytext` 中的文件名不以 `.extract` 结尾，但当前 `input_filename` 以 `.extract` 结尾）→ 嵌套 → `add_wrapper = false`

实际上更简单：`parse_include()` 中，检查当前 `input_filename`：
- `input_filename` 以 `.extract` 结尾 → 已经在 `.extract` 中 → 嵌套 → `add_wrapper = false`
- `input_filename` 是 `.pc` 文件 → 顶层 → `add_wrapper = true`

但 `input_filename` 在 `parse_include()` 被调用时，还是当前文件（`.pc` 或 `.extract`）。

**最终实现**：

```c
/* parse_include() 中 */
bool add_wrapper = true;
if (input_filename != NULL)
{
    size_t len = strlen(input_filename);
    if (len > 8 && strcmp(input_filename + len - 8, ".extract") == 0)
        add_wrapper = false;  /* 已经在 .extract 文件中，嵌套包含 */
}
```

下面是完整 patch：

```diff
--- a/src/interfaces/ecpg/preproc/pgc.l
+++ b/src/interfaces/ecpg/preproc/pgc.l
@@ -79,6 +79,9 @@ static void parse_include(void);
 static bool ecpg_isspace(char ch);
 static bool isdefine(void);
 static bool isinformixdefine(void);
+static bool extract_host_vars(const char *srcfile, const char *dstfile, bool add_wrapper);
+static bool is_int_var_line(const char *line, bool *is_include);
+static int debug_log = 0;  /* 对应 ecpg -D 调试模式，=1 时不删除临时文件 */
 
 char	   *token_start;
 
@@ -2360,6 +2363,8 @@ parse_include(void)
 	struct _yy_buffer *yb;
 	struct _include_path *ip;
 	char		inc_file[MAXPGPATH];
+	char		extract_file[MAXPGPATH];
+	bool		use_extract = false;
 	unsigned int i;
 
 	yb = mm_alloc(sizeof(struct _yy_buffer));
@@ -2403,6 +2408,20 @@ parse_include(void)
 		memmove(yytext, yytext + 1, strlen(yytext));
 
 		strlcpy(inc_file, yytext, sizeof(inc_file));
+
+		/* 对双引号包含的头文件尝试提取宿主变量 */
+		snprintf(extract_file, sizeof(extract_file), "%s.extract", inc_file);
+		{
+			bool add_wrapper = true;
+			if (input_filename != NULL)
+			{
+				size_t len = strlen(input_filename);
+				if (len > 8 && strcmp(input_filename + len - 8, ".extract") == 0)
+					add_wrapper = false;  /* 嵌套包含，不加 BEGIN/END */
+			}
+			if (extract_host_vars(inc_file, extract_file, add_wrapper))
+				use_extract = true;
+		}
+
 		yyin = fopen(inc_file, "r");
 		if (!yyin)
 		{
@@ -2453,6 +2472,21 @@ parse_include(void)
 	if (!yyin)
 		mmfatal(NO_INCLUDE_FILE, "could not open include file \"%s\" on line %d", yytext, yylineno);
 
+	/* 如果提取成功，切换到临时文件 */
+	if (use_extract)
+	{
+		fclose(yyin);
+		yyin = fopen(extract_file, "r");
+		if (!yyin)
+			mmfatal(NO_INCLUDE_FILE, "could not open extract file \"%s\" on line %d", extract_file, yylineno);
+		input_filename = mm_strdup(extract_file);
+	}
+	else
+	{
+		input_filename = mm_strdup(inc_file);
+	}
+
-	input_filename = mm_strdup(inc_file);
 	yy_switch_to_buffer(yy_create_buffer(yyin, YY_BUF_SIZE));
 	yylineno = 1;
 	output_line_number();
@@ -2471,6 +2505,224 @@ parse_include(void)
 	BEGIN(C);
 }
 
+/*
+ * is_int_var_line --- 判断一行是否为 int 类型变量定义（含数组）
+ *
+ * 支持: int, unsigned int, signed int 全局变量定义（含数组形式）
+ * 过滤: extern, 函数声明(含'('), 预处理指令(除#include外), 注释
+ *
+ * 返回: true 表示需要提取，false 表示跳过
+ */
+static bool
+is_int_var_line(const char *line, bool *is_include)
+{
+	const char *p = line;
+	bool is_extern = false;
+	bool is_int = false;
+	bool has_paren = false;
+
+	*is_include = false;
+
+	/* 跳过行首空白 */
+	while (*p && (*p == ' ' || *p == '\t'))
+		p++;
+
+	/* 空行 */
+	if (*p == '\n' || *p == '\r' || *p == '\0')
+		return false;
+
+	/* 预处理指令 */
+	if (*p == '#')
+	{
+		const char *q = p + 1;
+		while (*q && (*q == ' ' || *q == '\t'))
+			q++;
+
+		/* 检查是否为 #include "..." */
+		if ((q[0] == 'i' || q[0] == 'I') &&
+			(q[1] == 'n' || q[1] == 'N') &&
+			(q[2] == 'c' || q[2] == 'C') &&
+			(q[3] == 'l' || q[3] == 'L') &&
+			(q[4] == 'u' || q[4] == 'U') &&
+			(q[5] == 'd' || q[5] == 'D') &&
+			(q[6] == 'e' || q[6] == 'E') &&
+			(q[7] == ' ' || q[7] == '\t'))
+		{
+			q += 8;
+			while (*q && (*q == ' ' || *q == '\t'))
+				q++;
+			if (*q == '"')
+			{
+				*is_include = true;
+				return true;
+			}
+		}
+		return false;
+	}
+
+	/* 跳过 // 注释 */
+	if (p[0] == '/' && p[1] == '/')
+		return false;
+
+	/* 跳过 /* 块注释开始 */
+	if (p[0] == '/' && p[1] == '*')
+		return false;
+
+	/* 检查 extern */
+	if ((p[0] == 'e' || p[0] == 'E') &&
+		(p[1] == 'x' || p[1] == 'X') &&
+		(p[2] == 't' || p[2] == 'T') &&
+		(p[3] == 'e' || p[3] == 'E') &&
+		(p[4] == 'r' || p[4] == 'R') &&
+		(p[5] == 'n' || p[5] == 'N') &&
+		(p[6] == ' ' || p[6] == '\t'))
+	{
+		is_extern = true;
+	}
+
+	/* 检查 unsigned / signed */
+	if (!is_extern)
+	{
+		if ((p[0] == 'u' || p[0] == 'U') &&
+			(p[1] == 'n' || p[1] == 'N') &&
+			(p[2] == 's' || p[2] == 'S') &&
+			(p[3] == 'i' || p[3] == 'I') &&
+			(p[4] == 'g' || p[4] == 'G') &&
+			(p[5] == 'n' || p[5] == 'N') &&
+			(p[6] == 'e' || p[6] == 'E') &&
+			(p[7] == 'd' || p[7] == 'D') &&
+			(p[8] == ' ' || p[8] == '\t'))
+		{
+			p += 9;
+			while (*p && (*p == ' ' || *p == '\t'))
+				p++;
+		}
+		else if ((p[0] == 's' || p[0] == 'S') &&
+				 (p[1] == 'i' || p[1] == 'I') &&
+				 (p[2] == 'g' || p[2] == 'G') &&
+				 (p[3] == 'n' || p[3] == 'N') &&
+				 (p[4] == 'e' || p[4] == 'E') &&
+				 (p[5] == 'd' || p[5] == 'D') &&
+				 (p[6] == ' ' || p[6] == '\t'))
+		{
+			p += 7;
+			while (*p && (*p == ' ' || *p == '\t'))
+				p++;
+		}
+	}
+
+	/* 检查 int */
+	if (!is_extern)
+	{
+		if ((p[0] == 'i' || p[0] == 'I') &&
+			(p[1] == 'n' || p[1] == 'N') &&
+			(p[2] == 't' || p[2] == 'T') &&
+			(p[3] == ' ' || p[3] == '\t' || p[3] == '\n' || p[3] == '\r' ||
+			 p[3] == ';' || p[3] == '\0'))
+		{
+			is_int = true;
+		}
+	}
+
+	/* 检查是否包含 '(' —— 函数声明的特征 */
+	if (is_int && !is_extern)
+	{
+		const char *q = p + 3;
+		while (*q && *q != '\n' && *q != '\r')
+		{
+			if (*q == '(')
+			{
+				has_paren = true;
+				break;
+			}
+			q++;
+		}
+	}
+
+	return (is_int && !is_extern && !has_paren);
+}
+
+/*
+ * extract_host_vars --- 提取 int 类型宿主变量定义到临时文件
+ *
+ * add_wrapper: true 添加 BEGIN/END DECLARE SECTION，false 不添加
+ * 行号对齐：不提取的行用空行占位
+ * 缩进对齐：提取的行原样输出
+ */
+static bool
+extract_host_vars(const char *srcfile, const char *dstfile, bool add_wrapper)
+{
+	FILE *src, *dst;
+	char line[4096];
+	bool has_content = false;
+	int brace_level = 0;
+
+	src = fopen(srcfile, "r");
+	if (!src)
+		return false;
+
+	dst = fopen(dstfile, "w");
+	if (!dst)
+	{
+		fclose(src);
+		return false;
+	}
+
+	if (add_wrapper)
+		fprintf(dst, "EXEC SQL BEGIN DECLARE SECTION;\n");
+
+	while (fgets(line, sizeof(line), src))
+	{
+		char *p;
+		bool is_include = false;
+		bool extract_line = false;
+
+		/* 更新大括号层级 */
+		for (p = line; *p; p++)
+		{
+			if (*p == '{')
+				brace_level++;
+			else if (*p == '}')
+				brace_level--;
+		}
+
+		if (brace_level == 0)
+			extract_line = is_int_var_line(line, &is_include);
+
+		if (extract_line)
+		{
+			/* #include 或 int 变量定义，原样输出 */
+			fprintf(dst, "%s", line);
+			has_content = true;
+		}
+		else
+		{
+			fprintf(dst, "\n");
+		}
+	}
+
+	if (add_wrapper)
+		fprintf(dst, "EXEC SQL END DECLARE SECTION;\n");
+
+	fclose(src);
+	fclose(dst);
+
+	if (!has_content)
+	{
+		remove(dstfile);
+		return false;
+	}
+
+	return true;
+}
+
+/*
+ * cleanup_extract_file --- 清理临时 .extract 文件
+ */
+static void
+cleanup_extract_file(void)
+{
+	if (debug_log)
+		return;
+
+	if (input_filename != NULL)
+	{
+		size_t len = strlen(input_filename);
+		if (len > 8 && strcmp(input_filename + len - 8, ".extract") == 0)
+		{
+			remove(input_filename);
+		}
+	}
+}
+
 /*
  * ecpg_isspace() --- return true if flex scanner considers char whitespace
  */
@@ -2498,6 +2750,9 @@ isdefine(void)
 			ptr->value != NULL &&
 			ptr->used == NULL)
 		{
+			/* 清理可能存在的临时文件 */
+			cleanup_extract_file();
+
 			/* Save state associated with the current buffer */
 			struct _yy_buffer *yb;
 
@@ -2548,6 +2803,9 @@ isinformixdefine(void)
 	if (new)
 	{
+		/* 清理可能存在的临时文件 */
+		cleanup_extract_file();
+
 		/* Save state associated with the current buffer */
 		struct _yy_buffer *yb;
 
@@ -2576,6 +2834,9 @@ isinformixdefine(void)
 					{
 						if (yy_buffer == NULL)
 						{
+							/* 退出主文件前清理临时文件 */
+							cleanup_extract_file();
+
 							/* No more input */
 							if (preproc_tos > 0)
 							{
@@ -2586,6 +2846,9 @@ isinformixdefine(void)
 						}
 						else
 						{
+							/* 切换文件前清理临时文件 */
+							cleanup_extract_file();
+
 							/* Revert to previous input source */
 							struct _yy_buffer *yb = yy_buffer;
 							int			i;
```

### 关键逻辑

1. **`parse_include()` 中判断嵌套**：
   ```c
   bool add_wrapper = true;
   if (input_filename != NULL)
   {
       size_t len = strlen(input_filename);
       if (len > 8 && strcmp(input_filename + len - 8, ".extract") == 0)
           add_wrapper = false;  /* 已经在 .extract 中，嵌套 */
   }
   ```

2. **`extract_host_vars()` 参数**：
   - `add_wrapper = true`：`xx1.h.extract` 包含 `BEGIN/END`
   - `add_wrapper = false`：`xx2.h.extract` 不包含 `BEGIN/END`

3. **`#include` 处理**：
   - `xx1.h.extract` 中保留原样 `#include "xx2.h"`
   - ECPG 处理时，`parse_include()` 打开 `xx2.h`，生成 `xx2.h.extract`（无 `BEGIN/END`），然后打开 `xx2.h.extract`

### 生成文件示例

**h1.h**：
```c
#ifndef H1_H
#define H1_H

#include "h2.h"

int global_a;

#endif
```

**h1.h.extract**：
```c
EXEC SQL BEGIN DECLARE SECTION;
                              /* 第1行 */
                              /* 第2行 */
#include "h2.h"               /* 第3行：原样保留 */
int global_a;                 /* 第4行 */
                              /* 第5行 */
EXEC SQL END DECLARE SECTION;
```

**h2.h**：
```c
#ifndef H2_H
#define H2_H

int global_b;

#endif
```

**h2.h.extract**：
```c
                              /* 第1行 */
                              /* 第2行 */
int global_b;                 /* 第3行：无 BEGIN/END */
                              /* 第4行 */
```

ECPG 处理 `h1.h.extract` 时：
1. `BEGIN DECLARE SECTION` → 进入 SQL 状态
2. `#include "h2.h"` → `parse_include()` 打开 `h2.h`
3. 生成 `h2.h.extract`（`add_wrapper = false`，因为 `input_filename` 是 `h1.h.extract`）
4. 打开 `h2.h.extract`，`int global_b;` 在 `BEGIN/END` 上下文中被识别
5. 回到 `h1.h.extract`，`int global_a;` 被识别
6. `END DECLARE SECTION`

无嵌套问题。请验证。


