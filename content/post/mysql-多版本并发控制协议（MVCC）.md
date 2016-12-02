+++
title = "mysql 多版本并发控制协议（MVCC）"
date = "2016-04-13T05:16:55+08:00"
tags = ["mysql"]
author = "xiaojiong"
+++

### 一、什么是MVCC
Multi-Version Concurrency Control 多版本并发控制

### 二、MVCC有什么用？
在关系型数据库中行锁与行的多个版本结合起来，只需要很小的开销,就可以实现非锁定读，从而大大提高数据库系统的并发性能

### 三、书上说
高性能mysql第三版：innodb 中每一行有两个隐藏字段。分别为创建版本号和删除版本号，一般用事务ID来进行标识。

* select: 读取创建版本号 <= 当前事务版本号，删除版本号为空或 > 当前事务版本号
* insert: 保存当前事务版本号作为创建版本号
* delete: 保存当前事务版本号作为删除版本号
* update: 插入一条新纪录，保存当前事务版本号作为创建版本号；同时保存当前事务版本号为修改前记录的删除版本号。

### 四、深入些
每一行的隐藏字段不是两个而是四个：DATA_TRX_ID，DATA_ROLL_PTR，DB_ROW_ID，DELETE BIT。

* 6字节的DATA_TRX_ID 标记了最新更新这条行记录的transaction id，每处理一个事务，其值自动+1
* 7字节的DATA_ROLL_PTR 指向当前记录项的rollback segment的undo log记录，找之前版本的数据就是通过这个指针
* 6字节的DB_ROW_ID，当由innodb自动产生聚集索引时，聚集索引包括这个DB_ROW_ID的值，否则聚集索引中不包括这个值.，这个用于索引当中
* DELETE BIT位用于标识该记录是否被删除，这里的不是真正的删除数据，而是标志出来的删除。真正意义的删除是在commit的时候

在一个sql进行查询时，读取到一行数据的DB_TRX_ID值和自己事物ID的对比，假如隔离级别为MySQL的默认级别，就只读取该ID值小于本身事物ID的数据，其余数据就需要通过DB_ROLL_PTR的信息到回滚段中读取。MVCC是否起到相应的作用需取决于数据库隔离级别的配置。

在insert和update、delete的操作是有区别的，一个insert语句插入数据再rollback就是直接对undo log的删除，因为他并不会影响其他事物的读取操作，而update、delete操作是在原有数据做更改，可能有其他事物在对该行数据做读取操作，所以update、delete产生的undo log数据是由内部线程自动清理，在该数据无任何事务在使用时清理掉，所以在undo log中insert和update、delete产生的数据存于不同位置。

上面说了数据的update、delete、insert操作，都会根据主键上的隐藏列来判断和查找，但是二级索引并不存在隐藏列，二级索引就是有索引列和主键列组成的一个小表，这该怎么判断呢？二级索引区别在于假如是一个update操作步骤为：

1. 标记删除原纪录
2. 插入新纪录
3. 对应主键做上面的隐藏字段修改，行数据更新，原行数据移入回滚段

一个查询语句在利用二级索引进行查找时，发现有个标记删除或者有新数据就会到主键扫描对应的DB_TRX_ID值对比当前事物ID大小，是否利用DB_ROLL_PTR进行读取数据。二级索引的delete、insert也是类似，只是一个没有新纪录，一个没有标记删除记录。

### 五、问题

既然知道mysql的每一行中存在记录的版本号，那可不可以通过版本号查询mysql某一行的历史记录呢？

答案是no,不过oracle有flashback可以实现。我的理解是oracle的undo log做的更出色一些吧。
