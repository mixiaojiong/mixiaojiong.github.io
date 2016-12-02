+++
title = "mysql的事务隔离级别"
date = "2016-03-30T05:10:36+08:00"
tags = ["mysql"]

+++

### 事务的特性（ACID）
* 原子性（atomicity）：一个事务（transaction）中的所有操作，要么全部完成，要么全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被回滚（Rollback）到事务开始前的状态，就像这个事务从来没有执行过一样。
* 一致性（consistency）：在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设规则，这包含资料的精确度、串联性以及后续数据库可以自发性地完成预定的工作。
* 隔离性（isolation）：当两个或者多个事务并发访问（此处访问指查询和修改的操作）数据库的同一数据时所表现出的相互关系。事务隔离分为不同级别，包括读未提交（Read uncommitted）、读提交（read committed）、可重复读（repeatable read）和串行化（Serializable）。
* 持久性（durability）：在事务完成以后，该事务对数据库所作的更改便持久地保存在数据库之中，并且是完全的。

### 事务的隔离级别（Isolation Level）
为解决脏读、幻读、不可重复读，引入如下4种事务隔离级别：

* 读未提交（Read Uncommitted）
* 读已提交（Read Committed）
* 可重复读（Repeatabel Read）
* 串行化读（Serializable）


| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| --- | --- | --- | --- |
| 读未提交（基本不用） | 允许 | 允许 | 允许 |
| 读已提交 | 不允许 | 允许 | 允许 |
| 可重复读（默认） | 不允许 | 不允许 | 不允许 |
| 串行化读（基本不用） | 不允许 | 不允许 | 不允许 |

* 脏读（dirty read）：一个事务读取到了另一个事务未提交的数据操作结果。
* 不可重复读（unrepeatable read）：事务T1读取某一数据，事务T2读取并修改了该数据，T1为了对读取值进行检验而再次读取该数据，便得到了不同的结果。
* 幻读（phantom read）：幻读是指当事务不是独立执行时发生的一种现象，例如第一个事务对一个表中的数据进行了修改，比如这种修改涉及到表中的“全部数据行”。同时，第二个事务也修改这个表中的数据，这种修改是向表中插入“一行新数据”。那么，以后就会发生操作第一个事务的用户发现表中还有没有修改的数据行，就好象发生了幻觉一样。

> update影响不可重复读;insert或delete影响幻读

### 注意
Repeatabel Read在其他数据库中并不能解决幻读问题，mysql/InnoDB通过Next-Key Lock机制实现。不过mysql并不能完全保证幻读，需要自己手动加锁实现。

举例：

```
mysql> show create table rr\G;
*************************** 1. row ***************************
       Table: rr
Create Table: CREATE TABLE `rr` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `value` varchar(30) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row in set (0.00 sec)
//查看当前隔离级别
mysql> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)
//开始实验
session1                                session2
begin;                                    begin;
select * from rr;
Empty set (0.00 sec)
                                        insert into rr values(1, 'haha');
select * from rr;
Empty set (0.00 sec)
                                        commit;
select * from rr;
Empty set (0.00 sec)

insert into rr values(1, 'haha');
ERROR 1062 (23000): Duplicate entry '1' for key 'PRIMARY'
```

幻读出现，MySQL InnoDB的可重复读并不保证避免幻读，需要应用使用加锁读来保证。而这个加锁度使用到的机制就是next-key locks。不过不建议使用加锁实现，会有性能问题。应该从应用层设计方面改善。
