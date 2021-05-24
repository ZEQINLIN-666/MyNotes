一条查询语句在经过MySQL查询优化器的各种基于成本和规则的优化会后生成一个所谓的`执行计划，这个执行计划展示了接下来具体执行查询的方式`，比如多表连接的顺序是什么，对于每个表采用什么访问方法来具体执行查询等等。

mysql 为我们提供了`EXPLAIN`语句来帮助我们查看某个查询语句的具体`执行计划`:

![img](https://s1.ax1x.com/2020/04/07/GgChLT.md.png)

explain 会展示一个表格信息, 表中的信息如下所示:

![img](https://s1.ax1x.com/2020/04/07/GgPlpn.png)

上面的内容如果感到困惑是正常的. 因为这就是本文要详细说明的内容.

## 建表

```sql
CREATE TABLE single_table (
    id INT NOT NULL AUTO_INCREMENT,
    key1 VARCHAR(100),
    key2 INT,
    key3 VARCHAR(100),
    key_part1 VARCHAR(100),
    key_part2 VARCHAR(100),
    key_part3 VARCHAR(100),
    common_field VARCHAR(100),
    PRIMARY KEY (id),
    KEY idx_key1 (key1),
    UNIQUE KEY idx_key2 (key2),
    KEY idx_key3 (key3),
    KEY idx_key_part(key_part1, key_part2, key_part3)
) Engine=InnoDB CHARSET=utf8;
```

假设有两个和single_table表构造一模一样的s1、s2表，而且这两个表里边儿有10000条记录，除id列外其余的列都插入随机值。

## 执行计划输出中各列详解

### table

不论我们的查询语句有多复杂，里边儿包含了多少个表，到最后也是需要对每个表进行单表访问的，所以`MySQL规定EXPLAIN语句输出的每条记录都对应着某个单表的访问方法，该条记录的table列代表着该表的表名`。

table列代表着该表的表名。所以我们看一条比较简单的查询语句：

![img](https://s1.ax1x.com/2020/04/07/GgEis1.png)

这个查询语句只涉及对s1表的单表查询，所以EXPLAIN输出中只有一条记录，其中的table列的值是s1，表明这条记录是用来说明对s1表的单表访问方法的。

![img](https://s1.ax1x.com/2020/04/07/GgEnRH.png)

可以看到这个连接查询的执行计划中有两条记录，这两条记录的table列分别是s1和s2，这两条记录用来分别说明对s1表和s2表的访问方法是什么。

### id

我们知道我们写的查询语句一般都以SELECT关键字开头，比较简单的查询语句里只有一个SELECT关键字，比如下边这个查询语句：

> SELECT * FROM s1 WHERE key1 = 'a';

稍微复杂一点的连接查询中也只有一个SELECT关键字，比如：

```sql
SELECT * FROM s1 INNER JOIN s2
    ON s1.key1 = s2.key1
    WHERE s1.common_field = 'a';
```

但是下边两种情况下在一条查询语句中会出现多个SELECT关键字：

查询中包含子查询的情况

> SELECT * FROM s1 WHERE key1 IN (SELECT key3 FROM s2);

查询中包含UNION语句的情况

> SELECT * FROM s1 UNION SELECT * FROM s2;

查询语句中每出现一个SELECT关键字，设计MySQL的大叔就会为它分配一个唯一的id值。这个id值就是EXPLAIN语句的第一个列. 对于连接查询来说，一个SELECT关键字后边的FROM子句中可以跟随多个表，所以在连接查询的执行计划中，每个表都会对应一条记录，但是这些记录的id值都是相同的

![img](https://s1.ax1x.com/2020/04/07/GgtEPH.png)

可以看到，上述连接查询中参与连接的s1和s2表分别对应一条记录，但是这两条记录对应的id值都是1。这里需要大家记住的是，在连接查询的执行计划中，每个表都会对应一条记录，这些记录的id列的值是相同的，**出现在前边的表表示驱动表，出现在后边的表表示被驱动表**。所以从上边的EXPLAIN输出中我们可以看出，查询优化器准备让s1表作为驱动表，让s2表作为被驱动表来执行查询。

> 如果查询优化器将子查询转换为了连接查询, 那么就是两个 select 对应一个 id.

### select_type

MySQL为每一个SELECT关键字代表的小查询都定义了一个称之为select_type的属性，意思是我们只要知道了某个小查询的select_type属性，就知道了这个小查询在整个大查询中扮演了一个什么角色

![img](https://s1.ax1x.com/2020/04/07/Gga6zj.png)

### type

执行计划的一条记录就代表着MySQL对某个表的执行查询时的访问方法

完整的访问方法如下：

> system，const，eq_ref，ref，fulltext，ref_or_null，index_merge，unique_subquery，index_subquery，range，index，ALL

### possible_keys和key

在EXPLAIN语句输出的执行计划中，possible_keys列表示在某个查询语句中，对某个表执行单表查询时可能用到的索引有哪些，key列表示实际用到的索引有哪些，比方说下边这个查询：

![img](https://s1.ax1x.com/2020/04/07/GgrXmd.png)

上述执行计划的possible_keys列的值是idx_key1,idx_key3，表示该查询可能使用到idx_key1,idx_key3两个索引，然后key列的值是idx_key3，表示经过查询优化器计算使用不同索引的成本后，最后决定使用idx_key3来执行查询比较划算。

另外需要注意的一点是，possible_keys列中的值并不是越多越好，可能使用的索引越多，查询优化器计算查询成本时就得花费更长时间，所以如果可以的话，尽量删除那些用不到的索引

### ref

ref列展示的就是与索引列作等值匹配的东东是个啥，比如只是一个常数或者是某个列:

![img](https://s1.ax1x.com/2020/04/07/Ggyd5q.png)

可以看到ref列的值是const，表明在使用idx_key1索引执行查询时，与key1列作等值匹配的对象是一个常数，当然有时候更复杂一点：

![img](https://s1.ax1x.com/2020/04/07/Gg6CLQ.md.png)

可以看到对被驱动表s2的访问方法是eq_ref，而对应的ref列的值是xiaohaizi.s1.id，这说明在对被驱动表进行访问时会用到PRIMARY索引，也就是聚簇索引与一个列进行等值匹配的条件，s2表的id作等值匹配的对象就是xiaohaizi.s1.id列（注意这里把数据库名也写出来了）。

有的时候与索引列进行等值匹配的对象是一个函数，比方说下边这个查询：

![img](https://s1.ax1x.com/2020/04/07/Gg6JW6.png)

### rows

如果查询优化器决定使用全表扫描的方式对某个表执行查询时，执行计划的rows列就代表预计需要扫描的行数，如果使用索引来执行查询时，执行计划的rows列就代表预计扫描的索引记录行数。比如下边这个查询：

![img](https://s1.ax1x.com/2020/04/07/Gg6fmQ.md.png)

我们看到执行计划的rows列的值是266，这意味着查询优化器在经过分析使用idx_key1进行查询的成本之后，觉得满足key1 > 'z'这个条件的记录只有266条。

### filtered

之前在分析连接查询的成本时提出过一个condition filtering的概念，就是MySQL在计算驱动表扇出时采用的一个策略：

> 如果使用的是全表扫描的方式执行的单表查询，那么计算驱动表扇出时需要估计出满足搜索条件的记录到底有多少条。
>
> 如果使用的是索引执行的单表扫描，那么计算驱动表扇出的时候需要估计出满足除使用到对应索引的搜索条件外的其他搜索条件的记录有多少条。

比方说下边这个查询：

![img](https://s1.ax1x.com/2020/04/07/GgcsHJ.png)

> 从执行计划的key列中可以看出来，该查询使用idx_key1索引来执行查询，从rows列可以看出满足key1 > 'z'的记录有266条。执行计划的filtered列就代表查询优化器预测在这266条记录中，有多少条记录满足其余的搜索条件，也就是common_field = 'a'这个条件的百分比。此处filtered列的值是10.00，说明查询优化器预测在266条记录中有10.00%的记录满足common_field = 'a'这个条件。

对于单表查询来说，这个filtered列的值没什么意义，我们更关注在连接查询中驱动表对应的执行计划记录的filtered值，比方说下边这个查询：

![img](https://s1.ax1x.com/2020/04/07/Ggg326.png)

> 从执行计划中可以看出来，查询优化器打算把s1当作驱动表，s2当作被驱动表。我们可以看到驱动表s1表的执行计划的rows列为9688， filtered列为10.00，这意味着驱动表s1的扇出值就是9688 × 10.00% = 968.8，这说明还要对被驱动表执行大约968次查询。

### Extra

Extra列是用来说明一些额外信息的，我们可以通过这些额外信息来更准确的理解MySQL到底将如何执行给定的查询语句

```
No tables used
```

> 当查询语句的没有FROM子句时将会提示该额外信息
> mysql> EXPLAIN SELECT 1;

```
Impossible WHERE
```

> 查询语句的WHERE子句永远为FALSE时将会提示该额外信息

```
No matching min/max row
```

> 当查询列表处有MIN或者MAX聚集函数，但是并没有符合WHERE子句中的搜索条件的记录时，将会提示该额外信息

```
Using index
```

> 当我们的查询列表以及搜索条件中只包含属于某个索引的列，也就是在可以使**用索引覆盖**的情况下，在Extra列将会提示该额外信息

```
Using index condition
```

> **有些搜索条件中虽然出现了索引列，但却不能使用到索引**

```
Using where
```

> 当我们使用全表扫描来执行对某个表的查询，并且该语句的WHERE子句中有针对该表的搜索条件时，在Extra列中会提示上述额外信息

```
Using join buffer (Block Nested Loop)
```

> 在连接查询执行过程中，当被驱动表不能有效的利用索引加快访问速度，MySQL一般会为其分配一块名叫join buffer的内存块来加快查询速度，也就是我们所讲的基于块的嵌套循环算法

```
Not exists
```

> 当我们使用左（外）连接时，如果WHERE子句中包含要求被驱动表的某个列等于NULL值的搜索条件，而且那个列又是不允许存储NULL值的，那么在该表的执行计划的Extra列就会提示Not exists额外信息:
> mysql> EXPLAIN SELECT * FROM s1 LEFT JOIN s2 ON s1.key1 = s2.key1 WHERE s2.id IS NULL;
> 上述查询中s1表是驱动表，s2表是被驱动表，s2.id列是不允许存储NULL值的，而WHERE子句中又包含s2.id IS NULL的搜索条件，这意味着必定是驱动表的记录在被驱动表中找不到匹配ON子句条件的记录才会把该驱动表的记录加入到最终的结果集，所以对于某条驱动表中的记录来说，如果能在被驱动表中找到1条符合ON子句条件的记录，那么该驱动表的记录就不会被加入到最终的结果集，也就是说我们没有必要到被驱动表中找到全部符合ON子句条件的记录，这样可以稍微节省一点性能。

```
Using intersect(...)、Using union(...)和Using sort_union(...)
```

> 如果执行计划的Extra列出现了Using intersect(...)提示，说明准备使用Intersect索引合并的方式执行查询，括号中的...表示需要进行索引合并的索引名称；如果出现了Using union(...)提示，说明准备使用Union索引合并的方式执行查询；出现了Using sort_union(...)提示，说明准备使用Sort-Union索引合并的方式执行查询。

```sql
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' AND key3 = 'a';
```

> 其中Extra列就显示了Using intersect(idx_key3,idx_key1)，表明MySQL即将使用idx_key3和idx_key1这两个索引进行Intersect索引合并的方式执行查询。

```
Zero limit
```

> 当我们的LIMIT子句的参数为0时，表示压根儿不打算从表中读出任何记录，将会提示该额外信息

```
Using filesort
```

> 有一些情况下对结果集中的记录进行排序是可以使用到索引的

```sql
mysql> EXPLAIN SELECT * FROM s1 ORDER BY key1 LIMIT 10;
```

> 这个查询语句可以利用 idx_key1 索引直接取出 key1 列的 10 条记录，然后再进行回表操作就好了。但是很多情况下排序操作无法使用到索引，只能在内存中（记录较少的时候）或者磁盘中（记录较多的时候）进行排序，MySQL 把**这种在内存中或者磁盘上进行排序的方式统称为文件排序**（英文名：filesort）。如果某个查询需要使用文件排序的方式执行查询，就会在执行计划的 Extra 列中显示 Using filesort 提示

```
Using temporary
```

> 在许多查询的执行过程中，MySQL可能会借助临时表来完成一些功能，比如去重、排序之类的，比如我们在执行许多包含DISTINCT、GROUP BY、UNION等子句的查询过程中，如果不能有效利用索引来完成查询，MySQL很有可能寻求通过建立内部的临时表来执行查询。如果查询中使用到了内部的临时表，在执行计划的Extra列将会显示Using temporary提示

...省略一部分

## Json格式的执行计划

上边介绍的 EXPLAIN 语句输出中缺少了一个衡量执行计划好坏的重要属性 —— 成本。不过 MySQL 提供了一种查看某个执行计划花费的成本的方式：

> 在EXPLAIN单词和真正的查询语句中间加上FORMAT=JSON。

```sql
mysql> EXPLAIN FORMAT=JSON SELECT * FROM s1 INNER JOIN s2 ON s1.key1 = s2.key2 WHERE s1.common_field = 'a'\G
*************************** 1. row ***************************

EXPLAIN: {
  "query_block": {
    "select_id": 1,     # 整个查询语句只有1个SELECT关键字，该关键字对应的id号为1
    "cost_info": {
      "query_cost": "3197.16"   # 整个查询的执行成本预计为3197.16
    },
    "nested_loop": [    # 几个表之间采用嵌套循环连接算法执行
    
    # 以下是参与嵌套循环连接算法的各个表的信息
      {
        "table": {
          "table_name": "s1",   # s1表是驱动表
          "access_type": "ALL",     # 访问方法为ALL，意味着使用全表扫描访问
          "possible_keys": [    # 可能使用的索引
            "idx_key1"
          ],
          "rows_examined_per_scan": 9688,   # 查询一次s1表大致需要扫描9688条记录
          "rows_produced_per_join": 968,    # 驱动表s1的扇出是968
          "filtered": "10.00",  # condition filtering代表的百分比
          "cost_info": {
            "read_cost": "1840.84",     # 稍后解释
            "eval_cost": "193.76",      # 稍后解释
            "prefix_cost": "2034.60",   # 单次查询s1表总共的成本
            "data_read_per_join": "1M"  # 读取的数据量
          },
          "used_columns": [     # 执行查询中涉及到的列
            "id",
            "key1",
            "key2",
            "key3",
            "key_part1",
            "key_part2",
            "key_part3",
            "common_field"
          ],
          
          # 对s1表访问时针对单表查询的条件
          "attached_condition": "((`xiaohaizi`.`s1`.`common_field` = 'a') and (`xiaohaizi`.`s1`.`key1` is not null))"
        }
      },
      {
        "table": {
          "table_name": "s2",   # s2表是被驱动表
          "access_type": "ref",     # 访问方法为ref，意味着使用索引等值匹配的方式访问
          "possible_keys": [    # 可能使用的索引
            "idx_key2"
          ],
          "key": "idx_key2",    # 实际使用的索引
          "used_key_parts": [   # 使用到的索引列
            "key2"
          ],
          "key_length": "5",    # key_len
          "ref": [      # 与key2列进行等值匹配的对象
            "xiaohaizi.s1.key1"
          ],
          "rows_examined_per_scan": 1,  # 查询一次s2表大致需要扫描1条记录
          "rows_produced_per_join": 968,    # 被驱动表s2的扇出是968（由于后边没有多余的表进行连接，所以这个值也没啥用）
          "filtered": "100.00",     # condition filtering代表的百分比
          
          # s2表使用索引进行查询的搜索条件
          "index_condition": "(`xiaohaizi`.`s1`.`key1` = `xiaohaizi`.`s2`.`key2`)",
          "cost_info": {
            "read_cost": "968.80",      # 稍后解释
            "eval_cost": "193.76",      # 稍后解释
            "prefix_cost": "3197.16",   # 单次查询s1、多次查询s2表总共的成本
            "data_read_per_join": "1M"  # 读取的数据量
          },
          "used_columns": [     # 执行查询中涉及到的列
            "id",
            "key1",
            "key2",
            "key3",
            "key_part1",
            "key_part2",
            "key_part3",
            "common_field"
          ]
        }
      }
    ]
  }
}
1 row in set, 2 warnings (0.00 sec)
```

先看s1表的"cost_info"部分：

```json
"cost_info": {
    "read_cost": "1840.84",
    "eval_cost": "193.76",
    "prefix_cost": "2034.60",
    "data_read_per_join": "1M"
}
```

`read_cost`是由下边这两部分组成的：

> IO成本
> 检测rows × (1 - filter)条记录的CPU成本

`eval_cost`是这样计算的：

> 检测 rows × filter条记录的成本。

`prefix_cost`就是单独查询s1表的成本，也就是：

> read_cost + eval_cost

`data_read_per_join`表示在此次查询中需要读取的数据量。