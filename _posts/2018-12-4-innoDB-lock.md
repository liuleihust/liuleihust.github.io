---
layout:     post
title:      "InnoDB-锁"
author:     "liulei"
header-img: "img/post-bg-miui6.jpg"
tags:
    - InnoDB
---


# 锁

> 锁机制用于管理公共资源的**并发访问**。

InnoDB 可以加`行锁`，别的引擎一般是`表锁`或者`页锁`。

InnoDB 提供了一致性的非锁定读、行级锁支持，不像以前会有额外的开销，但是也提供了并发性和一致性。

## lock 与 latch 

latch: 一般称为轻量级的锁，其要求锁定的时间非常短，可以分为 mutex(互斥量)和rwlock（读写锁）。目的是用来保证并发线程操作**临界资源**的正确性，并且通常**没有死锁**检测的机制。

lock:对象通常是事务，用来锁定的是数据库中的对象，如表、页、行。lock 的对象一般在事务commit或rollback后进行释放。具有死锁机制。

##  InnoDB 中的LOCK

利用 `information_schema.INNODB_TRX`查看当前系统事务状态。

另一个session 开启了一个事务，更新了6条记录。

```
*************************** 1. row ***************************
                    trx_id: 35439900    //事务ID
                 trx_state: RUNNING    //事务状态
               trx_started: 2018-11-28 20:25:45   // 事务开启时间
     trx_requested_lock_id: NULL  //
          trx_wait_started: NULL  //事务等待开始的时间
                trx_weight: 8    // 事务的权重，表示一个事务修改和锁住的行数，当发生死锁回滚是，选择改值最小的事务回滚。
       trx_mysql_thread_id: 4  //线程ID
                 trx_query: NULL 
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 7
         trx_rows_modified: 6
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
1 row in set (0.00 sec)
```

## 一致性非锁定读

如果读取的行正在执行的DELETE 或 UPDATE 操作，这时读取操作不会因此去**等待行上锁的释放**，而是去读取一个快照数据。

如果没有锁就直接读取。

一个行记录可能不止一个快照数据，一般称这种技术为 MVCC（多版本并发控制）

只有在事务隔离级别在` Read committed `和`Repeatable read `时使用MVCC，

在 ` Read committed ` 下，非一致性读总是读取**被锁定行**的**最新**的一份快照数据。

在`Repeatable read `下，非一致性读总数读取**事务开始时的行数据版本**。

## 一致性锁定读

```
select...for update   //对读取到的行加一个X锁
select...lock in share mode //对读取到的行加一个S锁
```

## 自增长与锁

InnoDB 中自增长列必须是索引，同时必须是索引的第一个列。

对于自增长列，InnoDB 都有一个自增长计数器，对于自增列的实现方式有两种：

1. **AUTO-INC Locking** ：这种方式每次插入时都会实行表锁，插入结束就释放锁。
2. 采用`mutex`互斥量：对内存中的计数器进行累加操作。对于高并发的情况这种性能最高。但是每次插入时可能不是连续的。

## 外键与锁

InnoDB 存储引擎会对外键列自动加索引好处是：

1. 如果不加索引，当对父表进行更新时候，则在整个更新过程中将整个子表进行锁定，而实际上加了索引后仅需锁定几行。