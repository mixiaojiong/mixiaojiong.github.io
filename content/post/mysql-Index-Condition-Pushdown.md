+++
title = "mysql Index Condition Pushdown"
date = "2016-02-18T14:36:47+08:00"
tags = ["mysql"]
author = "xiaojiong"

+++


从mysql5.6开始引入一个重要的优化，就是Index Condition Pushdown(ICP)
### 什么是ICP
Index Condition Pushdown(ICP)是MySQL用索引去表里取数据的一种优化。如果禁用ICP，引擎层会穿过索引在基表中寻找数据行，然后返回给MySQL Server层，再去为这些数据行进行WHERE后的条件的过滤。ICP启用，如果部分WHERE条件能使用索引中的字段，MySQL Server 会把这部分下推到引擎层。存储引擎通过使用索引条目，然后推索引条件进行评估，使用这个索引把满足的行从表中读取出。ICP能减少引擎层访问基表的次数和MySQL Server访问存储引擎的次数。总之是ICP的优化在引擎层就能够过滤掉大量的数据，这样无疑能够减少了对base table和mysql server的访问次数。
### ICP图解
没有ICP之前：
where条件->index索引->table filter（mysql_server）
![ICP之前](http://ww1.sinaimg.cn/mw690/68faff51gw1f13fxanouvj20df0camz5.jpg)
ICP之后：
where条件->index索引->index filter->table filter(mysql_server)
![ICP之后](http://ww1.sinaimg.cn/mw690/68faff51gw1f13fxanouvj20df0camz5.jpg)

### 数据库
[示例数据库下载地址](https://launchpad.net/test-db/employees-db-1/1.0.6/+download/employees_db-full-1.0.6.tar.bz2)
employees的数据库如下:
```
mysql> SHOW TABLES;
+---------------------+
| Tables_in_employees |
+---------------------+
| departments         |
| dept_emp            |
| dept_manager        |
| employees           |
| salaries            |
| titles              |
+---------------------+
6 rows in set (0.00 sec)
```
employees表结构如下:
```
mysql> show create table employees;
-------------------------------------------------------+
| employees | CREATE TABLE `employees` (
  `emp_no` int(11) NOT NULL,
  `birth_date` date NOT NULL,
  `first_name` varchar(14) NOT NULL,
  `last_name` varchar(16) NOT NULL,
  `gender` enum('M','F') NOT NULL,
  `hire_date` date NOT NULL,
  PRIMARY KEY (`emp_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)
```
### 建立联合索引
这个表默认只有一个主索引，因为ICP只能作用于二级索引，所以我们建立一个二级索引：
```
ALTER TABLE employees.employees ADD INDEX first_name_last_name (first_name, last_name);
```

### 查询
启用profiling并关闭query cache：
```
SET profiling = 1;
SET query_cache_type = 0;
SET GLOBAL query_cache_size = 0;
```
执行查询：
```
+----------+------------+-----------------------------------
| Query_ID | Duration   | Query  |
+----------+------------+-----------------------------------
|        8 | 0.00054575 | SELECT * FROM employees WHERE first_name='Mary' AND last_name LIKE '%man'              |
|        9 | 0.00046400 | SELECT * FROM employees WHERE first_name='Mary' AND last_name LIKE '%man'              |
|       10 | 0.00048650 | SELECT * FROM employees WHERE first_name='Mary' AND last_name LIKE '%man'              |
```
关闭ICP
```
SET optimizer_switch='index_condition_pushdown=off';
```
执行查询
```
+----------+------------+-----------------------------------
| Query_ID | Duration   | Query  |
+----------+------------+-----------------------------------
|       12 | 0.00159075 | SELECT * FROM employees WHERE first_name='Mary' AND last_name LIKE '%man'              |
|       13 | 0.00109525 | SELECT * FROM employees WHERE first_name='Mary' AND last_name LIKE '%man'              |
|       14 | 0.00130850 | SELECT * FROM employees WHERE first_name='Mary' AND last_name LIKE '%man'              |
+----------+------------+----------------------------------------------------------------------------------------+
```
通过以上的示例可以很清楚的发现开启和关闭ICP的区别，我们看一下explain

开启ICP
```
mysql> SET optimizer_switch='index_condition_pushdown=on';
Query OK, 0 rows affected (0.00 sec)

mysql> EXPLAIN SELECT * FROM employees WHERE first_name='Mary' AND last_name LIKE '%man'\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: employees
         type: ref
possible_keys: first_name_last_name
          key: first_name_last_name
      key_len: 58
          ref: const
         rows: 224
        Extra: Using index condition
1 row in set (0.00 sec)
```

关闭ICP
```
mysql> SET optimizer_switch='index_condition_pushdown=off';
Query OK, 0 rows affected (0.00 sec)

mysql> EXPLAIN SELECT * FROM employees WHERE first_name='Mary' AND last_name LIKE '%man'\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: employees
         type: ref
possible_keys: first_name_last_name
          key: first_name_last_name
      key_len: 58
          ref: const
         rows: 224
        Extra: Using where
1 row in set (0.00 sec)
```
注意Extra的区别，开启ICP是Using index condition，关闭ICP是Using where

### 注意

 - ICP只能用于二级索引，不能用于主索引。
 - 也不是全部where条件都可以用ICP筛选，如果某where条件的字段不在索引中，当然还是要读取整条记录做筛选，在这种情况下，仍然要到server端做where筛选。
 - ICP的加速效果取决于在存储引擎内通过ICP筛选掉的数据的比例。（筛选的数据越多，效果越明显）
 
### 参考资料

 1.[https://mariadb.com/kb/en/mariadb/index-condition-pushdown/](https://mariadb.com/kb/en/mariadb/index-condition-pushdown/)
 2.[http://dev.mysql.com/doc/refman/5.6/en/index-condition-pushdown-optimization.html](http://dev.mysql.com/doc/refman/5.6/en/index-condition-pushdown-optimization.html)
