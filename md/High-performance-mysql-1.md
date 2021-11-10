> 《高性能MySQL》笔记一：MySQL架构和历史

## 1. MySQL逻辑架构

**前文**：[关系型数据库是怎么工作的？](https://orzlinux.cn/blog/how-does-a-relational-database-work.html)

MySQL最重要、最与众不同的特性是它的存储引擎架构，这种架构的设计将查询处理及其他系统任务和数据的存储、提取相分离。这种处理和分离的设计可以在使用时根据性能、特性、以及其他需求来选择数据存储的方式。

MySQL逻辑架构：

![image-20211109170050136](https://gitee.com/hqinglau/img/raw/master/img/20211109170050.png)



> 触发器（trigger）：监视某种情况，并触发某种操作，它是提供给程序员和数据分析员来**保证数据完整性**的一种方法，它是与表事件相关的**特殊的存储过程**，它的执行不是由程序调用，也不是手工启动，而是由事件来触发，例如当对一个表进行操作（ insert，delete， update）时就会激活它执行。
>
> 触发器的创建创建四要素：
>
> - 监视地点（table）
>
> - 监视事件（update、insert、delete）
>
> - 触发时间（before、after）
>
> - 触发事件（update、insert、delete）
>
> 触发器语法：
>
> ```sql
> CREATE TRIGGER <触发器名> <监视时间> <监视事件>
> ON 监视地点 FOR EACH ROW 
> begin
>     触发事件
> end;
> ```

## 2. 并发控制

参看：[锁的实现和并发数据结构](https://orzlinux.cn/blog/concurrencyLock.html)

### 读写锁

常规概念。

### 锁粒度

在锁的开销和数据安全性之间寻求平衡。

表锁

行级锁：只在存储引擎层实现

## 3. 事务

要么全执行，要么全不执行。用`START TRANSACTION`语句开始一个事务，之后要么用`COMMIT`提交事务将修改数据持久保留，要么`ROLLBACK`撤销所有修改。

ACID：原子性（atomicity）、一致性（consistency）、隔离性（isolation）、持久性（durability）。

事务会增加额外的开销。对于一些需要事务的查询应用，可以选择一个非事务的存储引擎来获取更高性能。

### 隔离级别

SQL标准定义了四种隔离级别。

`READ UNCOMMITTED`：事务中的修改，就算未提交，其他事务也可以看见。事务读取未提交的数据：**脏读**。实际中很少应用。

`READ COMMITTED`：一个事务开始直到提交之前，所做的修改对其他事务不可见。有时也叫**不可重复读**，两次同样的查询，结果可能不一样。大多数数据库系统的默认隔离级别，但MySQL不是。

`REPEATABLE READ`：解决了脏读，在同一个事务中可以重复读，记录一致。无法解决幻读问题。**幻读**：当一个事务在读取某个范围内记录时，另一个事务又在这个范围插入了新的数据。InnoDB通过多版本并发控制解决幻读。MySQL默认事务隔离级别。

如图：张三查又查不到，插又不成功，幻读。

![image-20211110115842968](https://gitee.com/hqinglau/img/raw/master/img/20211110115843.png)

`SERIALIZABLE`：最高隔离级别，强制事务串行执行，避免了幻读问题。每一行数据都加锁。

总结：

![image-20211110120145944](https://gitee.com/hqinglau/img/raw/master/img/20211110120145.png)

### 死锁

如两个事务，一个要更新3然后4，另一个事务更新4然后3，二者都执行到中间等3或者4，但是被对方锁住了。

**解决方案：死锁检测和死锁超时机制。**InnoDB：将持有最少行级排他锁的事务回滚。

**死锁产生的原因**：数据冲突、存储引擎的实现方式。

### 事务日志

提高事务的效率。存储引擎修改表数据只需修改器内存拷贝，再把修改行为持久化到硬盘事务日志，后台慢慢把数据刷回磁盘。

事务日志追加方式，写到磁盘顺序区域，比更新数据库要快。

### MySQL事务

MySQL默认采用AUTOCOMMIT模式，每个查询默认当做一个事务执行提交操作。

```shell
mysql> SHOW VARIABLES LIKE 'AUTOCOMMIT';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+
1 row in set (1.16 sec)
```

查看隔离级别：

```shell
mysql> SELECT @@tx_isolation; # mysql8之前
mysql> SELECT @@transaction_isolation; # 之后
+-------------------------+
| @@transaction_isolation |
+-------------------------+
| REPEATABLE-READ         |
+-------------------------+
1 row in set (0.00 sec)
```

`@`是用户变量，`@@`是系统变量。

设置隔离级别：

```shell
mysql > SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

**事务中混合使用存储引擎**：

MySQL服务器层不管理事务，事务由下层存储引擎实现，在一个事务中，使用多种存储引擎是不可靠的。

混合使用时，非事务型表的数据无法回滚。

## 4. 多版本并发控制MVCC

可以认为MVCC是行级锁的变种，很多情况避免了加锁操作，开销更低。一般都实现了非阻塞读，写也只锁定必要行。

**InnoDB的MVCC**，在每行记录后面保存两个隐藏列，**行创建时间**和**行过期时间**对应的系统版本号（随新事务开始递增）。用于支持RC和RR隔离级别的实现。

如`REPEATABLE READ`情况：

**SELECT:**

只查找版本小于等于当前事务的数据行。行的删除版本要么未定义，要么比当前事务大。

**INSERT:**

插入行以当前版本号作为行版本号。

**DELETE：**

为每一行保存当前系统版本号作为行删除标识。

**UPDATE:**

插入新记录，删除旧记录。

### MVCC能否解决幻读问题？

查看自动提交事务是否开启：

```shell
mysql> show variables like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+
1 row in set (0.00 sec)
```

默认开启的，关闭。

```shell
mysql> set autocommit=0;
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | OFF   |
+---------------+-------+
1 row in set (0.01 sec)
```

创建一个测试用的表：

```shell
mysql> show create table person \G;
*************************** 1. row ***************************
       Table: person
Create Table: CREATE TABLE `person` (
  `id` int NOT NULL,
  `name` varchar(32) DEFAULT NULL,
  `age` int DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.02 sec)
```

准备一些基础数据：

```shell
mysql> select * from person;
+----+------+------+
| id | name | age  |
+----+------+------+
|  1 | a    |   18 |
|  2 | b    |   19 |
|  5 | e    |   19 |
+----+------+------+
3 rows in set (0.00 sec)
```

开启两个事务，左为事务A，右为事务B。

![image-20211110150439456](https://gitee.com/hqinglau/img/raw/master/img/20211110150439.png)

事务B中插入一条记录，但不提交，然后A和B都读一次：

> 红线代表代码运行的时间相对位置。

![image-20211110150750681](https://gitee.com/hqinglau/img/raw/master/img/20211110150750.png)

在RR隔离级别下，数据快照只在事务开始时创建，A执行的还是**快照读**，B执行了`insert`操作，为**当前读**。

尝试在事务A中更新一条记录：

![image-20211110151335171](https://gitee.com/hqinglau/img/raw/master/img/20211110151335.png)

当前读都是对本事务修改的记录，其他的仍然是快照读，不会出现幻读问题。

尝试在事务A中修改事务B新添加的那一条数据：

![image-20211110151655279](https://gitee.com/hqinglau/img/raw/master/img/20211110151655.png)

因为获取不到锁，B还没有提交，尝试提交事务B，然后在事务A中执行更新操作：

![image-20211110152003538](https://gitee.com/hqinglau/img/raw/master/img/20211110152003.png)

可以看到，本来是快照读，没有`id=3`的数据，执行更新操作后，出现了`id=3`的数据（变成了当前读）。

> MySQL 里除了普通查询是快照度，其他都是**当前读**，比如update、insert、delete，这些语句执行前都会查询最新版本的数据，然后再做进一步的操作。假设要 update 一个记录，另一个事务已经 delete 这条记录并且提交事务了，这样就会产生冲突，所以 update 的时候肯定要知道最新的数据。
>
> 另外，`select ... for update` 这种查询语句是当前读，每次执行的时候都是读取最新的数据。

解决方案，间隙锁，见下文：

![image-20211110153525579](https://gitee.com/hqinglau/img/raw/master/img/20211110153525.png)



## 5. MySQL存储引擎

MySQL为每个数据库保存为数据目录下的子目录，表的定义放在数据库子目录下：表名.frm的文件中。不同的存储引擎保存数据和索引的方式不同，但是表的定义是在服务层统一处理的。

查询表相关信息：

```shell
mysql> SHOW TABLE STATUS LIKE 'user' \G;
*************************** 1. row ***************************
           Name: user
         Engine: InnoDB
        Version: 10
     Row_format: Dynamic
           Rows: 15
 Avg_row_length: 1092
    Data_length: 16384  # 表数据的大小，字节
Max_data_length: 0    # 表数据最大容量，与存储引擎油管
   Index_length: 32768  # 索引大小，字节单位
      Data_free: 0
 Auto_increment: 19
    Create_time: 2021-10-22 19:37:14
    Update_time: 2021-11-06 19:39:07
     Check_time: NULL
      Collation: utf8mb4_0900_ai_ci  # 默认字符集和字符列排序规则
       Checksum: NULL # 启用的话为实时校验和
 Create_options: 
        Comment: 
1 row in set (0.00 sec)
```

查询表的字段：

```shell
mysql> desc user;
+-----------+--------------+------+-----+---------+----------------+
| Field     | Type         | Null | Key | Default | Extra          |
+-----------+--------------+------+-----+---------+----------------+
| id        | int          | NO   | PRI | NULL    | auto_increment |
| id_name   | varchar(128) | NO   | UNI | NULL    |                |
| nick_name | varchar(128) | NO   | MUL | NULL    |                |
| password  | varchar(32)  | NO   |     | NULL    |                |
+-----------+--------------+------+-----+---------+----------------+
4 rows in set (0.17 sec)

mysql> SHOW CREATE TABLE user \G;
*************************** 1. row ***************************
       Table: user
Create Table: CREATE TABLE `user` (
  `id` int NOT NULL AUTO_INCREMENT,
  `id_name` varchar(128) NOT NULL,
  `nick_name` varchar(128) NOT NULL,
  `password` varchar(32) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `id_name` (`id_name`),
  KEY `nick_name` (`nick_name`)
) ENGINE=InnoDB AUTO_INCREMENT=19 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)
```

> 在MySQL的sql语句后加上 \G ，表示将查询结果进行按列打印，可以使每个字段打印到单独的行。 即将查到的结构旋转90度变成纵向。

### InnoDB存储引擎

InnoDB数据存储在表空间中，由InnoDB管理的一系列数据文件组成。在MySQL 4.1之后，可以将每个表的数据和索引存放在单独文件中。

MVCC通过行锁和间隙锁共同组成的（next-key locking）策略防止幻读的出现。不仅锁定查询涉及的行，还有索引中的间隙锁定，防止幻影行的插入。

**快照读因为有MVCC，所以不会产生幻读；有了next-key lock，当前读不会产生幻读**

间隙锁的原则：

- 加锁的基本单位是（next-key lock）,他是前开后闭原则
- 查询过程中访问的对象会增加锁
- 索引上的等值查询，给唯一索引加锁的时候，next-key lock升级为行锁
- 索引上的等值查询，向右遍历时最后一个值不满足查询需求时，next-key lock 退化为间隙锁
- 唯一索引上的范围查询会访问到不满足条件的第一个值为止

例子：

```shell
mysql> desc test;
+-------+------+------+-----+---------+-------+
| Field | Type | Null | Key | Default | Extra |
+-------+------+------+-----+---------+-------+
| id    | int  | NO   | PRI | NULL    |       | # 主键
| a     | int  | NO   | UNI | NULL    |       | # 唯一索引
| b     | int  | NO   | MUL | NULL    |       | # 普通索引
| c     | int  | NO   |     | NULL    |       | # 没有索引
+-------+------+------+-----+---------+-------+
4 rows in set (0.01 sec)

mysql> select * from test;
+----+----+----+----+
| id | a  | b  | c  |
+----+----+----+----+
|  0 |  0 |  0 |  0 |
|  5 |  5 |  5 |  5 |
| 10 | 10 | 10 | 10 |
| 20 | 20 | 20 | 20 |
+----+----+----+----+
4 rows in set (0.00 sec)
```

幻读是指一个事务前后两次查询同一范围的时候，后一次查询看到了前一次查询没有看到的行记录。

产生幻读的原因是，行锁只能锁住行，但是新插入记录的这个动作，要更新的是记录之间的“间隙”。因此，为了解决幻读的问题，InnoDB在RR隔离级别引入了新的锁，它就是间隙锁（Gap Lock）。

**RR无索引情况**：

![image-20211110174604516](https://gitee.com/hqinglau/img/raw/master/img/20211110174604.png)

RR级别下，无索引的条件字段的当前读会把每条记录都加上排他锁，还有间隙锁。所以，当前读或者插入、更新、删除操作要加上索引。

**RR级别普通索引**：

```shell
select * from test where b = 10 for update;
```

b=10的记录加行锁。上下两条间隙加间隙锁。锁住的应当为（5,20）。这个范围内的操作会阻塞。

**RR唯一索引**：只能查出一条记录，加行锁就行了。

![image-20211110175205817](https://gitee.com/hqinglau/img/raw/master/img/20211110175205.png)

主键情况：

![image-20211110175438722](https://gitee.com/hqinglau/img/raw/master/img/20211110175438.png)

间隙锁之间不是互斥的，会有死锁情况。

> lock in share mode共享锁，for update排他锁。

### MyISAM存储引擎

是MySQL 5.1之前的默认存储引擎，不支持事务和行级锁。对整张表加锁，读时共享锁，写时排他锁。可以手工或自动执行检查和修复操作（不同于事务恢复、崩溃恢复），可能导致数据丢失，慢。对于文本等长字段，可以基于前500个字符创建索引，也支持全文索引，基于分词，可以支持复杂的查询。可以延迟更新索引键，写入性能提升，数据库崩溃需要执行修复操作。

如果表创建并导入数据之后，不会再修改，可能适合MyISAM压缩表，较少磁盘IO。





## 参考

高性能MySQL第三版

[MYSQL（04）-间隙锁详解](https://www.jianshu.com/p/32904ee07e56)

[MySQL：图解MVCC到底能不能解决幻读问题？](https://blog.csdn.net/saintmm/article/details/120784426)

[MySQL的幻读是怎么被解决的？](https://www.cnblogs.com/xiaolincoding/p/15308381.html)

[MYSQL（04）-间隙锁详解](https://www.jianshu.com/p/32904ee07e56)

[MySQL的间隙锁](https://zhuanlan.zhihu.com/p/356824126)