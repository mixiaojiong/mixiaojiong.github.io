+++
title = "【转】MySQL的分页优化"
date = "2015-12-02T21:39:25+08:00"
tags = ["mysql"]
author = "xiaojiong"

+++


### [原文链接：http://www.amogoo.com/article/9](http://www.amogoo.com/article/9)
### 准备工作
1.  本文示例环境为MySQL5.6、InnoDB
2.  建表与数据初始化，数据来源于官网提供的employees库salaries表
```
CREATE TABLE `t_salaries` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `emp_no` int(11) NOT NULL,
  `salary` int(11) NOT NULL,
  `from_date` date NOT NULL,
  `to_date` date NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 分页语法
MySQL的分页语法非常简单：
```
select * from table where ? order by c asc|desc limit m, n;
```
表示在table中，根据where过滤并通过c列排序后的结果集，获取从m行开始往后的n条数据，其中不包括第m行，用区间可表示为[m+1, m+n]。

### 实验
那么如何来优化呢，废话不多说，下面直接上干货。
场景1（全表扫描带来的性能问题）：
```
-- 先看表数据
mysql> select count(*) from t_salaries;
+----------+
| count(*) |
+----------+
|  2844047 |
+----------+
1 row in set
  
-- sql1
mysql> select * from t_salaries order by emp_no asc limit 0,20;
......
20 rows in set
  
mysql> show profiles;
+----------+------------+---------------------------------------------------------+
| Query_ID | Duration   | Query                                                   |
+----------+------------+---------------------------------------------------------+
|        1 |  1.4158355 | select * from t_salaries order by emp_no asc limit 0,20 |
+----------+------------+---------------------------------------------------------+
1 rows in set
  
mysql> explain select * from t_salaries order by emp_no desc limit 0,20;
+----+-------------+------------+------+---------------+------+---------+------+---------+----------------+
| id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows    | Extra          |
+----+-------------+------------+------+---------------+------+---------+------+---------+----------------+
|  1 | SIMPLE      | t_salaries | ALL  | NULL          | NULL | NULL    | NULL | 2837536 | Using filesort |
+----+-------------+------------+------+---------------+------+---------+------+---------+----------------+
1 row in set
  
-- 添加order by列的索引
mysql> alter table `t_salaries` add index `idx_emp_no` (`emp_no`) using btree;
Query OK, 0 rows affected
Records: 0  Duplicates: 0  Warnings: 0
  
-- sql2
mysql> select * from t_salaries order by emp_no asc limit 0,20;
......
20 rows in set
  
mysql> show profiles;
+----------+------------+---------------------------------------------------------+
| Query_ID | Duration   | Query                                                   |
+----------+------------+---------------------------------------------------------+
|        1 | 0.00107775 | select * from t_salaries order by emp_no asc limit 0,20 |
+----------+------------+---------------------------------------------------------+
1 rows in set
  
mysql> explain select * from t_salaries order by emp_no asc limit 0,20;
+----+-------------+------------+-------+---------------+------------+---------+------+------+-------+
| id | select_type | table      | type  | possible_keys | key        | key_len | ref  | rows | Extra |
+----+-------------+------------+-------+---------------+------------+---------+------+------+-------+
|  1 | SIMPLE      | t_salaries | index | NULL          | idx_emp_no | 4       | NULL |   20 | NULL  |
+----+-------------+------------+-------+---------------+------------+---------+------+------+-------+
1 row in set
```
通过show profiles可以看到同一个sql，因为索引的缘故，性能上有1000多倍的差距，而通过对比两者的执行计划，我们发现sql1是一个全表扫描，所影响的行为所有行，而sql2是索引扫描，只影响20行，而且也没有了filesort的过程。

那么为什么这样就能优化呢？
解释很简单，随机给出100个数字，要找出其中最大的20个，怎么找？是不是至少需要读取一次所有数据，通过比较再得出最大的20个，这个比较的过程通常会是一个排序的过程。但要是给出的这100个数字已经是按从大到小顺序排列好的，那就简单了，只要读取前20个数字就好了，不需要读取剩下的。直接反应到数据中，就是减少了IO。
而通常实际开发中，需要排序的字段在表中存储往往是无序的（除非是InnoDB表中的主键字段），比如上述sql中的emp_no在表中就是无序的，而对该列创建的索引就是使他能够被有序的检索，以此来达到优化的目的。

场景2（limit偏移量过大带来的性能问题）：
依然是这个sql，不过这次是查询从200w开始往后的20条数据，来看执行结果。
```
-- sql3
mysql> select * from t_salaries order by emp_no asc limit 2000000,20;
......
20 rows in set
  
mysql> show profiles;
+----------+-----------+---------------------------------------------------------------+
| Query_ID | Duration  | Query                                                         |
+----------+-----------+---------------------------------------------------------------+
|        1 | 1.9852275 | select * from t_salaries order by emp_no asc limit 2000000,20 |
+----------+-----------+---------------------------------------------------------------+
1 rows in set
  
mysql> explain select * from t_salaries order by emp_no asc limit 2000000,20;
+----+-------------+------------+------+---------------+------+---------+------+---------+----------------+
| id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows    | Extra          |
+----+-------------+------------+------+---------------+------+---------+------+---------+----------------+
|  1 | SIMPLE      | t_salaries | ALL  | NULL          | NULL | NULL    | NULL | 2837536 | Using filesort |
+----+-------------+------------+------+---------------+------+---------+------+---------+----------------+
1 row in set
```
对比场景1中sql2，我们看到随着limit偏移量的增加，查询时间也会随之增加，通过执行计划，会发现受影响的行也会随之增加，直到全表扫描（上例sql的偏移量已超过走索引的阀值，所以直接是全表，可自行测试分别从100,1000....开始取值，观察执行计划的变化）。

此案例的优化方式有多种，但思路是一致的，就是减少IO，那么如何减少IO，下面介绍几中常用的优化方式。
方案1：
```
-- sql4
mysql> select t1.* from t_salaries t1 inner join (select id from t_salaries order by emp_no asc limit 2000000,20) t2 on t1.id = t2.id;
......
20 rows in set
  
mysql> show profiles;
+----------+-----------+--------------------------------------------------------------------------------------------------------------------------------+
| Query_ID | Duration  | Query                                                                                                                          |
+----------+-----------+--------------------------------------------------------------------------------------------------------------------------------+
|        1 | 0.3565585 | select t1.* from t_salaries t1 inner join (select id from t_salaries order by emp_no asc limit 2000000,20) t2 on t1.id = t2.id |
+----------+-----------+--------------------------------------------------------------------------------------------------------------------------------+
1 row in set
  
mysql> explain select t1.* from t_salaries t1 inner join (select id from t_salaries order by emp_no asc limit 2000000,20) t2 on t1.id = t2.id;
+----+-------------+------------+--------+---------------+------------+---------+-------+---------+-------------+
| id | select_type | table      | type   | possible_keys | key        | key_len | ref   | rows    | Extra       |
+----+-------------+------------+--------+---------------+------------+---------+-------+---------+-------------+
|  1 | PRIMARY     | <derived2> | ALL    | NULL          | NULL       | NULL    | NULL  | 2000020 | NULL        |
|  1 | PRIMARY     | t1         | eq_ref | PRIMARY       | PRIMARY    | 4       | t2.id |       1 | NULL        |
|  2 | DERIVED     | t_salaries | index  | NULL          | idx_emp_no | 4       | NULL  | 2837536 | Using index |
+----+-------------+------------+--------+---------------+------------+---------+-------+---------+-------------+
3 rows in set
```
通过执行计划可以看到，此方案虽然也要扫描2837536行，但走的是全索引扫描（只扫描索引块），相比较全表扫描，很大程度上减少了IO读取。再通过id的匹配，走了主键索引。因此对比sql3的全表扫描，有了5倍多的提速。
注意点，inner join中的子查询sql一定要走全索引扫描，即要将子查询sql中所有涉及的列（select列、where列、order by列）都要加到一个索引里，如果还要回表的话，就达不到优化的目的。百度上很多文章都提到此优化方案，但对于这一点，都是说的不清不楚，对初学者会造成很大的困扰。

方案2（基于主键排序的分页优化）：
```
-- sql5
mysql> explain select * from t_salaries where id >= (select id from t_salaries order by id asc limit 2000000,1) order by id asc limit 20;
+----+-------------+------------+-------+---------------+---------+---------+------+---------+-------------+
| id | select_type | table      | type  | possible_keys | key     | key_len | ref  | rows    | Extra       |
+----+-------------+------------+-------+---------------+---------+---------+------+---------+-------------+
|  1 | PRIMARY     | t_salaries | range | PRIMARY       | PRIMARY | 4       | NULL | 1418768 | Using where |
|  2 | SUBQUERY    | t_salaries | index | NULL          | PRIMARY | 4       | NULL | 2837536 | Using index |
+----+-------------+------------+-------+---------------+---------+---------+------+---------+-------------+
2 rows in set
```
此方案用于基于主键排序的分页优化，在实际中较少使用，写在这里是觉得有助于拓宽思路，依旧是通过主键索引扫描，减少IO来达到优化的目的。

总而言之，分页SQL优化的关键就是更好的利用索引，而利用索引的本质就是为了减少IO。
其实，分页另一种优化思路就是不在数据库分页，比如可以用一些全文检索的方案来解决，当然这不是本文要探讨的，但也确实是个思路。

