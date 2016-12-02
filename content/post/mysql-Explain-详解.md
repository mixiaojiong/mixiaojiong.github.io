+++
title = "mysql Explain 详解"
date = "2016-03-12T04:51:22+08:00"
tags = ["mysql"]
author = "xiaojiong"

+++

[示例数据库下载地址](https://launchpad.net/test-db/employees-db-1/1.0.6/+download/employees_db-full-1.0.6.tar.bz2)

示例查询：

query1：

```
mysql> EXPLAIN SELECT * FROM employees\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: employees
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 299512
        Extra: NULL
1 row in set (0.00 sec)
```

query2:

```
mysql> EXPLAIN SELECT (SELECT emp_no FROM salaries LIMIT 1) FROM employees\G;
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: employees
         type: index
possible_keys: NULL
          key: first_name_last_name
      key_len: 94
          ref: NULL
         rows: 299512
        Extra: Using index
*************************** 2. row ***************************
           id: 2
  select_type: SUBQUERY
        table: salaries
         type: index
possible_keys: NULL
          key: emp_no
      key_len: 4
          ref: NULL
         rows: 2838426
        Extra: Using index
2 rows in set (0.00 sec)
```

query3:

```
mysql> EXPLAIN SELECT * FROM (SELECT emp_no FROM salaries) AS b\G;
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: <derived2>
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 2838426
        Extra: NULL
*************************** 2. row ***************************
           id: 2
  select_type: DERIVED
        table: salaries
         type: index
possible_keys: NULL
          key: emp_no
      key_len: 4
          ref: NULL
         rows: 2838426
        Extra: Using index
2 rows in set (0.01 sec)
```

query4:

```
mysql> EXPLAIN SELECT 1 UNION ALL SELECT 1\G;
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: NULL
         type: NULL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
        Extra: No tables used
*************************** 2. row ***************************
           id: 2
  select_type: UNION
        table: NULL
         type: NULL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
        Extra: No tables used
*************************** 3. row ***************************
           id: NULL
  select_type: UNION RESULT
        table: <union1,2>
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
        Extra: Using temporary
3 rows in set (0.00 sec)
```

### 一、id
标识select所属于的行

### 二、select_type

标识是简单查询还是复杂查询
简单(SIMPLE)：不包括子查询和UNION。如**query1**
复杂：最外层部分为PRIMARY,其余部分如下：

* SUBQUERY：包括在SELECT中，不在FROM子句中。如**query2**
* DERIVED：表示包含在FROM中的SELECT中。如**query3**
* UNION：UNION中的第二个和随后的SELECT，如果UNION中的第一个SELECT被FROM包含，那么第一个为DERIVED。如**query4**
* UNION RESULT：UNION匿名临时表检索结果的SELECT。如query4 ps：UNION总是将结果存放在匿名临时表中，之后MYSQL将结果读取到临时表外。临时表并不在原sql中出现，因此id为NULL。

### 三、table
显示访问哪个表，join的时候左侧深度优先树。

示例：

```
mysql> EXPLAIN SELECT emp_no FROM employees INNER JOIN dept_emp USING (emp_no) INNER JOIN departments USING (dept_no)\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: departments
         type: index
possible_keys: PRIMARY
          key: dept_name
      key_len: 122
          ref: NULL
         rows: 9
        Extra: Using index
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: dept_emp
         type: ref
possible_keys: PRIMARY,emp_no,dept_no
          key: dept_no
      key_len: 12
          ref: employees.departments.dept_no
         rows: 1
        Extra: Using index
*************************** 3. row ***************************
           id: 1
  select_type: SIMPLE
        table: employees
         type: eq_ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: employees.dept_emp.emp_no
         rows: 1
        Extra: Using index
3 rows in set (0.00 sec)
```

当在FROM子句中有子查询时，TABLE列是<derivedN>形式，其中N是子查询id。

```
mysql> EXPLAIN SELECT * FROM (SELECT emp_no FROM (SELECT * FROM employees ) AS a) AS b\G;
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: <derived2>
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 299512
        Extra: NULL
*************************** 2. row ***************************
           id: 2
  select_type: DERIVED
        table: <derived3>
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 299512
        Extra: NULL
*************************** 3. row ***************************
           id: 3
  select_type: DERIVED
        table: employees
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 299512
        Extra: NULL
3 rows in set (0.00 sec)
```
注意：MySQL对待这些表和普通表一样，但是这些“临时表”是没有任何索引的

### 四、type

关联类型（访问类型）

* ALL：全表扫描
* index：与全表扫描一样，只是Mysql扫描表时按索引次序进行而不是行。优点：避免了排序。缺点：按索引次序读取整个表的开销。Extra：Using index，说明Mysql正在使用覆盖索引，只扫描索引的数据，而不是索引次序的每一行。
* range：范围扫描是一个有限的索引扫描。例如：Between, where：>, IN, OR。
* ref：索引查找，非唯一索引和唯一索引前缀。ref_or_null是ref上的一个变体，它意味着MySQL必须在初次查找结果进行一次查找以找出NULL记录。
* eq_ref：是多返回一条符合条件的记录。
* const：where （主键=$num） 方式。
* NULL：优化阶段分解查询语句。

### 五、possible_key列

MySQL能使用哪些索引

### 六、key

mysql具体使用哪个索引

### 七、key_len

索引使用的字节数

k计算公式：

L：索引列所定义字段类型字符长度

C：不同编码下一个字符所占的字节数（如utf8=3，gbk=2）

N：字段为空标记，占1字节（非空字段此标记不占用字节）

S：索引列字段是否定长（int、char、datetime为定长，varchar为不定长），不定长字段类型需记录长度信息，占2字节

key_len = L\*C[+N][+S]

举例：

```
CREATE TABLE `dept_emp` (
  `emp_no` int(11) NOT NULL,
  `dept_no` char(4) NOT NULL,
  `from_date` date NOT NULL,
  `to_date` date NOT NULL,
  PRIMARY KEY (`emp_no`,`dept_no`),
  KEY `emp_no` (`emp_no`),
  KEY `dept_no` (`dept_no`),
  CONSTRAINT `dept_emp_ibfk_1` FOREIGN KEY (`emp_no`) REFERENCES `employees` (`emp_no`) ON DELETE CASCADE,
  CONSTRAINT `dept_emp_ibfk_2` FOREIGN KEY (`dept_no`) REFERENCES `departments` (`dept_no`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8

mysql> EXPLAIN SELECT * FROM dept_emp WHERE dept_no = 'd007'\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: dept_emp
         type: ref
possible_keys: dept_no
          key: dept_no
      key_len: 12
          ref: const
         rows: 93330
        Extra: Using index condition
1 row in set (0.00 sec)
```

计算方法：4\*3 = 12

```
CREATE TABLE `employees` (
  `emp_no` int(11) NOT NULL,
  `birth_date` date NOT NULL,
  `first_name` varchar(14) NOT NULL,
  `last_name` varchar(16) DEFAULT NULL,
  `gender` enum('M','F') NOT NULL,
  `hire_date` date NOT NULL,
  PRIMARY KEY (`emp_no`),
  KEY `first_name_last_name` (`first_name`,`last_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

mysql> EXPLAIN SELECT * FROM employees WHERE first_name = 'Tomofumi' AND last_name = 'Asmuth'\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: employees
         type: ref
possible_keys: first_name_last_name
          key: first_name_last_name
      key_len: 95
          ref: const,const
         rows: 1
        Extra: Using index condition
1 row in set (0.00 sec)
```

计算方法：(14+16)\*3 + 1 + 2\*2 = 95

### 八、ref
之前在key列记录的索引中查找值所用的列或常量

### 九、row
MySQL为了查找所需的行而要读取的行数。无法反应limit，不代表结果集。

### 十、Extra
额外信息

* Using index：使用覆盖索引。
* Using where：MySQL将存储引擎返回服务层以后再应用where条件过滤。
* Using temporary：对查询结果排序时会使用一个临时表。
* Using filesort：MySQL会对结果使用外部索引排序。
* Using Index condition：5.6新特性，可以进行索引筛选。[详情](http://www.xiaojiong.net/2016/02/18/mysql-Index-Condition-Pushdown/)
* Not exists：MySQL优化了left_join，一旦它找到了left join标准的行，就不再搜索了。

### 注意
mysql5.6以后支持非select语句。

### 参考资料
[高性能MYSQL(第三版)](http://www.oreilly.com.cn/index.php?func=book&isbn=978-7-121-19885-4)


