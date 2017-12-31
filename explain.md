# explain 语句每个列的简单解释
## select_type
表示 SELECT 的类型，常见的取值有 SIMPLE（简单表，即不使用表连接或者子查询）、PRIMARY（主查询，即外层的查询）、UNION（UNION 中的第二个或者后面的查询语句）、SUBQUERY（子查询中的第一个 SELECT）等。

## table
输出结果集的表。

##  type
表示表的连接类型，性能由最差到最好的连接类型为
- type=ALL
全表扫描，mysql会遍历全表来找到匹配的行
```sql
mysql> explain select trade_number from tbl_order_goods where pay_way=1\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_order_goods
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 86739
        Extra: Using where
1 row in set (0.00 sec)
```
- type=index
索引全扫描，mysql会便利整个索引来查询
```sql
mysql> explain select trade_number from tbl_order_goods\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_order_goods
         type: index
possible_keys: NULL
          key: trade_number
      key_len: 122
          ref: NULL
         rows: 86739
        Extra: Using index
1 row in set (0.00 sec)
```

- type=range
索引范围扫描，<,<=,>,>=,between等
```sql
mysql> explain select * from tbl_order_goods where id>20 and id<300\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_order_goods
         type: range
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: NULL
         rows: 277
        Extra: Using where
1 row in set (0.00 sec)
```
- type=ref
使用非唯一索引扫描或者唯一索引的前缀扫描，返回匹配某个单独值得记录行，
```sql
trade_number 为非唯一索引
mysql> explain select * from tbl_order_goods where trade_number = '1712211553136604'\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_order_goods
         type: ref
possible_keys: trade_number
          key: trade_number
      key_len: 122
          ref: const
         rows: 1
        Extra: Using index condition
1 row in set (0.01 sec)

mysql> explain select a.*,b.* from tbl_order_goods a,tbl_consume_trade b where a.trade_number = b.trade_number\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: b
         type: ALL
possible_keys: trade_number_key
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 51282
        Extra: Using where
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: a
         type: ref
possible_keys: trade_number
          key: trade_number
      key_len: 122
          ref: shizhan_trade.b.trade_number
         rows: 1
        Extra: Using index condition
2 rows in set (0.00 sec)

ERROR: 
No query specified
```
- type=eq_ref
类似于ref，区别就在使用的索引是唯一索引，对于每个索引键值，表中只有一条记录匹配，简单的说就是多表链接中国使用了primary key 或者unique index  作为关联条件
```sql
mysql> EXPLAIN SELECT * FROM tbl_user a,tbl_user_login b where a.uid = b.uid\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: b
         type: ALL
possible_keys: PRIMARY
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 5535
        Extra: NULL
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: a
         type: eq_ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: ucloud.b.uid
         rows: 1
        Extra: NULL
2 rows in set (0.01 sec)
```
- type=system/const
表中最多有一个匹配行，查询起来非常迅速，所以这个匹配行中的其他列的值可以被优化器在当前查询中当做常量来处理，例如，根据主键primary key 或者唯一索引 unique index 进行的查询
```sql
mysql> EXPLAIN SELECT * from (SELECT * from tbl_user where uid = 203476)a\G;
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: <derived2>
         type: system
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 1
        Extra: NULL
*************************** 2. row ***************************
           id: 2
  select_type: DERIVED
        table: tbl_user
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
        Extra: NULL
2 rows in set (0.00 sec)
```
- type=NULL
mysql不用访问表或者索引，直接就能得到结果
```sql
mysql> EXPLAIN SELECT 1 from dual where 1\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: NULL
         type: NULL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
        Extra: No tables used
1 row in set (0.00 sec)

```
- type另外还有
```fi
ref_or_null（与 ref 类似，区别在于条件中包含对 NULL 的查询）、
index_merge(索引合并优化)、
unique_subquery（in的后面是一个查询主键字段的子查询）、
index_subquery（与 unique_subquery 类似，区别在于 in 的后面是查询非唯一索引字段的子查询）、
```


## possible_keys
表示查询时，可能使用的索引。
## key
表示实际使用的索引。
## key_len
索引字段的长度。
## rows
扫描行的数量。
## Extra
执行情况的说明和描述。
Extra是EXPLAIN输出中另外一个很重要的列，该列显示MySQL在查询过程中的一些详细信息，包含的信息很多，只选择几个重点的介绍下。

- Using filesort

MySQL有两种方式可以生成有序的结果，通过排序操作或者使用索引，当Extra中出现了Using filesort 说明MySQL使用了后者，但注意虽然叫filesort但并不是说明就是用了文件来进行排序，只要可能排序都是在内存里完成的。大部分情况下利用索引排序更快，所以一般这时也要考虑优化查询了。

- Using temporary

说明使用了临时表，一般看到它说明查询需要优化了，就算避免不了临时表的使用也要尽量避免硬盘临时表的使用。

 

- Not exists

MYSQL优化了LEFT JOIN，一旦它找到了匹配LEFT JOIN标准的行， 就不再搜索了。

- Using index

说明查询是覆盖了索引的，这是好事情。MySQL直接从索引中过滤不需要的记录并返回命中的结果。这是MySQL服务层完成的，但无需再回表查询记录。

- Using index condition

这是MySQL 5.6出来的新特性，叫做“索引条件推送”。简单说一点就是MySQL原来在索引上是不能执行如like这样的操作的，但是现在可以了，这样减少了不必要的IO操作，但是只能用在二级索引上，详情点这里。

- Using where

使用了WHERE从句来限制哪些行将与下一张表匹配或者是返回给用户。




>explain extended 输出更清晰的SQL
>
>explain partitions 输出SQL访问的分区

