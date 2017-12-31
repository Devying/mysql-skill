# show profile 分析SQL
默认情况下profiling是关闭的，可以通过set在Session基本开启profiling
```sql
mysql> select @@profiling;
+-------------+
| @@profiling |
+-------------+
|           0 |
+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> set profiling=1;
Query OK, 0 rows affected, 1 warning (0.01 sec)
```
通过profile,我们能够更淸楚地了解SQL执行的过程。例如，我们知道MylSAM表有表元数据的缓存（例如行数，即COUNT(*)值）,那么对一个MylSAM及的COUNT(*)是不需耍消耗太多资源的.而对于Innodb来说，就没有这种元数据缓存，COUNT(*)执行得较慢。
```sql

mysql> select count(*) from ucloud.tbl_user;
+----------+
| count(*) |
+----------+
|  3291437 |
+----------+
1 row in set (1.22 sec)

mysql> show profiles;
+----------+------------+----------------------------------------------------+
| Query_ID | Duration   | Query                                              |
+----------+------------+----------------------------------------------------+
|        1 | 0.07366675 | select count(*) from shizhan_trade.tbl_order_goods |
|        2 | 0.99248925 | select count(*) from ucloud.tbl_user               |
|        3 | 1.07439575 | select count(*) from ucloud.tbl_user               |
|        4 | 0.00059200 | show engines                                       |
|        5 | 0.00014000 | show engines --help                                |
|        6 | 0.00016500 | show engines -h                                    |
|        7 | 0.00014375 | show engines from tbl_order_goods                  |
|        8 | 0.00291500 | desc ucloud.tbl_user                               |
|        9 | 0.00046675 | show create table ucloud.tbl_user                  |
|       10 | 1.21806650 | select count(*) from ucloud.tbl_user               |
+----------+------------+----------------------------------------------------+
10 rows in set, 1 warning (0.00 sec)

mysql> show profile for query 10;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000056 |
| checking permissions | 0.000007 |
| Opening tables       | 0.000015 |
| init                 | 0.000010 |
| System lock          | 0.000008 |
| optimizing           | 0.000005 |
| statistics           | 0.000010 |
| preparing            | 0.000009 |
| executing            | 0.000004 |
| Sending data         | 1.217784 |
| end                  | 0.000022 |
| query end            | 0.000022 |
| closing tables       | 0.000020 |
| freeing items        | 0.000046 |
| cleaning up          | 0.000050 |
+----------------------+----------+
15 rows in set, 1 warning (0.00 sec)
```
由此可见count(*) 这个过程主要时间消耗在<font color="#f00">**Sending data**</font> 这个过程
>Sending data 状态表示Mysql 线程开始访问数据行并把结果返回给客户端，而不仅仅是返回结果给客户端，由于在Sending data状态下，Mysql线程往往需要做大量的磁盘读取操作，所以经常是真个查询中耗时最长的状态
为了更加清晰的看到排序结果，可以查询INFORMATION_SCHEMA>PROFILING 表并按照时间做一个DESC排序
```sql
mysql> select STATE,SUM(DURATION) AS Total_R, ROUND(100*SUM(DURATION)/(SELECT SUM(DURATION) FROM INFORMATION_SCHEMA.PROFILING WHERE QUERY_ID = @query_id),2) AS Calls,SUM(DURATION)/COUNT(*) AS "R/Call" FROM INFORMATION_SCHEMA.PROFILING WHERE QUERY_ID = @query_id GROUP BY STATE ORDER BY 
Total_R DESC;
+----------------------+----------+-------+--------------+
| STATE                | Total_R  | Calls | R/Call       |
+----------------------+----------+-------+--------------+
| Sending data         | 1.217784 | 99.98 | 1.2177840000 |
| starting             | 0.000056 |  0.00 | 0.0000560000 |
| cleaning up          | 0.000050 |  0.00 | 0.0000500000 |
| freeing items        | 0.000046 |  0.00 | 0.0000460000 |
| query end            | 0.000022 |  0.00 | 0.0000220000 |
| end                  | 0.000022 |  0.00 | 0.0000220000 |
| closing tables       | 0.000020 |  0.00 | 0.0000200000 |
| Opening tables       | 0.000015 |  0.00 | 0.0000150000 |
| statistics           | 0.000010 |  0.00 | 0.0000100000 |
| init                 | 0.000010 |  0.00 | 0.0000100000 |
| preparing            | 0.000009 |  0.00 | 0.0000090000 |
| System lock          | 0.000008 |  0.00 | 0.0000080000 |
| checking permissions | 0.000007 |  0.00 | 0.0000070000 |
| optimizing           | 0.000005 |  0.00 | 0.0000050000 |
| executing            | 0.000004 |  0.00 | 0.0000040000 |
+----------------------+----------+-------+--------------+
15 rows in set (0.01 sec)

```
然后可以选择all、cpu、block io、context、switch、page faults 等查看mysql到底在用什么资源上耗费了这么高的时间
```sql
mysql> show profile all for query 10;
mysql> show profile cpu,block io for query 10 ;
+----------------------+----------+----------+------------+--------------+---------------+
| Status               | Duration | CPU_user | CPU_system | Block_ops_in | Block_ops_out |
+----------------------+----------+----------+------------+--------------+---------------+
| starting             | 0.000056 | 0.000000 |   0.000000 |            0 |             0 |
| checking permissions | 0.000007 | 0.000000 |   0.000000 |            0 |             0 |
| Opening tables       | 0.000015 | 0.000000 |   0.000000 |            0 |             0 |
| init                 | 0.000010 | 0.000000 |   0.000000 |            0 |             0 |
| System lock          | 0.000008 | 0.000000 |   0.000000 |            0 |             0 |
| optimizing           | 0.000005 | 0.000000 |   0.000000 |            0 |             0 |
| statistics           | 0.000010 | 0.000000 |   0.000000 |            0 |             0 |
| preparing            | 0.000009 | 0.000000 |   0.000000 |            0 |             0 |
| executing            | 0.000004 | 0.000000 |   0.000000 |            0 |             0 |
| Sending data         | 1.217784 | 1.205816 |   0.000000 |            0 |             0 |
| end                  | 0.000022 | 0.000000 |   0.000000 |            0 |             0 |
| query end            | 0.000022 | 0.000000 |   0.000000 |            0 |             0 |
| closing tables       | 0.000020 | 0.000000 |   0.000000 |            0 |             0 |
| freeing items        | 0.000046 | 0.000000 |   0.000000 |            0 |             0 |
| cleaning up          | 0.000050 | 0.000000 |   0.001000 |            0 |             0 |
+----------------------+----------+----------+------------+--------------+---------------+
15 rows in set, 1 warning (0.00 sec)

mysql>
```
<font color="#f00">我们发现真个时间主要耗在CPU上了。但是MyISAM 引擎的表在executing 后直接就结束了查询，完全不不要访问数据。</font>