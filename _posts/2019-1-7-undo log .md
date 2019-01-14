

## MySQL · 引擎特性 · InnoDB undo log 漫游

本文是对整个Undo生命周期过程的阐述，代码分析基于当前最新的MySQL5.7版本。本文也可以作为了解整个Undo模块的代码导读。由于涉及到的模块众多，因此部分细节并未深入。 

## 前言

Undo log是InnoDB MVCC事务特性的重要组成部分。当我们对记录做了变更操作时就会产生undo记录，Undo记录默认被记录到系统表空间(ibdata)中，但从5.6开始，也可以使用独立的Undo 表空间。

Undo记录中存储的是老版本数据，当一个旧的事务需要读取数据时，为了能读取到老版本的数据，需要顺着undo链找到满足其可见性的记录。当版本链很长时，通常可以认为这是个比较耗时的操作（例如[bug#69812](http://bugs.mysql.com/bug.php?id=69812)）。

大多数对数据的变更操作包括INSERT/DELETE/UPDATE，其中INSERT操作在事务提交前只对当前事务可见，因此产生的Undo日志可以在事务提交后直接删除（谁会对刚插入的数据有可见性需求呢！！），而对于UPDATE/DELETE则需要维护多版本信息，在InnoDB里，UPDATE和DELETE操作产生的Undo日志被归成一类，即update_undo。

## 基本文件结构

为了保证事务并发操作时，在写各自的undo log时不产生冲突，InnoDB采用回滚段的方式来维护undo log的并发写入和持久化。回滚段实际上是一种 Undo 文件组织方式，每个回滚段又有多个undo log slot。具体的文件组织方式如下图所示： 

