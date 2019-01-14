---
layout:     post
title:      "InnoDB-文件"
author:     "liulei"
header-img: "img/post-bg-miui6.jpg"
tags:
    - InnoDB
---
# inooDB 文件  

```
   命令行方式：启动与关闭 msyql 
        开启  ./mysqld_safe 
        关闭  mysqladmin -uroot -p  shutdown
```



## 参数文件

Mysql 启动时，数据库会去读一个配置参数文件，用来寻找数据库的各种文件所在位置以及指定某些初始化参数，这些参数通常定义了某种内存结构有多大等。

命令,查询Mysql 实例读取配置文件的顺序

```sql
mysql --help | grep my.cnf
```



结果：

```sql
 order of preference, my.cnf, $MYSQL_TCP_PORT,
/etc/my.cnf /etc/mysql/my.cnf /home/m201873306/eclipse-workspace/software/mysql/etc/my.cnf ~/.my.cnf 

```

与 oracle  不同的是，启动时如果没有参数文件，可以采用默认的值启动（源代码中指定的）。如果没有则会启动失败。

### 相关参数

所有参数都是键/值对的形式

可以通过命令查询数据库中所有参数：

```sql
mysql> show variables\G
```

结果：506 条记录

```sql
*************************** 1. row ***************************
Variable_name: auto_increment_increment
        Value: 1
*************************** 2. row ***************************
Variable_name: auto_increment_offset
        Value: 1
*************************** 3. row ***************************
Variable_name: autocommit
        Value: ON
        *
        *
        *
*************************** 502. row ***************************
Variable_name: version_comment
        Value: Source distribution
*************************** 503. row ***************************
Variable_name: version_compile_machine
        Value: x86_64
*************************** 504. row ***************************
Variable_name: version_compile_os
        Value: Linux
*************************** 505. row ***************************
Variable_name: wait_timeout
        Value: 28800
*************************** 506. row ***************************
Variable_name: warning_count
        Value: 0
506 rows in set (0.00 sec)

```

可以通过like过滤变量

```sql
mysql> show variables like 'innodb_buffer%'\G
```

结果：

```
*************************** 1. row ***************************
Variable_name: innodb_buffer_pool_chunk_size
        Value: 134217728
*************************** 2. row ***************************
Variable_name: innodb_buffer_pool_dump_at_shutdown
        Value: ON
*************************** 3. row ***************************
Variable_name: innodb_buffer_pool_dump_now
        Value: OFF
*************************** 4. row ***************************
Variable_name: innodb_buffer_pool_dump_pct
        Value: 25
*************************** 5. row ***************************
Variable_name: innodb_buffer_pool_filename
        Value: ib_buffer_pool
*************************** 6. row ***************************
Variable_name: innodb_buffer_pool_instances
        Value: 1
*************************** 7. row ***************************
Variable_name: innodb_buffer_pool_load_abort
        Value: OFF
*************************** 8. row ***************************
Variable_name: innodb_buffer_pool_load_at_startup
        Value: ON
*************************** 9. row ***************************
Variable_name: innodb_buffer_pool_load_now
        Value: OFF
*************************** 10. row ***************************
Variable_name: innodb_buffer_pool_size
        Value: 134217728
10 rows in set (0.00 sec)

```

**Oracle 与 SQL server 都存在隐藏函数，而mysql 中没有**

### 参数类型

1. **动态参数**(有些参数修改完后可以在整个实例生命周期都会生效，有的只能在会话中更改)

可以在Mysql 实例运行中进行更改，可以通过`SET`命令来进行更改

```
set read_buffer_size = 524288;  //修改了 sesison  变量 
```

```
mysql> select @@session.read_buffer_size;
+----------------------------+
| @@session.read_buffer_size |
+----------------------------+
|                     524288 |
+----------------------------+
1 row in set (0.00 sec)

mysql> select @@global.read_buffer_size;
+---------------------------+
| @@global.read_buffer_size |
+---------------------------+
|                    131072 |
+---------------------------+
1 row in set (0.00 sec)

```

结果中 @@global 未改变

需要用：

```sql
mysql> set  global read_buffer_size = 1048576;
```

来改变 。

```sql
mysql> select @@global.read_buffer_size;
+---------------------------+
| @@global.read_buffer_size |
+---------------------------+
|                   1048576 |
+---------------------------+
1 row in set (0.00 sec)

// 而 session   值还是 512KB 
mysql> select @@session.read_buffer_size;
+----------------------------+
| @@session.read_buffer_size |
+----------------------------+
|                     524288 |
+----------------------------+
1 row in set (0.00 sec)
```

2. **静态参数**

在整个实例运行中不可以更改，显示为 read only  变量



## 日志文件

### 错误日志

编译运行的错误日志默认

```mysql
mysql> show variables like 'log_error'\G;
*************************** 1. row ***************************
Variable_name: log_error
        Value: stderr // 默认 错误文件输出到控制台中
1 row in set (0.00 sec)
```

修改 /etc/my.cnf   添加了

```
log_error=/home/m201873306/eclipse-workspace/software/mysql/data/liulei.err
```

重启数据库

```mysql
m201873306@m201873306-VirtualBox:~/eclipse-workspace/software/mysql/bin$ ./mysqld_safe 
2018-11-26T08:37:42.945828Z mysqld_safe Logging to '/home/m201873306/eclipse-workspace/software/mysql/data/liulei.err'.   // 可以看到输入到 liulei.err 中
2018-11-26T08:37:42.995146Z mysqld_safe Starting mysqld daemon with databases from /home/m201873306/eclipse-workspace/software/mysql/data  //并且获取 data 中表的数据
```

再次查询

```
mysql> show variables like '%error'\G
*************************** 1. row ***************************
Variable_name: log_error
        Value: /home/m201873306/eclipse-workspace/software/mysql/data/liulei.err
1 row in set (0.01 sec)
```



错误日志文件可以对Mysql 的**启动、运行、关闭**过程进行记录，不仅记录了所有的错误信息，也记录了一些**警告信息和正确信息**。

这里查看下 错误 日志的  后10 条信息

```mysql
m201873306@m201873306-VirtualBox:~/eclipse-workspace/software/mysql/data$ tail -n10 liulei.err 
```

结果为：具有 note  与 warning  与 error 

```
2018-11-26T08:37:43.418041Z 0 [Note] IPv6 is available.
2018-11-26T08:37:43.418044Z 0 [Note]   - '::' resolves to '::';
2018-11-26T08:37:43.418053Z 0 [Note] Server socket created on IP: '::'.
2018-11-26T08:37:43.424366Z 0 [Note] Event Scheduler: Loaded 0 events
2018-11-26T08:37:43.424451Z 0 [Note] /home/m201873306/eclipse-workspace/software/mysql/bin/mysqld: ready for connections.
Version: '5.7.19'  socket: '/tmp/mysql.sock'  port: 3306  Source distribution
2018-11-26T08:37:43.424457Z 0 [Note] Executing 'SELECT * FROM INFORMATION_SCHEMA.TABLES;' to get a list of tables using the deprecated partition engine. You may use the startup option '--disable-partition-engine-check' to skip this check. 
2018-11-26T08:37:43.424458Z 0 [Note] Beginning of list of non-natively partitioned tables
2018-11-26T08:37:43.440602Z 0 [Note] End of list of non-natively partitioned tables
2018-11-26T08:32:58.954169Z 0 [Warning] Changed limit
```

### 慢查询日志

>慢查询可以帮助定位可能存在问题的SQL 语句。

查看相关变量

```mysql
mysql> set global slow_query_log = on;  // 默认关闭
mysql> show variables like 'slow%'\G 
*************************** 1. row ***************************
Variable_name: slow_launch_time
        Value: 2                        // 创建线程时间的阈值
*************************** 2. row ***************************
Variable_name: slow_query_log
        Value: ON                       // 开启slow query 
*************************** 3. row ***************************
Variable_name: slow_query_log_file   // 存放位置
        Value: /home/m201873306/eclipse-workspace/software/mysql/data/m201873306-VirtualBox-slow.log
3 rows in set (0.00 sec)


mysql> show variables like 'long_query_time'\G
*************************** 1. row ***************************
Variable_name: long_query_time
        Value: 10.000000                   // 表示超过这个值得sql 语句会记录进去
1 row in set (0.00 sec)

mysql> show variables like 'log_quer%'\G
*************************** 1. row ***************************
Variable_name: log_queries_not_using_indexes   
        Value: OFF                 //  如果是 on 那么运行的sql 语句 没有使用索引 将会记录到 log 里
1 row in set (0.00 sec)

mysql> set global log_queries_not_using_indexes = on;
Query OK, 0 rows affected (0.01 sec)

mysql> show variables like 'log_quer%'\G
*************************** 1. row ***************************
Variable_name: log_queries_not_using_indexes
        Value: ON
1 row in set (0.00 sec)

mysql> show variables like 'log_throttle%'\G
*************************** 1. row ***************************
Variable_name: log_throttle_queries_not_using_indexes
        Value: 0           // 表示每分钟记录到slow log 且未使用索引的SQL 语句次数，当 0时表示没有限制。如果不限制的话会导致slow_log文件不断增大
1 row in set (0.00 sec)

```

查看 slow _log

```
mysql> select sleep(11);
+-----------+
| sleep(11) |
+-----------+
|         0 |
+-----------+
1 row in set (11.00 sec)

mysql> select sleep(11);
+-----------+
| sleep(11) |
+-----------+
|         0 |
+-----------+
1 row in set (11.00 sec)

mysql> select sleep(12);
+-----------+
| sleep(12) |
+-----------+
|         0 |
+-----------+
1 row in set (12.00 sec)

```

```mysql
m201873306@m201873306-VirtualBox:~/eclipse-workspace/software/mysql/bin$ mysqldumpslow ../data/m201873306-VirtualBox-slow.log

Reading mysql slow query log from ../data/m201873306-VirtualBox-slow.log
Count: 3  Time=11.34s (34s)  Lock=0.00s (0s)  Rows=1.0 (3), root[root]@localhost
  select sleep(N)
```

slow_log 可以放在表中，默认为File,修改`log_output` 可以在表中显示，在`mysql.slow_log`表中

```
mysql> set global log_output = 'TABLE';
Query OK, 0 rows affected (0.00 sec)
mysql> select sleep(11);
+-----------+
| sleep(11) |
+-----------+
|         0 |
+-----------+
1 row in set (11.00 sec)
mysql> select * from mysql.slow_log \G;
*************************** 1. row ***************************
    start_time: 2018-11-26 17:22:32.546006
     user_host: root[root] @ localhost []
    query_time: 00:00:00.000106
     lock_time: 00:00:00.000048
     rows_sent: 0
 rows_examined: 0
            db: 
last_insert_id: 0
     insert_id: 0
     server_id: 0
      sql_text: select * from mysql.slow_log
     thread_id: 3
*************************** 2. row ***************************
    start_time: 2018-11-26 17:22:54.428821
     user_host: root[root] @ localhost []
    query_time: 00:00:11.001921
     lock_time: 00:00:00.000000
     rows_sent: 1
 rows_examined: 0
            db: 
last_insert_id: 0
     insert_id: 0
     server_id: 0
      sql_text: select sleep(11)
     thread_id: 3

```

###查询日志

 查询日志记录了所有对 Mysql 数据库 请求的信息，无论这些请求是否得到了正确的执行。`general_log`默认是OFF ，

```
mysql> show variables like 'general%';
+------------------+----------------------------------------------------------------------------------+
| Variable_name    | Value                                                                            |
+------------------+----------------------------------------------------------------------------------+
| general_log      | OFF                                                                              |
| general_log_file | /home/m201873306/eclipse-workspace/software/mysql/data/m201873306-VirtualBox.log |
+------------------+----------------------------------------------------------------------------------+
2 rows in set (0.00 sec)
mysql> set global general_log = on;
Query OK, 0 rows affected (0.00 sec)
```

将日志输出改为 FILE,TABLE

```
mysql> set global log_output = 'TABLE,FILE';
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like 'log_output%';
+---------------+------------+
| Variable_name | Value      |
+---------------+------------+
| log_output    | FILE,TABLE |
+---------------+------------+
1 row in set (0.00 sec)
```



查询在 mysql.general_log 表中

```
mysql> select * from mysql.general_log \G
*************************** 1. row ***************************
  event_time: 2018-11-26 18:28:42.216988
   user_host: root[root] @ localhost []
   thread_id: 3
   server_id: 0
command_type: Query
    argument: set global general_log = on
*************************** 2. row ***************************
  event_time: 2018-11-26 18:30:41.521245
   user_host: root[root] @ localhost []
   thread_id: 3
   server_id: 0
command_type: Query
    argument: show variables like 'log_output%'
*************************** 3. row ***************************
  event_time: 2018-11-26 18:31:26.353436
   user_host: root[root] @ localhost []
   thread_id: 3
   server_id: 0
command_type: Query
    argument: set global log_output = 'TABLE,FILE'
*************************** 4. row ***************************
  event_time: 2018-11-26 18:31:28.217532
   user_host: root[root] @ localhost []
   thread_id: 3
   server_id: 0
command_type: Query
    argument: show variables like 'log_output%'
*************************** 5. row ***************************
  event_time: 2018-11-26 18:31:40.721708
   user_host: root[root] @ localhost []
   thread_id: 3
   server_id: 0
command_type: Query
    argument: show variables like 'log_output%'
*************************** 6. row ***************************
  event_time: 2018-11-26 18:33:28.041196
   user_host: root[root] @ localhost []
   thread_id: 3
   server_id: 0
command_type: Query
    argument: select * from mysql.general_log
*************************** 7. row ***************************
  event_time: 2018-11-26 18:34:42.673424
   user_host: root[root] @ localhost []
   thread_id: 3
   server_id: 0
command_type: Query
    argument: select * from mysql.general_log
7 rows in set (0.00 sec)

```

###二进制日志

记录了数据库执行**更改**的所有操作，对数据本身**可以**产生修改的语句。但是即使操作本身没有使数据库产生变化，那么该操作可能也会写入二进制日志。

**TODO** 



## 套接字文件

unix 套接字不是一个网络协议，只能在Mysql 客户端和数据库实例在一台服务器上的情况下使用。

```
mysql> show variables like 'socket' \G
*************************** 1. row ***************************
Variable_name: socket
        Value: /tmp/mysql.sock
1 row in set (0.01 sec)

```



## pid 文件

每次Mysql 实例启动时会将自己的进程ID 写入一个文件中。后缀名` .pid`

## 表结构定义文件

Mysql  数据的存储是根据表进行的，每个表都会有与之对应的文件。

其中`.frm` 结尾的文件 记录了 表的表结构定义

```
hexdump -C db.frm | less  // 可以用hexdump 查看
```

`.frm`还可以用来存放视图的定义。

## InnoDB 存储引擎文件

每个存储引擎的存储文件都不同，如 myisam  存储表的数据文件后缀名为`.MYD`

存放表索引的文件后缀名为`.MYI`;如 InooDB 存储表数据和索引都放在一个文件中

`ibdata`后缀名为`.ibd`

InnoDB 存储引擎密切相关的文件包括重做日志文件、表空间文件。

### 表空间文件

InnoDB 采用将存储的数据 **按表空间**（tablespace）进行存放，

```
mysql> show variables like 'innodb_data%' \G
*************************** 1. row ***************************
Variable_name: innodb_data_file_path
        Value: ibdata1:12M:autoextend   
```

默认情况下初始化一个ibdata1 文件 ，并且可以自动增长，该文件就是默认的表空间文件。可以通过`innodb_data_file_path`设置 ，可以由一个或者多个文件组成表空间。

同时 还有一个参数：

```
*************************** 4. row ***************************
Variable_name: innodb_file_per_table
        Value: ON
4 rows in set (0.00 sec)
```

这个参数`innodb_file_per_table`的 值为`on `表示每个表还会产生一个独立表空间，独立表空间的命名规则为`表名.ibd`。

> 总的来说，InnoDB 在默认配置下，表的存储地方分为共享表空间和独立表空间。InnoDB 将每个新创建的**表的数据及索引**存储在一个独立的`.ibd`文件。这时在数据库文件夹下每个表会存在一个`.frm`文件和`.ibd`文件。其余的数据存放在共享表空间（ibdata1）中。

#### 共享表空间与独立表空间

共享表空间：

1. **优点**： 可以将表空间分成多个文件存放到各个磁盘上（表空间文件大小不受表大小的限制，如一个表可以分布在不同的文件上）。数据和文件放在一起方便管理。 
2. **缺点**：所有的数据和索引存放到一个文件中，虽然可以把一个大文件分成多个小文件，但是多个表及索引在表空间中混合存储，这样对于一个表做了大量删除操作后表空间中将会有大量的空隙，特别是对于统计分析，日值系统这类应用最不适合用共享表空间。 

独立表空间：

1. **优点**：每个表都有自已独立的表空间。 每个表的数据和索引都会存在自已的表空间中。  可以实现单表在不同的数据库中移动。 空间可以回收（除drop table操作处，表空不能自已回收） 。

2. **缺点**：单表增加过大，如超过100个G。 

   

   **相比较之下，使用独占表空间的效率以及性能会更高一点。**

### 重做日志文件

![](https://images0.cnblogs.com/i/572361/201405/181533184212320.png)

默认情况下，InnoDB 存储引擎的目录下会有两个`ib_logfile0`和`ib_logfile1`文件，逻辑上呗当成一个文件。这两个文件定义为 `redo log file `重做日志文件。

```mysql
*************************** 8. row ***************************
Variable_name: innodb_log_file_size
        Value: 50331648    //每个重做日志文件大小 ，大小50.3M
*************************** 9. row ***************************
Variable_name: innodb_log_files_in_group
        Value: 2           // 日志文件组中重做日志文件的数量 
*************************** 10. row ***************************
Variable_name: innodb_log_group_home_dir
        Value: ./   //表示存放在/data 下

```

`innodb_log_file_size` 文件的大小不能设置过大，如果设置过大，恢复时可能需要很长时间；如果设置过小，否则可能导致一个事务的日志需要多次切换重做日志文件。同时重做日志文件太小的话会导致频繁的async checkpoint ，导致性能的抖动。重做日志有一个`log_group_capacity `变量，该值代表了 `checkpoint_age`（表示redo log 中还未持久化的日志）不能超过这个值，如果**超过**则必须将缓冲池（innodb buffer pool)中脏页列表（flush list）中部分脏页写会磁盘。因为此时有可能会复用redo log。

#### Checkpoint  机制         

​     在Innodb事务日志中，采用了Fuzzy Checkpoint，Innodb每次取最老的modified page(last checkpoint)对应的LSN，再将此脏页的LSN作为Checkpoint点记录到日志文件，意思就是“此LSN之前的LSN对应的日志和数据都已经flush到redo log

当mysql crash的时候，Innodb扫描redo log，从last checkpoint开始apply redo log到buffer pool，直到**last checkpoint**对应的LSN等于**Log flushed up to**对应的LSN，则恢复完成。

**日志生命周期：**

![](https://images2015.cnblogs.com/blog/268981/201601/268981-20160108210013918-92051279.png)

Innodb 一条事务日志 经过四个阶段才算结束：也就是说这条redo 日志可以删除

1. **创建阶段**：事务创建一条日志 对应于`Log sequence number` （当前系统LSN最大值 ）= 之前系统LSN最大值+这条事务日志的大小
2. **日志刷盘**：日志写入到磁盘上的日志文件，对应于`Log flushed up to `（ 当前已经写入日志文件的LSN）；
3. **数据刷盘**：日志对应的脏页数据写入到磁盘上的数据文件,对应于`Oldest modified data log `，表示flush list 中最旧的脏页，也就是 LSN最小的脏页。
4. **写CKP**：Checkpoint 写入 redo 日志文件。checkpoint =`last checkpoint at `为 `Oldest modified data log ` 的值 。



`lsn_t log_group_capacity ` 为 log_sys 中表示当前日志文件的总的容量。

1. **Sharp Checkpoint**

 发生在数据库关闭时将所有脏页都刷新回磁盘。数据库默认的方式，即`innodb_fast_shutdown`=1。

2. **Fuzzy Checkpoint** 

   运行时 的刷新机制，只刷新部分脏页，而不是刷新所有的脏页回磁盘。包括几种情况：

   1. Master Thread Checkpoint ：

      每秒和每十秒都会刷新脏页。

   2. FLUSH_LRU_LIST Checkpoint 

      innodb 维持了一个 `innodb_buffer_pool`，利用LRU算法来管理**已经读取的页**（自适应哈希索引、look信息、Insert Buffer 不需要LRU算法），即最频繁的使用的页（默认为16K）在LRU列表的前端，最少使用的页在LRU的尾端。`Free`列表存放着可用的空闲页。Innodb 存储引擎会保证 `LRU`列表将会有一定数量的可用页（由`innodb_lru_scan_depth`决定，默认是1024）,故如果不够的话，就需要将LRU列表中尾端的页移除掉。如果这些页有脏页的话，就需要进行checkpoint ，由于这些页来自LRU列表，故称为`FLUSH_LRU_LIST Checkpoint `，解决了缓冲池不够的问题。

   3. Async/Sync Flush Checkpoint

      **指的是重做日志空间不够的情况下**，进行checkpoint，其中`redo_lsn` 表示已经写入到重做日志的LSN，`checkpoint_lsn`表示刷新回磁盘的LSN。`chekpoint_age`= `redo_lsn`-`checkpoint_lsn`  表示`redo log`文件中未持久化的字节数。如果这个数大于日志总容量的话，就会发生覆盖日志，导致未刷新的脏页的日志丢失。其中 `max_checkpoint_age_async`=`log_group_capacity` *75%

      `max_modified_age_sync `= `log_group_capacity`*90%

      其中当 `chekpoint_age`在`max_checkpoint_age_async`与`max_modified_age_sync `之间时进行异步刷脏。大于`max_modified_age_sync `时同步刷脏。这时从`FLush 列表`刷脏。

   4. Dirty Page too much Checkpoint 

      这种也是保证了缓冲池中有足够的页，`innodb_max_dirty_pages_pct`表示脏页比例，默认为75% ，当为75%的时候，进行checkpoint。 

[参考链接](https://yq.aliyun.com/articles/219?spm=5176.100240.searchblog.93)

