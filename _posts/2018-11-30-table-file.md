---
layout:     post
title:      "InnoDB-表"
author:     "liulei"
header-img: "img/post-bg-miui6.jpg"
tags:
    - InnoDB
---



# 表

[TOC]

## 索引组织表

InnoDB 中，表都是根据**主键**顺序组织存放的，称为**索引组织表**。故每张表都有个主键，主键一般是 非空 唯一 的索引存在的。如果没有则会自动创建一个6字节大小的指针。主键的选择是根据定义索引的顺序选择，而不是建表时的顺序。当 表为 单个列为主键的时候，可利用`_rowid `显示表的主键。

## InnoDB 逻辑存储结构

所有数据都被**逻辑**地存放在一个空间中，称为**表空间**。

表空间由segment 组成，segment 由 extent 组成,extent 由page 组成.page 由row 组成。

![](https://images2015.cnblogs.com/blog/990532/201701/990532-20170116094754739-1789768872.png)

### 表空间

所有数据都存放在表空间，InnoDB由共享表空间 `ibdata1` 与 独立表空间组成`表名.ibd`组成。`innodb_file_per_table`这个参数为`on` 时，每张表的数据可以单独放在独立表空间。

独立表空间`表名.ibd`包括**数据**、**索引**和**插入缓存Bitmap页**。

共享表空间`ibdata1`包含了 undo 信息，插入缓存索引页，**系统事务信息**，二次写缓冲等。

共享表空间中包含了 undo 的信息，undo 的信息不会在rollback 时立刻收回这些空间，而是在以后的purge 中回收这些空间。master thread  每十秒会执行一次full purge 的操作。

### 段

表空间由各个段组成，常见的段有**数据段、索引段、回滚段**等。因为Innodb 存储引擎中表是索引组织的，因此 B+树的叶子节点（Leaf node segment)是数据段，非叶子节点（Non-leaf node segment)存放的是索引段。

### 区

区由连续的页组成，每个区的大小总是1MB，默认情况下，一个区由64个页组成。

**每个段**开始时，先用32个页大小的碎片页（fragment page）来存放数据，使用完这些页后才开始64个页的申请，这样可以节省磁盘容量开销。

例如一个很简单的表

```
mysql> select * from t;
+----+------+
| id | name |
+----+------+
|  1 | xxx  |
|  2 | 000  |
|  3 | 111  |
|  4 | 222  |
|  5 | 222  |
|  6 | 33   |
+----+------+
6 rows in set (0.00 sec)
```

利用 py_innodb_page_infp.py 查看表空间

```
m201873306@m201873306-VirtualBox:~/eclipse-workspace/software/mysql/data$ python2 py_innodb_page_info.py -v test001/t.ibd
page offset 00000000, page type <File Space Header>
page offset 00000001, page type <Insert Buffer Bitmap>
page offset 00000002, page type <File Segment inode>
page offset 00000003, page type <B-tree Node>, page level <0000> // 这个是数据也， page level<0000> 表示B+ 树的层级，0表示叶子节点
page offset 00000000, page type <Freshly Allocated Page>
page offset 00000000, page type <Freshly Allocated Page>
Total number of page: 6:   // 总共6个页，一个页16K
Freshly Allocated Page: 2   // 2个空闲页
Insert Buffer Bitmap: 1  
File Space Header: 1  // 
B-tree Node: 1
File Segment inode: 1

```

当插入数据过多的时候，会导致B+ 树分裂操作 

当 数据段页大于32 个时，新的页会采用区的方式进行空间的申请。表的空间将会成为区的大小（1M）的倍数。

### 页

页是InnoDB 磁盘管理的最小单位。

常见的页类型为：

1. **数据页**（B-tree Node)
2. **undo页** （undo Log Page)
3. **系统页**（System Page)
4. **事务数据页**（Transaction system Page)
5. **插入缓冲位图页**（Insert Buffer Bitmap)
6. **插入缓冲空闲列表页**（Insert Buffer Free List)
7. **未压缩的二进制大对象页**（ Uncompressed BLOG Page)
8. **压缩的二进制大对象页**(comparessed BLOG Page)

### 行

Innodb 存储引擎是 row-oriented ,所以数据是按行进行存放的。但最多允许存放7992行记录。

Innodb  存储行数据具有几种格式，在InnoDB 1.0.X之前，InnoDB存储引擎提供了Compact和Redundant两种格式来存放行记录数据。Redundant是mysql5.0版本之前的行记录存储方式，之后仍然支持这个格式是为了兼容之前版本的格式，5.1之后很少用到了，因为Compact的结构设计比它好得多，compact格式消耗的磁盘空间和备份耗时更小，Redundant相比之下大了一些。compact格式更适用于大多数的业务场景。 

在InnoDB 1.0.X版本开始又引入了新的文件格式(file format)，
以前支持Compact和Redundant格式称为Antelope文件格式，
新引入的文件格式称为Barracuda文件格式。
Barracuda文件格式下拥有两种新的行记录格式：Compressed和Dynamic，
同时，Barracuda文件格式也包括了Antelope所有的文件格式。
这样Barracuda文件格式支持4种row_format：

```
Redundant、Compact、Compressed、Dynamic  //基本都是使用的这种
```

而Antelope文件格式只支持2种row_format：

```
Redundant、Compact  //很少使用
```
> 数据库的实例的作用之一就是根据行的存放格式读取存放的行记录。

查看表的状态信息：

```
mysql> show table status like 't' \G
*************************** 1. row ***************************
           Name: t
         Engine: InnoDB
        Version: 10
     Row_format: Dynamic  
           Rows: 6
 Avg_row_length: 2730
    Data_length: 16384     
Max_data_length: 0
   Index_length: 0
      Data_free: 0
 Auto_increment: 7
    Create_time: 2018-11-18 16:50:11
    Update_time: NULL
     Check_time: NULL
      Collation: utf8_general_ci
       Checksum: NULL
 Create_options: 
        Comment: 
1 row in set (0.00 sec)
```

 

####Compact 行记录格式 

这里仅仅是**数据页**的格式。。

设计目标是高效地存储数据。一个页中存放的行数据越多，其性能就越高。

![](https://images2015.cnblogs.com/blog/990532/201701/990532-20170116112748427-708866698.png)

组成：

1. 变长字段长度列表：记录了变长字段（包括char）实际的长度，逆序存放，如果值为null 则不再该列表中。
2. NULL 标志位：标志该行是否有NULL 值，有则用1表示，该部分所占的字节为1个字节。
3. **记录头信息**（record header):固定为5字节，40位。

|     名称     | 大小（bit) |                             描述                             |
| :----------: | :--------: | :----------------------------------------------------------: |
|     （）     |     1      |                             未知                             |
|     （）     |     1      |                             未知                             |
| deleted_flag |     1      |                       该行是否已被删除                       |
| min_rec_flag |     1      |           为1，如果该记录是预先被定义为最小的记录            |
|   n_owned    |     4      |                      该记录拥有的记录数                      |
|   head_no    |     13     |                  索引堆中该条记录的排序记录                  |
| record_type  |     3      | 记录类型，000表示普通，001表示B+树节点指针，010表示infimum,011表示Supremum,1XX保留 |
| next_record  |     16     |                   页中下一条记录的相对位置                   |
|    Total     |     40     |                                                              |

最后的部分是实际存储每个列的数据。其中NULL 不占任何空间，只占NULL 标志位。**每行数据除了用户定义的列外，还有两个隐藏列**，`事务ID`和`回滚指针列`。若InnoDB 没有定义主键，每行还会增加一个6字节的rowid 列。

创建表：

```
mysql> create table mytest(
    -> t1 varchar(10),
    -> t2 varchar(10),
    -> t3 char(10),
    -> t4 varchar(10)
    -> ) ROW_FORMAT = COMPACT;
Query OK, 0 rows affected (0.03 sec)
mysql> insert into mytest values('a','bb','bb','ccc');
Query OK, 1 row affected (0.01 sec)

mysql> insert into mytest values('d','ee','ee','fff');
Query OK, 1 row affected (0.00 sec)

mysql> insert into mytest values('d',null,null,'fff');
Query OK, 1 row affected (0.00 sec)
```

生成的行数据：

```
0000c070  73 75 70 72 65 6d 75 6d  03 0a 02 01 00 00 00 10  |supremum........|
0000c080  00 2d 00 00 00 07 56 00  00 00 02 1c c3 09 a9 00  |.-....V.........|
0000c090  00 04 06 01 10 61 62 62  62 62 20 20 20 20 20 20  |.....abbbb      |
0000c0a0  20 20 63 63 63 03 0a 02  01 00 00 00 18 00 2b 00  |  ccc.........+.|
0000c0b0  00 00 07 56 01 00 00 02  1c c3 0a aa 00 00 04 2d  |...V...........-|
0000c0c0  01 10 64 65 65 65 65 20  20 20 20 20 20 20 20 66  |..deeee        f|
0000c0d0  66 66 03 01 06 00 00 20  ff 96 00 00 00 07 56 02  |ff..... ......V.|
0000c0e0  00 00 02 1c c3 0f ad 00  00 04 08 01 10 64 66 66  |.............dff|
0000c0f0  66 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |f...............|
```

解析：

```
第一行数据：
03 0a 02 01 // 表示变长字段的实际长度，逆序存放的。 表示顺序变长字段实际 1 2 10 3 个值
00 // null 标志位 ，表示该行没有null值
00 00 10 00 2d //reacord header
00 00 00 07 56 00 // RowID  Innodb 自动创建
00 00 02 1c c3 09 // 事务ID
a9 00 00 04 06 01 10 // ROLL pointer 
61 // 列1数据 'a'
62 62 // 列2 数据'bb'
62 62 20 20 20 20 20 20 20 // 列3 数据'bb' 其余位置由 20 填充
63 63 63 // 列4 数据 'ccc'
第二行数据：
03 0a 02 01
00 
00 00 18 00 2b
00 00 00 07 56 01 
00 00 02 1c c3 0a
aa 00 00 04 2d 01 10
64 
65 65 
65 65 20 20 20 20 20 20 20 20
66 66 66 
第三行：
03 01 // 表示 顺序 实际长度为 1 3 
06 // 06 转换为 00000110 表示 2 3 列为 null
后面不会存储null 的值
```

### 行溢出数据

#### compact 做法

 InnoDB 存储引擎可以将一条记录中的某些数据存储在真正的数据页面之外。一般认为BLOB(二进制大对象)与LOB（大对象）的列类型的存储会将数据存放在数据页面之外，**实际上有时候BLOB 不会讲数据放在溢出页面，但是Varchar 列数据有可能会被存放在行溢出数据。**



```
mysql> create table test1(
    -> a varchar(65535)
    -> ) charset = utf8;
ERROR 1074 (42000): Column length too big for column 'a' (max = 21845); use BLOB or TEXT instead
```

mysql 文档中说明的 支持的是65535字节，这里 `a  varchar(65535)` 代表的是utf8 编码的字符的65535 个数。此外，65535自己还是指所有varchar 列的长度总和。

```
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. This includes storage overhead, check the manual. You have to change some columns to TEXT or BLOBs
```



InnoDB 存储引擎的页为16K ，即16384 字节，一般情况下数据都是存放在B-tree node 中，但是当发生行溢出时，数据存放在页类型Uncompress BLOG页中。

下面演示

```
mysql> create table t(
    -> a varchar(65532)
    -> )charset= latin1;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into t select repeat('a',65532);
Query OK, 1 row affected (0.03 sec)
Records: 1  Duplicates: 0  Warnings: 0
```

利用 py_innodb_page_info.py 查看

```
m201873306@m201873306-VirtualBox:~/eclipse-workspace/software/mysql/data$ python2 py_innodb_page_info.py -v  test001/t.ibd
page offset 00000000, page type <File Space Header>
page offset 00000001, page type <Insert Buffer Bitmap>
page offset 00000002, page type <File Segment inode>
page offset 00000003, page type <B-tree Node>, page level <0000>
page offset 00000004, page type <Uncompressed BLOB Page>
page offset 00000005, page type <Uncompressed BLOB Page>
page offset 00000006, page type <Uncompressed BLOB Page>
page offset 00000007, page type <Uncompressed BLOB Page>
page offset 00000008, page type <Uncompressed BLOB Page>
Total number of page: 9:
Insert Buffer Bitmap: 1
Uncompressed BLOB Page: 5
File Space Header: 1
B-tree Node: 1
File Segment inode: 1
```

可以看到 表空间中有一个 `B-tree Node`  与 `Uncompressed BLOB Page: 5`5个未压缩的二进制大对象页 Uncompressed  Blog Page 。 这些页实际存放了65532字节的数据。对于行溢出的数据，`B-tree Node`只存取一部分数据，之后是偏移量，指向行溢出页。InnoDB 保证一个页至少能够保存两个行记录，否则将会自动存放到溢出页。

#### Dynamic 与 Compressed 的做法

这两种新纪录格式对于存放在BLOG中的数据采用了**完全的行溢出的方式**。在数据页中只存放了20字节的指针，实际数据都存放在Off Page 中。Compressed 行记录格式还会对行记录进行压缩.

### CHAR 的行结构存储

字段属性的 char(n) 中的n 指的是 字符的长度。如：建表是charset = GBK, 插入`'a'`和插入汉字所占的字节数不同。所以对于多字节字符编码CHAR数据类型的存储，InnoDB 存储引擎在内部将其视为变长字符类型，故在变长长度列表中会记录。

## InnoDB 数据页结构

数据页由 

1. File Header （文件头）38字节
2. Page Header (页头)  56字节
3.  Infimun+Supremum Records（下确界+上确界记录）
4. User Records（用户记录，即行记录） 
5. Free Space（空闲空间）
6. Page Directory（页目录）
7. File Trailer（文件结尾信息） 8字节

![](https://images2015.cnblogs.com/blog/990532/201701/990532-20170116155533052-1549145940.png)

### File Header

记录页的一些头信息，由如下8个部分组成，共占用38个字节 :

`FIL_PAGE_SPACE_OR_CHKSUM`：当MySQL版本小于MySQL-4.0.14，该值代表该页属于哪个表空间，因为如果我们没有开启innodb_file_per_table，共享表空间中可能存放了许多页，并且这些页属于不同的表空间。之后版本的MySQL，该值代表页的checksum值（一种新的checksum值）。

`FIL_PAGE_OFFSET`：表空间中页的偏移值。

`FIL_PAGE_PREV，FIL_PAGE_NEXT`：当前页的上一个页以及下一个页。B+Tree特性决定了叶子节点必须是双向列表。

`FIL_PAGE_LSN`：该值代表该页最后被修改的日志序列位置LSN（Log Sequence Number）。

`FIL_PAGE_TYPE`：页的类型。通常有以下几种.请记住0x45BF，该值代表了存放的数据页。

 ![](https://images2015.cnblogs.com/blog/990532/201701/990532-20170116155831880-681271734.png)

![](https://images2015.cnblogs.com/blog/990532/201701/990532-20170116155844567-1546951191.png)

`FIL_PAGE_FILE_FLUSH_LSN`：该值仅在数据文件中的一个页中定义，代表文件至少被更新到了该LSN值。

`FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID`：从MySQL 4.1开始，该值代表页属于哪个表空间。

### Page Header

用来记录数据页的状态信息，由以下14个部分组成，共占用56个字节。见表4-5。 

![](https://images2015.cnblogs.com/blog/990532/201701/990532-20170116160008255-1274799455.png)

`PAGE_N_DIR_SLOTS`：在Page Directory（页目录）中的Slot（槽）数。Page Directory会在后面介绍。

`PAGE_HEAP_TOP`：堆中第一个记录的指针。

`PAGE_N_HEAP`：堆中的记录数。

`PAGE_FREE`：指向空闲列表的首指针。

`PAGE_GARBAGE`：已删除记录的字节数，即行记录结构中，delete flag为1的记录大小的总数。

`PAGE_LAST_INSERT`：最后插入记录的位置。

`PAGE_DIRECTION`：最后插入的方向。可能的取值为PAGE_LEFT（0x01），PAGE_RIGHT（0x02），PAGE_SAME_REC（0x03），PAGE_SAME_PAGE（0x04），PAGE_NO_DIRECTION（0x05）。

`PAGE_N_DIRECTION`：一个方向连续插入记录的数量。

`PAGE_N_RECS`：该页中记录的数量。

`PAGE_MAX_TRX_ID`：修改当前页的最大事务ID，注意该值仅在Secondary Index定义。

`PAGE_LEVEL`：当前页在索引树中的位置，0x00代表叶节点。

`PAGE_INDEX_ID`：当前页属于哪个索引ID。

`PAGE_BTR_SEG_LEAF`：B+树的叶节点中，文件段的首指针位置。注意该值仅在B+树的Root页中定义。

`PAGE_BTR_SEG_TOP`：B+树的非叶节点中，文件段的首指针位置。注意该值仅在B+树的Root页中定义。

 ### Infimum和Supremum记录

在InnoDB存储引擎中，每个数据页中有两个虚拟的行记录，用来限定记录的边界。Infimum记录是比该页中任何主键值都要小的值，Supremum指比任何可能大的值还要大的值。这两个值在页创建时被建立，并且在任何情况下不会被删除。在Compact行格式和Redundant行格式下，两者占用的字节数各不相同。下图显示了Infimum和Supremum Records。 

 ![](https://images2015.cnblogs.com/blog/990532/201701/990532-20170116160256067-537269791.png)

### User Records 与FreeSpace

User Records即实际存储行记录的内容。再次强调，InnoDB存储引擎表总是B+树索引组织的。

Free Space指的就是空闲空间，同样也是个链表数据结构。当一条记录被删除后，该空间会被加入空闲链表中。

 ### Page Directory

Page Directory（页目录）中**逆序**存放了**记录的相对位置**（与 页初始地址相比，偏移量就是与此时地址相比），记录就是每个槽的相对位置。有些时候这些记录指针称为**Slots**（槽）或者**目录槽**（**Directory Slots**）。与其他数据库系统不同的是，InnoDB并不是每个记录拥有一个槽，InnoDB存储引擎的槽是一个稀疏目录（sparse directory），即一个槽中可能属于（belong to）多个记录，**最少属于4条记录**，**最多属于8条记录**。

Slots中记录按照键顺序存放，这样可以利用二叉查找迅速找到记录的指针。假设我们有（'i'，'d'，'c'，'b'，'e'，'g'，'l'，'h'，'f'，'j'，'k'，'a'），同时假设一个槽中包含4条记录，则Slots中的记录可能是（'a'，'e'，'i'）。

由于InnoDB存储引擎中Slots是稀疏目录，二叉查找的结果只是一个粗略的结果，所以InnoDB必须通过`recorder header`中的`next_record`来继续查找相关记录。同时，slots很好地解释了`recorder header`中的`n_owned`值的含义，即还有多少记录需要查找，因为这些记录并不包括在slots中。

需要牢记的是，B+树索引本身并不能找到具体的一条记录，B+树索引能找到只是该记录所在的页。数据库把页载入内存，然后通过Page Directory再进行二分查找。只不过二叉查找的时间复杂度很低，同时内存中的查找很快，因此通常我们忽略了这部分查找所用的时间。

![](https://img-blog.csdn.net/20180116160102834?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbXByZW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

- 从上图可以看出`Page Directory`包含至少两个`infimum slot`,`supermum slot`，slot指向record(rec)指针(pointer to ‘A’),n_owned代表的是向前有多少个rec属于这个slot，中间被管辖的`rec`的`n_owned `= 0。 
- 通过directory的二分查找只能查到对应记录所属的slot，还需要通过slot内部的二分查找才能精确定位到对应的记录。这种设计的做法可以减小directory对page空间的占用，又能有很好查找的效率。



#### 如何对Page Directory 进行二分查找？

已知表中数据时按照主键的进行升序连续排列（逻辑上，在磁盘上不一定是连续的排列）。如上图，先到`Page Directory `中间的slots （记录的是相对页的偏移）找到对应的行的主键 ，然后判断后再找下一个slots。一个slots里有至少4条属性，故slots里也可以进行二分查找。