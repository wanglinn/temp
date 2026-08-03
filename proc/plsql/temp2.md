#
现在 pl/sql中的begin .. end 之间的内容语句提取出来了。现在想映射到.c 文件中， 类似 exec sql select 1 into :v1 from dual; 语句 映射为

ECPGdo (__LINE__, 0, 1, NULL, 0, ECPGst_normal, "select 1 from dual", ECPGt_E0IT,

ECPGt_int, &(v1), (long)1, (long)1, sizeof(int),

ECPGt_NO_INDICATOR, NULL, 0L, 0L, 0L, ECPGt_EORT); 

：： 我想的方法是：获取 语句块中变量信息，然后安装上面ECPGdo  那样，把变量列举出来，参数类型为出入参 混合类型

#
1. 目标数据库 : oracle 

2. A ，B, C 都有

3. 需要支持指示器变量

4. 支持 At :conn -- 这个可以先不用管

5. 新增类型 意义大吗？若不大，先用老的 ECPGst_normal


#
有关绑定变量， 比如 select 1 into :v1 from t1 where id1= :v2 and id2=:v1;

我发现 ECPGdo 中语句为 select 1 from t1 where id1 = :1 and id2 = :2这种，且 后面跟的

是3个变量，2个输入1个输出

#
你上面说的有问题 ， 若是入参参数，编号是一直增加的，不会判断重复

exec sql select 1 into :v1 from t1 where id2 = :v2 and id3 = :v2 and id4 = :v1;

对应 ECPGdo 为 

"select 1 from t1 where id2 = :1 and id3 = :2 and id4 = :3",

ECPGt_int, &(v2), ..

ECPGt_NO_INDICATOR, ..

ECPGt_int, &(v2), ..

ECPGt_NO_INDICATOR, ..

ECPGt_int, &(v1), ..
ECPGt_NO_INDICATOR, .., ECPGt_EOIT,

ECPGt_int, &(v1), ..
ECPGt_NO_INDICATOR, ..,  ECPGt_EORT

#
1. OCI 使用双向绑定方式，不区分输入输出。

2. 对于指示器 :var:ind，SQL 字符串中 :var 替换为 :N 后，:ind 是完全删除（指示器信息只放在参数列表中）

#
output.c.txt 就是 ecpg中的 output.c 文件，你看下函数 output_statement 语句， 打印ECPGdo 中的输入参数使用 dump_variables(argsinsert, 1); 打印输出参数使用 dump_variables(argsresult, 1);  感觉上面可以不用使用函数 output_hv_binding  这类函数


#
1. 新增函数 

2. 先不管是否还有其它字段需要初始化

3. 是的， 你看下 这个结构体变量 no_indicator ， 在 add_variable_to_head 函数中是最后1个参数

4. 是的需要 whenever_action 处理
