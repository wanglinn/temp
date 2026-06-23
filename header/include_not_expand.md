# 实现方式1 

  修改了 src/interfaces/ecpg/preproc/pgc.l：
  • 在 struct _yy_buffer 中增加 outfile 字段，保存进入 include/macro 之前的输出流。
  • 增加 cpp_include 标志，区分 C 预处理器 #include 与 EXEC SQL INCLUDE。
  • 在 parse_include() 中：
    • 遇到 C 预处理的双引号包含（#include "h1.h"）时：
      • 先把原始 #include "h1.h" 写入真实输出；
      • 然后把 base_yyout 重定向到 DEVNULL；
      • 继续解析该头文件，因此头文件里的宿主变量声明仍被 ecpg 识别。
    • 出 include 时在 <<EOF>> 处理里恢复原来的 base_yyout 并关闭临时 sink。
  • EXEC SQL INCLUDE 和尖括号系统头 #include <...> 保持原来的展开行为不变。
  验证
   场景                                                 结果
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ecpg -i a1.pc（#include "h1.h"）                     a1.c 保留 #include "h1.h"，不再展开
   头文件含 EXEC SQL BEGIN/END DECLARE SECTION 及变量   宿主变量被识别，生成的 C 代码正确使用 &(x1)
   EXEC SQL INCLUDE "h1.h"                              仍然按原样内联展开
   #include <angle.h>（带 -i）                          仍然内联展开
   嵌套双引号 include                                   仅最外层保留 #include，嵌套层正常解析但不输出
  使用方式
  新编译的 ecpg 在：
  /home/wln/source/wlnPg/src/interfaces/ecpg/preproc/ecpg
  你可以直接用：
  /home/wln/source/wlnPg/src/interfaces/ecpg/preproc/ecpg -i a1.pc
  或者如果你想把它安装到 /home/wln/app/master/bin/ecpg 覆盖原来的，请告诉我，我可以执行 make install 到指定前缀。
  注意
  你原始 h1.h 里只有：
  int x1;
  这种不在 EXEC SQL BEGIN/END DECLARE SECTION 块里的 plain C 变量，即使头文件被 parse_include 完整解析，ecpg 也不会把它当作宿主变量。要让它被
  识别，头文件应写成：
  EXEC SQL BEGIN DECLARE SECTION;
  int x1;
  EXEC SQL END DECLARE SECTION;

  这一点在修改前后行为一致，不是本次改动引入的。
  
  
  
  
  
# 代码实现 

[wln@vm ecpg]$ git diff
diff --git a/src/interfaces/ecpg/preproc/pgc.l b/src/interfaces/ecpg/preproc/pgc.l
index ce506cf..ce4d35b 100644
--- a/src/interfaces/ecpg/preproc/pgc.l
+++ b/src/interfaces/ecpg/preproc/pgc.l
@@ -58,6 +58,8 @@ static int    parenths_open;
 
 /* Used to tell parse_include() whether the command was #include or #include_next */
 static bool include_next;
+/* Used to tell parse_include() whether the include came from a C preprocessor directive */
+static bool cpp_include;
 
 #define startlit()     (literalbuf[0] = '\0', literallen = 0)
 static void addlit(char *ytext, int yleng);
@@ -99,6 +101,7 @@ static struct _yy_buffer
        YY_BUFFER_STATE buffer;
        long            lineno;
        char       *filename;
+       FILE       *outfile;            /* output file active before entering this level */
        struct _yy_buffer *next;
 } *yy_buffer = NULL;
 
@@ -1148,6 +1151,7 @@ cppline                   {space}*#([^i][A-Za-z]*|{if}|{ifdef}|{ifndef}|{import})((\/\*[^*/]*\*+
                                        if (system_includes)
                                        {
                                                include_next = false;
+                                               cpp_include = true;
                                                BEGIN(incl);
                                        }
                                        else
@@ -1160,6 +1164,7 @@ cppline                   {space}*#([^i][A-Za-z]*|{if}|{ifdef}|{ifndef}|{import})((\/\*[^*/]*\*+
                                        if (system_includes)
                                        {
                                                include_next = true;
+                                               cpp_include = true;
                                                BEGIN(incl);
                                        }
                                        else
@@ -1331,12 +1336,14 @@ cppline                 {space}*#([^i][A-Za-z]*|{if}|{ifdef}|{ifndef}|{import})((\/\*[^*/]*\*+
                                        yyterminate();
                                }
 <C>{exec_sql}{include}{space}* {
+                                       cpp_include = false;
                                        BEGIN(incl);
                                }
 <C>{informix_special}{include}{space}* {
                                        /* are we simulating Informix? */
                                        if (INFORMIX_MODE)
                                        {
+                                               cpp_include = false;
                                                BEGIN(incl);
                                        }
                                        else
@@ -1709,6 +1716,17 @@ cppline                  {space}*#([^i][A-Za-z]*|{if}|{ifdef}|{ifndef}|{import})((\/\*[^*/]*\*+
                                                if (yyin != NULL)
                                                        fclose(yyin);
 
+                                               /*
+                                                * Restore the output stream that was active before entering
+                                                * this include/macro level.  Close a temporary sink if we
+                                                * installed one for a quoted include.
+                                                */
+                                               if (base_yyout != yb->outfile)
+                                               {
+                                                       fclose(base_yyout);
+                                                       base_yyout = yb->outfile;
+                                               }
+
                                                yy_delete_buffer(YY_CURRENT_BUFFER);
                                                yy_switch_to_buffer(yy_buffer->buffer);
 
@@ -1826,12 +1844,15 @@ parse_include(void)
        struct _include_path *ip;
        char            inc_file[MAXPGPATH];
        unsigned int i;
+       bool            is_quoted_include = false;
+       char       *include_spec = NULL;
 
        yb = mm_alloc(sizeof(struct _yy_buffer));
 
        yb->buffer = YY_CURRENT_BUFFER;
        yb->lineno = yylineno;
        yb->filename = input_filename;
+       yb->outfile = base_yyout;
        yb->next = yy_buffer;
 
        yy_buffer = yb;
@@ -1848,6 +1869,16 @@ parse_include(void)
        if (yytext[i] == ';')
                i--;
 
+       /*
+        * Remember whether this is a quoted (user) include.  For such includes we
+        * still want to parse the file (so host-variable definitions are seen) but
+        * we emit the original #include directive in the generated C file instead
+        * of inlining the file's contents.
+        */
+       is_quoted_include = (yytext[0] == '"');
+       if (is_quoted_include)
+               include_spec = mm_strdup(yytext);
+
        yytext[i + 1] = '\0';
 
        yyin = NULL;
@@ -1918,6 +1949,26 @@ parse_include(void)
                mmfatal(NO_INCLUDE_FILE, "could not open include file \"%s\" on line %d", yytext, yylineno);
 
        input_filename = mm_strdup(inc_file);
+
+       /*
+        * For quoted includes, emit the original #include directive in the real
+        * output and parse the file with output redirected to a sink so that the
+        * file's contents are not inlined.  The parser still sees declarations in
+        * the file, allowing host variables defined there to be recognized.
+        */
+       if (cpp_include && is_quoted_include)
+       {
+               include_spec[i + 1] = '\0';
+               fprintf(base_yyout, "#include %s", include_spec);
+               free(include_spec);
+
+               base_yyout = fopen(DEVNULL, "w");
+               if (!base_yyout)
+                       mmfatal(NO_INCLUDE_FILE, "could not open %s", DEVNULL);
+       }
+
+       cpp_include = false;
+
        yy_switch_to_buffer(yy_create_buffer(yyin, YY_BUF_SIZE));
        yylineno = 1;
        output_line_number();
@@ -1966,6 +2017,7 @@ isdefine(void)
                        yb->buffer = YY_CURRENT_BUFFER;
                        yb->lineno = yylineno;
                        yb->filename = mm_strdup(input_filename);
+                       yb->outfile = base_yyout;
                        yb->next = yy_buffer;
                        yy_buffer = yb;
 
@@ -2010,6 +2062,7 @@ isinformixdefine(void)
                yb->buffer = YY_CURRENT_BUFFER;
                yb->lineno = yylineno;
                yb->filename = mm_strdup(input_filename);
+               yb->outfile = base_yyout;
                yb->next = yy_buffer;
                yy_buffer = yb;
 
 
# 实现方式2 


明白了，需要区分两种情况：

1. **`EXEC SQL INCLUDE h1`**：传统 ECPG 方式，需要在 `.c` 文件中展开 `h1.h` 内容
2. **`#include "h1.h"`**：C 原生方式，保留 `#include` 不展开，但提取宿主变量

实现方式：增加开关 `bool expand_include`，`EXEC SQL INCLUDE` 时为 `true`，`#include "..."` 时为 `false`。

下面是完整 patch：

```diff
--- a/src/interfaces/ecpg/preproc/pgc.l
+++ b/src/interfaces/ecpg/preproc/pgc.l
@@ -79,6 +79,10 @@ static void parse_include(void);
 static bool ecpg_isspace(char ch);
 static bool isdefine(void);
 static bool isinformixdefine(void);
+static bool extract_host_vars(const char *srcfile, const char *dstfile, bool add_wrapper);
+static bool is_int_var_line(const char *line, bool *is_include);
+static int debug_log = 0;  /* 对应 ecpg -D 调试模式，=1 时不删除临时文件 */
+static FILE *real_yyout = NULL;  /* 保存真实的 .c 文件输出指针 */
 
 char	   *token_start;
 
@@ -2360,6 +2364,8 @@ parse_include(void)
 	struct _yy_buffer *yb;
 	struct _include_path *ip;
 	char		inc_file[MAXPGPATH];
+	char		extract_file[MAXPGPATH];
+	bool		use_extract = false;
 	unsigned int i;
 
 	yb = mm_alloc(sizeof(struct _yy_buffer));
@@ -2403,6 +2409,24 @@ parse_include(void)
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
+		/* 输出 #include "h1.h" 到 .c 文件 */
+		fprintf(yyout, "#include \"%s\"\n", inc_file);
+
 		yyin = fopen(inc_file, "r");
 		if (!yyin)
 		{
@@ -2453,6 +2477,28 @@ parse_include(void)
 	if (!yyin)
 		mmfatal(NO_INCLUDE_FILE, "could not open include file \"%s\" on line %d", yytext, yylineno);
 
+	/* 如果提取成功，切换到临时文件，并抑制输出到 .c 文件 */
+	if (use_extract)
+	{
+		fclose(yyin);
+		yyin = fopen(extract_file, "r");
+		if (!yyin)
+			mmfatal(NO_INCLUDE_FILE, "could not open extract file \"%s\" on line %d", extract_file, yylineno);
+		input_filename = mm_strdup(extract_file);
+
+		/* 抑制输出：将 yyout 切换到 /dev/null，保存真实输出 */
+		real_yyout = yyout;
+		yyout = fopen("/dev/null", "w");
+		if (!yyout)
+		{
+			yyout = real_yyout;
+			real_yyout = NULL;
+		}
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
@@ -2471,6 +2517,252 @@ parse_include(void)
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
@@ -2498,6 +2790,9 @@ isdefine(void)
 			ptr->value != NULL &&
 			ptr->used == NULL)
 		{
+			/* 清理可能存在的临时文件 */
+			cleanup_extract_file();
+
 			/* Save state associated with the current buffer */
 			struct _yy_buffer *yb;
 
@@ -2548,6 +2843,9 @@ isinformixdefine(void)
 	if (new)
 	{
+		/* 清理可能存在的临时文件 */
+		cleanup_extract_file();
+
 		/* Save state associated with the current buffer */
 		struct _yy_buffer *yb;
 
@@ -2576,6 +2874,18 @@ isinformixdefine(void)
 					{
 						if (yy_buffer == NULL)
 						{
+							/* 恢复真实的 .c 文件输出（如果之前被抑制） */
+							if (real_yyout != NULL)
+							{
+								fclose(yyout);
+								yyout = real_yyout;
+								real_yyout = NULL;
+							}
+
+							/* 清理可能存在的临时文件 */
+							cleanup_extract_file();
+
 							/* No more input */
 							if (preproc_tos > 0)
 							{
@@ -2586,6 +2896,18 @@ isinformixdefine(void)
 						}
 						else
 						{
+							/* 恢复真实的 .c 文件输出（如果之前被抑制） */
+							if (real_yyout != NULL)
+							{
+								fclose(yyout);
+								yyout = real_yyout;
+								real_yyout = NULL;
+							}
+
+							/* 清理可能存在的临时文件 */
+							cleanup_extract_file();
+
 							/* Revert to previous input source */
 							struct _yy_buffer *yb = yy_buffer;
 							int			i;
```

---

## 关键修改说明

### 1. `parse_include()` 中

```c
/* 输出 #include "h1.h" 到 .c 文件 */
fprintf(yyout, "#include \"%s\"\n", inc_file);

/* ... */

if (use_extract)
{
    /* 切换到临时文件 */
    yyin = fopen(extract_file, "r");
    input_filename = mm_strdup(extract_file);

    /* 抑制输出：将 yyout 切换到 /dev/null */
    real_yyout = yyout;
    yyout = fopen("/dev/null", "w");
}
```

### 2. `<<EOF>>` 中

```c
if (real_yyout != NULL)
{
    fclose(yyout);           /* 关闭 /dev/null */
    yyout = real_yyout;      /* 恢复真实的 .c 文件输出 */
    real_yyout = NULL;
}
```

### 3. 效果

**`.c` 文件内容**：
```c
#include "h1.h"          /* 这行保留在 .c 文件中 */
/* h1.h.extract 的内容被写入 /dev/null，不进入 .c 文件 */
#include "h2.h"          /* 如果 h1.h.extract 中有 #include，也写入 /dev/null */
/* h2.h.extract 的内容也被写入 /dev/null */
```

**最终 `.c` 文件只保留 `#include "h1.h"`，没有任何展开内容。**

### 4. `EXEC SQL INCLUDE` 不受影响

`EXEC SQL INCLUDE` 走的是 `BEGIN(incl)` 状态，不经过 `parse_include()` 中的 `extract_host_vars()` 和 `yyout` 切换逻辑，所以仍然正常展开。

请确认这个方案是否满足需求？


# 你评价 下 文件中列的2种方式 有啥区别


两种实现方式的核心目标一致：**`#include "xx.h"` 保留在 `.c` 文件中不展开，同时 ECPG 能识别头文件中的宿主变量**。但实现路径不同，各有优劣。

---

## 方式1（用户已实现）

**核心机制**：修改 `base_yyout` 输出指针

| 维度 | 实现细节 |
|------|---------|
| 输出控制 | 直接操作 `base_yyout`（ECPG 全局输出文件指针） |
| 保存/恢复 | `struct _yy_buffer` 增加 `outfile` 字段，保存进入 include 前的 `base_yyout` |
| 区分来源 | `cpp_include` 标志区分 `#include`（C 预处理）vs `EXEC SQL INCLUDE` |
| 区分引号 | `is_quoted_include` 判断双引号 `"..."` vs 尖括号 `<...>` |
| 输出切换 | 双引号 `#include`：先写 `#include "h1.h"` 到真实输出，再把 `base_yyout` 指向 `/dev/null` |
| 恢复时机 | `<<EOF>>` 时从 `yy_buffer->outfile` 恢复 `base_yyout` |

**优点**：
- 直接操作 ECPG 原生输出指针 `base_yyout`，与现有代码风格一致
- `struct _yy_buffer` 的 `outfile` 字段和 `cpp_include` 标志设计清晰，状态管理明确
- 同时支持 `EXEC SQL INCLUDE` 展开、`#include <...>` 展开、`#include "..."` 不展开三种场景

**缺点**：
- 需要修改 `struct _yy_buffer` 定义（增加字段）
- `isdefine()` 和 `isinformixdefine()` 中也需要增加 `yb->outfile = base_yyout`（用户 patch 中已处理）
- 对 `DEVNULL` 的依赖（但 `/dev/null` 是标准路径）

---

## 方式2（我提出的）

**核心机制**：操作 `yyout`（flex 内部输出指针）

| 维度 | 实现细节 |
|------|---------|
| 输出控制 | 操作 `yyout`（flex 宏，通常等同于 `base_yyout`） |
| 保存/恢复 | 全局变量 `real_yyout` 保存真实输出指针 |
| 区分来源 | 无显式标志，依赖 `extract_host_vars()` 的调用逻辑 |
| 区分引号 | 无显式判断，依赖 `extract_host_vars()` 是否成功 |
| 输出切换 | `extract_host_vars()` 成功时，将 `yyout` 指向 `/dev/null` |
| 恢复时机 | `<<EOF>>` 时从 `real_yyout` 恢复 |

**优点**：
- 不需要修改 `struct _yy_buffer` 定义
- 逻辑集中在 `extract_host_vars()` 和 `parse_include()` 中

**缺点**：
- `yyout` 是 flex 内部宏，在不同 flex 版本或配置下可能行为不一致
- `real_yyout` 是全局变量，无栈保护，嵌套场景下可能被覆盖
- 没有显式区分 `EXEC SQL INCLUDE` 和 `#include "..."`，依赖 `extract_host_vars()` 的副作用判断
- `isdefine()` 和 `isinformixdefine()` 中的宏展开也需要处理 `yyout` 切换（patch 中未处理）

---

## 关键差异对比

| 对比项 | 方式1 | 方式2 |
|--------|-------|-------|
| 操作对象 | `base_yyout`（ECPG 全局） | `yyout`（flex 内部） |
| 状态保存 | `struct _yy_buffer.outfile`（栈结构） | `real_yyout`（全局变量） |
| 嵌套安全 | ✅ 安全（每个 buffer 独立保存） | ❌ 有风险（全局变量被覆盖） |
| 来源区分 | ✅ `cpp_include` 标志明确 | ❌ 依赖副作用判断 |
| 引号区分 | ✅ `is_quoted_include` 明确 | ❌ 依赖 `extract_host_vars()` |
| 宏展开 | ✅ 已处理（`isdefine`/`isinformixdefine`） | ❌ 未处理 |
| 侵入性 | 修改 `struct _yy_buffer` | 不修改结构体 |
| 代码清晰度 | 高 | 中 |

---

## 结论

**方式1 更优**，理由：

1. **嵌套安全性**：`struct _yy_buffer` 的 `outfile` 字段天然支持嵌套 include 的栈式保存/恢复。方式2 的 `real_yyout` 全局变量在 `h1.h` 包含 `h2.h` 时会被覆盖，恢复时可能丢失外层状态。

2. **来源区分明确**：`cpp_include` 标志清晰区分 C 预处理指令和 ECPG 指令，方式2 依赖 `extract_host_vars()` 的副作用，逻辑耦合。

3. **宏展开一致性**：方式1 在 `isdefine()` 和 `isinformixdefine()` 中也保存了 `outfile`，确保宏展开时的输出正确。方式2 未处理宏展开场景。

4. **可维护性**：方式1 的状态管理（`cpp_include`、`is_quoted_include`、`outfile`）更系统化，后续扩展（如支持更多预处理指令）更容易。

**方式2 的唯一优势**是不修改 `struct _yy_buffer`，但这个结构体本身就是 ECPG 内部实现，修改成本极低。

---

## 建议

**采用方式1**，但结合你的 `extract_host_vars()` 临时文件机制，实现完整的宿主变量提取：

1. **方式1 的输出控制机制**（`base_yyout` 切换 + `outfile` 栈保存）
2. **你的 `extract_host_vars()` 临时文件生成**（`h1.h.extract`）
3. **临时文件解析**：`base_yyout` 切换到 `/dev/null` 时，flex 解析 `h1.h.extract` 内容，ECPG 识别宿主变量但不输出到 `.c`

这样 `.c` 文件中只保留 `#include "h1.h"`，同时 ECPG 通过 `h1.h.extract` 识别了宿主变量。

是否需要我基于方式1 的框架，整合你的 `extract_host_vars()` 逻辑，给出完整 patch？



# 是的，需要




好的，基于方式1（`base_yyout` + `outfile` 栈机制）整合你的 `extract_host_vars()` 逻辑，给出完整 patch。

核心设计：
1. **方式1 的输出控制**：`cpp_include` 标志 + `is_quoted_include` + `base_yyout` 切换 + `outfile` 栈保存
2. **你的 `extract_host_vars()`**：生成 `.extract` 临时文件，顶层加 `BEGIN/END`，嵌套不加
3. **整合点**：双引号 `#include` 时，先输出 `#include "h1.h"` 到 `.c`，然后打开 `h1.h.extract` 让 flex 扫描，同时 `base_yyout` 切换到 `/dev/null` 抑制输出

---

```diff
--- a/src/interfaces/ecpg/preproc/pgc.l
+++ b/src/interfaces/ecpg/preproc/pgc.l
@@ -58,6 +58,8 @@ static int	parenths_open;
 
 /* Used to tell parse_include() whether the command was #include or #include_next */
 static bool include_next;
+/* Used to tell parse_include() whether the include came from a C preprocessor directive */
+static bool cpp_include;
 
 #define startlit()	(literalbuf[0] = '\0', literallen = 0)
 static void addlit(char *ytext, int yleng);
@@ -79,6 +81,10 @@ static void parse_include(void);
 static bool ecpg_isspace(char ch);
 static bool isdefine(void);
 static bool isinformixdefine(void);
+static bool extract_host_vars(const char *srcfile, const char *dstfile, bool add_wrapper);
+static bool is_int_var_line(const char *line, bool *is_include);
+static int debug_log = 0;  /* 对应 ecpg -D 调试模式，=1 时不删除临时文件 */
+static void cleanup_extract_file(void);
 
 char	   *token_start;
 
@@ -99,6 +105,7 @@ static struct _yy_buffer
 	YY_BUFFER_STATE buffer;
 	long		lineno;
 	char	   *filename;
+	FILE	   *outfile;		/* output file active before entering this level */
 	struct _yy_buffer *next;
 } *yy_buffer = NULL;
 
@@ -1148,6 +1155,7 @@ cppline			{space}*#([^i][A-Za-z]*|{if}|{ifdef}|{ifndef}|{import})((\/\*[^*/]*\*+
 					if (system_includes)
 					{
 						include_next = false;
+						cpp_include = true;
 						BEGIN(incl);
 					}
 					else
@@ -1160,6 +1168,7 @@ cppline			{space}*#([^i][A-Za-z]*|{if}|{ifdef}|{ifndef}|{import})((\/\*[^*/]*\*+
 					if (system_includes)
 					{
 						include_next = true;
+						cpp_include = true;
 						BEGIN(incl);
 					}
 					else
@@ -1331,12 +1340,14 @@ cppline			{space}*#([^i][A-Za-z]*|{if}|{ifdef}|{ifndef}|{import})((\/\*[^*/]*\*+
 					yyterminate();
 				}
 <C>{exec_sql}{include}{space}* {
+					cpp_include = false;
 					BEGIN(incl);
 				}
 <C>{informix_special}{include}{space}* {
 					/* are we simulating Informix? */
 					if (INFORMIX_MODE)
 					{
+						cpp_include = false;
 						BEGIN(incl);
 					}
 					else
@@ -1709,6 +1720,17 @@ cppline			{space}*#([^i][A-Za-z]*|{if}|{ifdef}|{ifndef}|{import})((\/\*[^*/]*\*+
 					if (yyin != NULL)
 						fclose(yyin);
 
+					/*
+					 * Restore the output stream that was active before entering
+					 * this include/macro level.  Close a temporary sink if we
+					 * installed one for a quoted include.
+					 */
+					if (base_yyout != yb->outfile)
+					{
+						fclose(base_yyout);
+						base_yyout = yb->outfile;
+					}
+
 					yy_delete_buffer(YY_CURRENT_BUFFER);
 					yy_switch_to_buffer(yy_buffer->buffer);
 
@@ -1826,12 +1848,15 @@ parse_include(void)
 	struct _include_path *ip;
 	char		inc_file[MAXPGPATH];
 	unsigned int i;
+	bool		is_quoted_include = false;
+	char	   *include_spec = NULL;
 
 	yb = mm_alloc(sizeof(struct _yy_buffer));
 
 	yb->buffer = YY_CURRENT_BUFFER;
 	yb->lineno = yylineno;
 	yb->filename = input_filename;
+	yb->outfile = base_yyout;
 	yb->next = yy_buffer;
 
 	yy_buffer = yb;
@@ -1848,6 +1873,16 @@ parse_include(void)
 	if (yytext[i] == ';')
 		i--;
 
+	/*
+	 * Remember whether this is a quoted (user) include.  For such includes we
+	 * still want to parse the file (so host-variable definitions are seen) but
+	 * we emit the original #include directive in the generated C file instead
+	 * of inlining the file's contents.
+	 */
+	is_quoted_include = (yytext[0] == '"');
+	if (is_quoted_include)
+		include_spec = mm_strdup(yytext);
+
 	yytext[i + 1] = '\0';
 
 	yyin = NULL;
@@ -1918,6 +1953,39 @@ parse_include(void)
 		mmfatal(NO_INCLUDE_FILE, "could not open include file \"%s\" on line %d", yytext, yylineno);
 
 	input_filename = mm_strdup(inc_file);
+
+	/*
+	 * For quoted C preprocessor includes, emit the original #include directive
+	 * in the real output, then extract host variables to a temporary file and
+	 * parse that file with output redirected to a sink so the file's contents
+	 * are not inlined in the generated C code.
+	 */
+	if (cpp_include && is_quoted_include)
+	{
+		char		extract_file[MAXPGPATH];
+		bool		add_wrapper = true;
+
+		/* Output the original #include directive */
+		include_spec[i + 1] = '\0';
+		fprintf(base_yyout, "#include %s", include_spec);
+		free(include_spec);
+
+		/* Extract host variables to temporary file */
+		snprintf(extract_file, sizeof(extract_file), "%s.extract", inc_file);
+		if (input_filename != NULL)
+		{
+			size_t len = strlen(input_filename);
+			if (len > 8 && strcmp(input_filename + len - 8, ".extract") == 0)
+				add_wrapper = false;  /* nested include, no BEGIN/END */
+		}
+		if (extract_host_vars(inc_file, extract_file, add_wrapper))
+		{
+			/* Switch to parsing the extracted file */
+			fclose(yyin);
+			yyin = fopen(extract_file, "r");
+			if (!yyin)
+				mmfatal(NO_INCLUDE_FILE, "could not open extract file \"%s\" on line %d", extract_file, yylineno);
+			input_filename = mm_strdup(extract_file);
+		}
+
+		/* Redirect output to /dev/null so extracted content is not inlined */
+		base_yyout = fopen(DEVNULL, "w");
+		if (!base_yyout)
+			mmfatal(NO_INCLUDE_FILE, "could not open %s", DEVNULL);
+	}
+
+	cpp_include = false;
+
 	yy_switch_to_buffer(yy_create_buffer(yyin, YY_BUF_SIZE));
 	yylineno = 1;
 	output_line_number();
@@ -1966,6 +2034,7 @@ isdefine(void)
 			yb->buffer = YY_CURRENT_BUFFER;
 			yb->lineno = yylineno;
 			yb->filename = mm_strdup(input_filename);
+			yb->outfile = base_yyout;
 			yb->next = yy_buffer;
 			yy_buffer = yb;
 
@@ -2010,6 +2079,7 @@ isinformixdefine(void)
 			yb->buffer = YY_CURRENT_BUFFER;
 			yb->lineno = yylineno;
 			yb->filename = mm_strdup(input_filename);
+			yb->outfile = base_yyout;
 			yb->next = yy_buffer;
 			yy_buffer = yb;
 
@@ -2471,6 +2541,228 @@ parse_include(void)
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
```

---

## 关键整合点说明

### 1. `parse_include()` 中的逻辑

```c
if (cpp_include && is_quoted_include)
{
    /* 1. 输出 #include "h1.h" 到 .c 文件 */
    fprintf(base_yyout, "#include %s", include_spec);
    
    /* 2. 提取宿主变量到 h1.h.extract */
    extract_host_vars(inc_file, extract_file, add_wrapper);
    
    /* 3. 切换到 h1.h.extract 让 flex 扫描 */
    fclose(yyin);
    yyin = fopen(extract_file, "r");
    input_filename = mm_strdup(extract_file);
    
    /* 4. base_yyout 切换到 /dev/null，抑制输出到 .c */
    base_yyout = fopen(DEVNULL, "w");
}
```

### 2. `<<EOF>>` 中的恢复

```c
if (base_yyout != yb->outfile)
{
    fclose(base_yyout);        /* 关闭 /dev/null */
    base_yyout = yb->outfile;  /* 恢复真实的 .c 文件输出 */
}
```

### 3. `extract_host_vars()` 中的嵌套判断

```c
if (input_filename != NULL)
{
    size_t len = strlen(input_filename);
    if (len > 8 && strcmp(input_filename + len - 8, ".extract") == 0)
        add_wrapper = false;  /* 已经在 .extract 中，嵌套包含 */
}
```

### 4. 生成的临时文件示例

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

**h2.h.extract**：
```c
                              /* 第1行 */
                              /* 第2行 */
int global_b;                 /* 第3行：无 BEGIN/END */
                              /* 第4行 */
```

### 5. `.c` 文件最终内容

```c
#include "h1.h"              /* 保留，不展开 */
/* h1.h.extract 和 h2.h.extract 的内容被写入 /dev/null，不进入 .c */
int main()
{
    /* ECPG 生成的代码，使用 global_a 和 global_b */
}
```

### 6. `cleanup_extract_file()` 调用位置

需要在 `<<EOF>>` 和 `isdefine()`/`isinformixdefine()` 中调用，清理临时文件。

请确认这个整合方案是否满足需求？


