# show status 命令
> show [session|global]status 命令可以提供服务器状态信息，也可以在操作系统上使用 mysqladmin extended-status 命令获得这些消息。
>
> show[session|global] status 可以根据需要加上参数“session”或者“global”来显示 session 级（当前连接）的统计结果和 global 级（自数据库上次启动至今）的统计结果。如果不写，默认使用参数是“session”

## 重要的几个指标
```sql
mysql> show global status like "Com_%";
```
👉 Com_select：执行 select 操作的次数，一次查询只累加 1。

👉 Com_insert：执行 INSERT 操作的次数，对于批量插入的 INSERT 操作，只累加一次。

👉 Com_update：执行 UPDATE 操作的次数。

👉 Com_delete：执行 DELETE 操作的次数。

根据这些指标我们就可以得出来整个库 读写比例，然后做读写分离。

## 针对innodb 引擎的几个指标
👉 Innodb_rows_read：select 查询返回的行数。

👉 Innodb_rows_inserted：执行 INSERT 操作插入的行数。

👉 Innodb_rows_updated：执行 UPDATE 操作更新的行数。

👉 Innodb_rows_deleted：执行 DELETE 操作删除的行数。

👉 Innodb_row_lock_current_waits：当前锁等待数

👉 Innodb_row_lock_time 从系统启动到现在锁定的总时间长度

👉 Innodb_row_lock_time_avg 每次等待所花平均时间

👉 Innodb_row_lock_time_max 从系统启动到现在等待最长的一次所花的时间

👉 Innodb_row_lock_waits 从系统启动到现在总共等待的次数

## 其他统计的指标

👉 Connections：试图连接 MySQL 服务器的次数。

👉 Uptime：服务器工作时间。

👉 Slow_queries：慢查询的次数