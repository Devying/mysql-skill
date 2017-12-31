# 索引优化

MyISAM 存储引擎的表的数据和索引是自动分开存储的，各自是独立的一个文件；InnoDB
存储引擎的表的数据和索引是存储在同一个表空间里面，但可以有多个文件组成。

## MyISAM Innodb Memory 引擎支持的索引

|           | MyISAM  | Innodb | Memory |
| :-------- | :------ | :----  | :----- |
| B-Tree    | ✔       | ✔     | ✔      |
| HASH      | ------  | ------ | ✔      |
| R-Tree    | ✔      | ------ | ------ |
| Full-text | ✔      | ------ | ------ |

>可以对某一个字段的前几个字符做索引
>```sql
>mysql> create index idx_name on tbl_course(name(5));
>Query OK, 0 rows affected (0.10 sec)
>Records: 0  Duplicates: 0  Warnings: 0
>```

## 查看索引的使用情况
```sql
mysql> show global status like 'Handler_read%';
+-----------------------+------------+
| Variable_name         | Value      |
+-----------------------+------------+
| Handler_read_first    | 1612525    |
| Handler_read_key      | 20789850   |
| Handler_read_last     | 106802     |
| Handler_read_next     | 5742717003 |
| Handler_read_prev     | 93137047   |
| Handler_read_rnd      | 899539     |
| Handler_read_rnd_next | 4901923226 |
+-----------------------+------------+
7 rows in set (0.00 sec)
```
假如索引正在工作Handler_read_key的值很高，这个值代表了一个行被索引值读的次数，很低的值表明增加索引得到的性能改善不高，因为索引并不经常使用。
Handler_read_rnd_next 的值越高意味着查询运行低效，应该建立索引补救。这个值得含义是在数据文件中读取下一行的请求数。如果正在进行大量的表扫描，它的值比较高，通常说明索引不正确或者写入的查询没有利用索引。


## 使用索引
### 匹配全值
对于索引中所有列都指定具体指，也就是对索引中的所有列都有等值匹配的条件。
```sql
| tbl_course_member | CREATE TABLE `tbl_course_member` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `course_id` int(11) unsigned NOT NULL DEFAULT '0' COMMENT '课程ID',
  `project_id` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '项目ID',
  `uid` int(11) unsigned NOT NULL DEFAULT '0' COMMENT '用户ID',
  `create_time` int(11) unsigned NOT NULL DEFAULT '0',
  `status` tinyint(2) NOT NULL DEFAULT '0' COMMENT '0正常 -1删除',
  `update_time` int(11) unsigned NOT NULL DEFAULT '0',
  `media_id` int(11) unsigned NOT NULL DEFAULT '0' COMMENT '节ID',
  `learn_time` int(11) unsigned NOT NULL DEFAULT '0' COMMENT '总学习时长',
  `chapter_media` char(16) DEFAULT '0' COMMENT '章节如1-1',
  `learn_rate` tinyint(3) NOT NULL DEFAULT '0' COMMENT '学习进度百分比',
  `mp` int(11) NOT NULL DEFAULT '0' COMMENT '经验',
  `is_push_comment` tinyint(3) DEFAULT '0' COMMENT '是否推送过评价',
  PRIMARY KEY (`id`),
  KEY `idx_uid` (`uid`),
  KEY `rate` (`learn_rate`),
  KEY `unique_courseid_uid_status` (`course_id`,`uid`,`status`)
) ENGINE=InnoDB AUTO_INCREMENT=8903 DEFAULT CHARSET=utf8 |

现在新增符合索引
mysql> ALTER TABLE `shizhan_course_study`.`tbl_course_member` ADD INDEX `idx_cid_uid_mid` (`course_id`, `uid`, `media_id`);
Query OK, 0 rows affected (0.07 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> mysql> show index from tbl_course_member;
+-------------------+------------+-----------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table             | Non_unique | Key_name        | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-------------------+------------+-----------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| tbl_course_member |          0 | PRIMARY         |            1 | id          | A         |        8724 |     NULL | NULL   |      | BTREE      |         |               |
| tbl_course_member |          1 | idx_cid_uid_mid |            1 | course_id   | A         |          54 |     NULL | NULL   |      | BTREE      |         |               |
| tbl_course_member |          1 | idx_cid_uid_mid |            2 | uid         | A         |        8724 |     NULL | NULL   |      | BTREE      |         |               |
| tbl_course_member |          1 | idx_cid_uid_mid |            3 | media_id    | A         |        8724 |     NULL | NULL   |      | BTREE      |         |               |
+-------------------+------------+-----------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
4 rows in set (0.00 sec)

mysql> 


来做一次查询分析

mysql> EXPLAIN SELECT * FROM tbl_course_member where course_id = 31 and uid = 100056 and media_id = 234\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_course_member
         type: ref
possible_keys: idx_cid_uid_mid
          key: idx_cid_uid_mid
      key_len: 12
          ref: const,const,const
         rows: 1
        Extra: NULL
1 row in set (0.00 sec)

```

### 匹配范围
使用复合索引中的部分字段
```sql
mysql> EXPLAIN SELECT * FROM tbl_course_member where course_id >88\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_course_member
         type: range
possible_keys: idx_cid_uid_mid
          key: idx_cid_uid_mid
      key_len: 4
          ref: NULL
         rows: 84
        Extra: Using index condition
1 row in set (0.00 sec)

或者用单独的一个字段加索引也是这样的

mysql> EXPLAIN SELECT * FROM tbl_course_member where uid >88 and uid < 155\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_course_member
         type: range
possible_keys: idx_uid
          key: idx_uid
      key_len: 4
          ref: NULL
         rows: 1
        Extra: Using index condition
1 row in set (0.00 sec)


```
### 匹配最左侧
这个我们都听说过，就是对于符合索引来说，遵循这个规律，
```sql
mysql> EXPLAIN SELECT * FROM tbl_course_member where course_id =88 and media_id = 155\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_course_member
         type: ref
possible_keys: idx_cid_uid_mid
          key: idx_cid_uid_mid
      key_len: 4
          ref: const
         rows: 1
        Extra: Using index condition
1 row in set (0.00 sec)

mysql> EXPLAIN SELECT * FROM tbl_course_member where course_id =88\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_course_member
         type: ref
possible_keys: idx_cid_uid_mid
          key: idx_cid_uid_mid
      key_len: 4
          ref: const
         rows: 1
        Extra: NULL
1 row in set (0.00 sec)

mysql> EXPLAIN SELECT * FROM tbl_course_member where uid =88 and media_id = 123\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_course_member
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 8724
        Extra: Using where
1 row in set (0.00 sec)


```

### 查询索引字段

假如我们查询的字段都是索引，那么查询的效率就更好
```sql
mysql> explain select media_id from tbl_course_member where course_id =51\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_course_member
         type: ref
possible_keys: idx_cid_uid_mid
          key: idx_cid_uid_mid
      key_len: 4
          ref: const
         rows: 856
        Extra: Using index
1 row in set (0.00 sec)

mysql> explain select media_id from tbl_course_member where course_id >=51\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_course_member
         type: range
possible_keys: idx_cid_uid_mid
          key: idx_cid_uid_mid
      key_len: 4
          ref: NULL
         rows: 1750
        Extra: Using where; Using index
1 row in set (0.00 sec)
```
### 匹配列前缀
比如我们做一个存储文章信息的表，我们把title的前10个字符和content的前20个字符拿来加一个符合索引。
```sql
create index idx_title_content_part on tbl_article (title(10),content(20));

explain select title from tbl_article where title like '我们%'\G

type:range
Extra:using where
表示这里需要通过索引回表查询数据
### 

```

### 复合索引阻断
就是说利用复合索引（a+b+c）查询条件是a = val1 and c > val2 and c < val3

```sql
mysql> explain select media_id from tbl_course_member where course_id =51 and media_id>=1130 and media_id <= 1430\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_course_member
         type: ref
possible_keys: idx_cid_uid_mid
          key: idx_cid_uid_mid
      key_len: 4
          ref: const
         rows: 856
        Extra: Using where; Using index
1 row in set (0.00 sec)

set optimizer_switch='index_condition_pushdown=on'; (索引下沉 5.6)开启这个之后

mysql> explain * from tbl_course_member where course_id =51 and media_id>=1130 and media_id <= 1430\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_course_member
         type: ref
possible_keys: idx_cid_uid_mid
          key: idx_cid_uid_mid
      key_len: 4
          ref: const
         rows: 856
        Extra: Using index condition
1 row in set (0.00 sec)

```
### 列是索引 列 is null 会用到索引

## 有索引但是不使用索引的情况

### 以%开头的LIKE查询不能够利用B-Tree索引，执行计划中key为NULL 表示没有使用索引

比如我们给一个表的name字段加上索引然后用一下sql
```sql
mysql> explain * from tbl_course where name like "%测试"\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_course
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 135
        Extra: Using where
1 row in set (0.00 sec)

可以看出来这里是扫描了全表

我们来改造一下

mysql> explain select * from (select id from tbl_course where name like "%测试%") a, tbl_course b where a.id = b.id\G
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: <derived2>
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 135
        Extra: NULL
*************************** 2. row ***************************
           id: 1
  select_type: PRIMARY
        table: b
         type: eq_ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: a.id
         rows: 1
        Extra: NULL
*************************** 3. row ***************************
           id: 2
  select_type: DERIVED
        table: tbl_course
         type: index
possible_keys: NULL
          key: name
      key_len: 302
          ref: NULL
         rows: 135
        Extra: Using where; Using index
3 rows in set (0.00 sec)
```

### 隐式转换时不能用到索引比如本来是字符串结果用整形

```sql
mysql> explain select * from tbl_course where name = 1\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_course
         type: ALL
possible_keys: name
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 135
        Extra: Using where
1 row in set (0.00 sec)

mysql> explain select * from tbl_course where name = "1"\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_course
         type: ref
possible_keys: name
          key: name
      key_len: 302
          ref: const
         rows: 1
        Extra: Using index condition
1 row in set (0.00 sec)

```
### 符合索引时如果查询没有包括最左侧部分，不会用到索引
```sql
mysql> explain select * from tbl_course_member where uid =51 and media_id=1130\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_course_member
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 8724
        Extra: Using where
1 row in set (0.00 sec)
```
### 假如MYSQL判定使用索引比全表扫描更慢，则不用索引。

```sql
比如以H开头的行非常多假如全都是以H开头的
mysql> explain select * from tbl_course where name like "H%"\G


```

### 用or分开的条件

比如or前面的条件中的列有索引，而后面的列中没有索引，那么涉及的索引都不会被用到
```sql
mysql> explain select * from tbl_order_goods where trade_number ='17123123252355' or goods_id = 59\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tbl_order_goods
         type: ALL
possible_keys: trade_number
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 86739
        Extra: Using where
1 row in set (0.00 sec)
```



ლ(′◉❥◉｀ლ)