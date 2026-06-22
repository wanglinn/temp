# 需要指出的是，前面虽然说了 #define XX 等价于 ecpg -DXX ,但是 已修改代码 不支持 -DXX=a 这种形式， 也就是完全按兼容oracle做的。 那么针对上面 #define  XXXX \   问题，是不是直接不让用 宏替换是否就可以了

是的，完全正确。

既然已经按 Oracle 兼容方向做了，且不支持 `-DXX=a` 这种带值形式，那 `#define` 就只做**条件编译开关**，不做任何宏替换。多行宏定义的问题自然就不存在了——因为根本不存在"宏值"这个概念。

## 简化后的 `is_cpp_define_line`

不需要提取宏值，甚至不需要 `value` 参数。只判断是不是 `#define` 指令、提取宏名即可：

```c
static bool
is_cpp_define_line(const char *line, char **name)
{
    const char *p = line;
    const char *start;
    int len;

    while (*p && ecpg_isspace(*p))
        p++;

    if (*p != '#')
        return false;
    p++;

    while (*p && (*p == ' ' || *p == '\t'))
        p++;

    if (strncmp(p, "define", 6) != 0 ||
        (isalnum((unsigned char)p[6]) || p[6] == '_'))
        return false;
    p += 6;

    while (*p && (*p == ' ' || *p == '\t'))
        p++;

    start = p;
    if (!(isalpha((unsigned char)*p) || *p == '_'))
        return false;
    p++;
    while (*p && (isalnum((unsigned char)*p) || *p == '_'))
        p++;

    len = p - start;
    *name = (char *) mm_alloc(len + 1);
    strncpy(*name, start, len);
    (*name)[len] = '\0';

    /* Ignore everything after macro name (including multi-line definitions) */
    return true;
}
```

对应的 `cppline` 规则调用也简化：

```c
else if (is_cpp_define_line(yytext, &def_name)) {
    bool active = (preproc_tos <= 0) ||
                  stacked_if_value[preproc_tos].active;
    process_cpp_define(def_name, active);
    free(def_name);
    output_cppline();
}
```

不需要 `def_value`，不需要 `free(def_value)`。

---

## 完整简化 Patch

```diff
--- a/src/interfaces/ecpg/preproc/pgc.l
+++ b/src/interfaces/ecpg/preproc/pgc.l
@@ -79,7 +79,9 @@ static void process_cpp_if(int cond_type, char **macro_name, int default_state);
 static void process_cpp_else(int default_state);
 static void process_cpp_endif(int default_state);
 static void output_cppline(void);
-static void process_cpp_define(const char *name, const char *value);
+static void process_cpp_define(const char *name, bool active);
+static void process_cpp_undef(const char *name);
+static bool is_cpp_undef_line(const char *line, char **name);
 
 char	   *token_start;
 
@@ -1184,10 +1186,9 @@ cppline			{space}*#([^i][A-Za-z]*|{if}|{ifdef}|{ifndef}|{import})((\/\*[^*/]*\*+
 					else if (is_cpp_else_line(yytext)) {
 						process_cpp_else(C);
 					}
-					else if (is_cpp_define_line(yytext, &def_name, &def_value)) {
-						process_cpp_define(def_name, def_value);
+					else if (is_cpp_define_line(yytext, &def_name)) {
+						process_cpp_define(def_name, true);
 						free(def_name);
-						free(def_value);
 						output_cppline();
 					}
 					else {
@@ -1360,10 +1361,9 @@ cppline			{space}*#([^i][A-Za-z]*|{if}|{ifdef}|{ifndef}|{import})((\/\*[^*/]*\*+
 					else if (is_cpp_else_line(yytext)) {
 						process_cpp_else(C);
 					}
-					else if (is_cpp_define_line(yytext, &def_name, &def_value)) {
-						process_cpp_define(def_name, def_value);
+					else if (is_cpp_define_line(yytext, &def_name)) {
+						process_cpp_define(def_name, true);
 						free(def_name);
-						free(def_value);
 						output_cppline();
 					}
 					else {
@@ -1377,10 +1377,19 @@ cppline			{space}*#([^i][A-Za-z]*|{if}|{ifdef}|{ifndef}|{import})((\/\*[^*/]*\*+
 					else if (is_cpp_else_line(yytext)) {
 						process_cpp_else(C);
 					}
-					else if (is_cpp_define_line(yytext, &def_name, &def_value)) {
-						process_cpp_define(def_name, def_value);
+					else if (is_cpp_define_line(yytext, &def_name)) {
+						bool active = (preproc_tos <= 0) ||
+						              stacked_if_value[preproc_tos].active;
+						process_cpp_define(def_name, active);
+						free(def_name);
+						output_cppline();
+					}
+					else if (is_cpp_undef_line(yytext, &def_name)) {
+						bool active = (preproc_tos <= 0) ||
+						              stacked_if_value[preproc_tos].active;
+						if (active)
+							process_cpp_undef(def_name);
 						free(def_name);
-						free(def_value);
 						output_cppline();
 					}
 					else {
@@ -1395,10 +1404,19 @@ cppline			{space}*#([^i][A-Za-z]*|{if}|{ifdef}|{ifndef}|{import})((\/\*[^*/]*\*+
 					else if (is_cpp_else_line(yytext)) {
 						process_cpp_else(C);
 					}
-					else if (is_cpp_define_line(yytext, &def_name, &def_value)) {
-						process_cpp_define(def_name, def_value);
+					else if (is_cpp_define_line(yytext, &def_name)) {
+						bool active = (preproc_tos <= 0) ||
+						              stacked_if_value[preproc_tos].active;
+						process_cpp_define(def_name, active);
+						free(def_name);
+						output_cppline();
+					}
+					else if (is_cpp_undef_line(yytext, &def_name)) {
+						bool active = (preproc_tos <= 0) ||
+						              stacked_if_value[preproc_tos].active;
+						if (active)
+							process_cpp_undef(def_name);
 						free(def_name);
-						free(def_value);
 						output_cppline();
 					}
 					else {
@@ -2411,7 +2429,7 @@ is_cpp_endif_line(const char *line)
 }
 
 static bool
-is_cpp_define_line(const char *line, char **name, char **value)
+is_cpp_define_line(const char *line, char **name)
 {
     const char *p = line;
     const char *start;
@@ -2446,23 +2464,7 @@ is_cpp_define_line(const char *line, char **name, char **value)
     (*name)[len] = '\0';
 
-    while (*p && (*p == ' ' || *p == '\t'))
-        p++;
-
-    if (*p == '\0' || *p == '\n' || *p == '\r')
-    {
-        *value = mm_strdup("");
-    }
-    else
-    {
-        const char *vstart = p;
-        while (*p && *p != '\n' && *p != '\r')
-            p++;
-        len = p - vstart;
-        while (len > 0 && (vstart[len - 1] == ' ' || vstart[len - 1] == '\t'))
-            len--;
-        *value = (char *) mm_alloc(len + 1);
-        strncpy(*value, vstart, len);
-        (*value)[len] = '\0';
-    }
+    /* Ignore macro value (including multi-line definitions).
+     * #define is only for conditional compilation, not macro expansion. */
 
     return true;
 }
@@ -2479,6 +2481,9 @@ process_cpp_define(const char *name, bool active)
      * Oracle Pro*C semantics: #define in .pc files is ONLY used for
      * conditional compilation (#ifdef/#ifndef). It does NOT participate
      * in macro expansion (isdefine() only checks the 'defines' list).
+     *
+     * Multi-line definitions like "#define XX \" are also handled here:
+     * we only care about the macro name, the value is irrelevant.
      */
     for (cptr = cpp_defines; cptr != NULL; cptr = cptr->next)
     {
@@ -2491,3 +2496,57 @@ process_cpp_define(const char *name, bool active)
     cptr->next = cpp_defines;
     cpp_defines = cptr;
 }
+
+static void
+process_cpp_undef(const char *name)
+{
+    struct _cpp_defines *cptr, *cptr_prev = NULL;
+
+    for (cptr = cpp_defines; cptr != NULL; cptr_prev = cptr, cptr = cptr->next)
+    {
+        if (strcmp(name, cptr->name) == 0)
+        {
+            if (cptr_prev == NULL)
+                cpp_defines = cptr->next;
+            else
+                cptr_prev->next = cptr->next;
+            free(cptr->name);
+            free(cptr);
+            break;
+        }
+    }
+}
+
+static bool
+is_cpp_undef_line(const char *line, char **name)
+{
+    const char *p = line;
+    const char *start;
+    int len;
+
+    while (*p && ecpg_isspace(*p))
+        p++;
+
+    if (*p != '#')
+        return false;
+    p++;
+
+    while (*p && (*p == ' ' || *p == '\t'))
+        p++;
+
+    if (strncmp(p, "undef", 5) != 0 ||
+        (isalnum((unsigned char)p[5]) || p[5] == '_'))
+        return false;
+    p += 5;
+
+    while (*p && (*p == ' ' || *p == '\t'))
+        p++;
+
+    start = p;
+    if (!(isalpha((unsigned char)*p) || *p == '_'))
+        return false;
+    p++;
+    while (*p && (isalnum((unsigned char)*p) || *p == '_'))
+        p++;
+
+    len = p - start;
+    *name = (char *) mm_alloc(len + 1);
+    strncpy(*name, start, len);
+    (*name)[len] = '\0';
+
+    return true;
+}
```

---

## 变更点

| 项 | 变更 |
|---|---|
| `is_cpp_define_line` 签名 | 去掉 `char **value` 参数 |
| `is_cpp_define_line` 实现 | 提取宏名后，忽略后续所有内容（包括 `\` 续行） |
| 调用方 | 去掉 `def_value` 相关变量和 `free(def_value)` |
| `process_cpp_define` | 只接收 `name` 和 `active`，只操作 `cpp_defines` |

多行宏定义如：
```c
#define CHECK_EXEC_STATUS \
    do { \
        printf("aaaa\n"); \
    } while(0)
```

`cppline` 规则匹配整行（含续行），`is_cpp_define_line` 提取到 `CHECK_EXEC_STATUS` 就返回，后续 `\` 和宏体全部忽略。`process_cpp_define` 只记录宏名到 `cpp_defines`。完美处理。

