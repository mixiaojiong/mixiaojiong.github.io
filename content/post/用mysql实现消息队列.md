+++
title = "用mysql实现消息队列"
date = "2016-01-04T21:04:15+08:00"
tags = ["mysql"]
author = "xiaojiong"

+++


在复杂的系统中，为了解耦业务之间的耦合关系，往往需要消息队列来实现模块之间的异步通信。消息队列要满足以下要求：

 - 消息不能丢失，最好能只被处理一次
 - FIFO顺序
 - 支持多生产者
 - 支持多消费者，每一个消息只能被一个消费者处理。
 - 支持并行消费

### 表结构
```
CREATE TABLE `queue` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `biz` int(11) DEFAULT NULL COMMENT '业务号',
  `conn_id` int(11) unsigned DEFAULT '0' COMMENT '连接ID',
  `status` enum('y','run','n') DEFAULT 'n' COMMENT '状态（n=>未处理，run=>处理中，y=>处理完毕）',
  `data_string` text COMMENT '数据载体',
  `times` int(10) DEFAULT NULL COMMENT '入库时间',
  `update_times` int(10) DEFAULT NULL COMMENT '状态更新时间',
  PRIMARY KEY (`id`),
  KEY `biz` (`biz`,`conn_id`,`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 实现方案
为了支持FIFO顺序，可以用MySql的自增ID来排序，先进入的消息ID要小些。MySql数据库是支持并发操作的，这就自动支持了多生产者的情况。为了支持多消费者并且每个消息只被一个消费者处理，可以为每个消费者分配一个ID，当某个消息正在被消费者处理时，这个消费者就被认为是这个消息的所有者。

关键在于怎么支持并发，怎么设置所有者？

#### 一、入队列
```
INSERT INTO `queue` (`biz`, `data_string`, `times`) VALUES ('1000', 'test', '1451889723')
```
#### 二、出队列
1.获取未被处理的消息，首先先占用这些消息，成为这些消息的所有者。
```
UPDATE `queue` SET `status` = 'run', `update_times` = 1451879201, `conn_id` = CONNECTION_ID() WHERE `conn_id` = 0 AND `status` = 'n' LIMIT x
```

根据mysql_affected_rows()的返回结果可以判断是否有未被处理的消息。若有，则取出消息：

```
SELECT `id`, `biz`, `status`, `times`, `data_string` FROM `athena_queue` WHERE `biz` = 1000 AND `conn_id` = CONNECTION_ID() AND `status` = 'run' 
```

2.处理消息，此时表不会被锁住，其它消费者也可以获取消息；
3.处理完毕后从数据库中更新
```
UPDATE `queue` SET `status` = 'y', `update_times` = 1451879201 WHERE `id` = xxx, `conn_id` = xxx AND `status` = 'run' 
```
如果在1和2之间有一个消费者进程被kill了，那么该进程获取的消息将永远无法被处理了。
#### 三、失败的处理

方案一：启动一个监护进程，定期去检查一下是否有超过一定时间限制，并且不在当前连接列表中的消息，并重置他们。可以使用:
```
SHOW PROCESSLIST 可以获取当前所有的连接ID
UPDATE `queue` SET `status` = 'n', `update_times` = 1451879201, `conn_id` = 0 WHERE `conn_id` NOT IN (xxx) AND `status` = 'run' AND `update_times` < xxx
```
方案二：将出库的第1步改为：
```
UPDATE `queue` SET `status` = 'n', `update_times` = 1451879201, `conn_id` = 0 WHERE (`conn_id` = 0 AND `status` = 'n' ) OR (`conn_id` NOT IN (xxx) AND `status` = 'run' AND `update_times` < xxx) limit xxx;
```

这样即使消费者A在第1步和第2步之间失败了，消费者B还可以重新取出该消息重新处理。所以只要还有消费者在，消息至少会被处理一次。

#### 四、如何实现只处理一次呢？
如果消费者在第2步和第3步之间失败的话，这个消息就会被再次取出来处理一次，这样就一共处理了2次。要保证每个消息只被处理一次，也就是要保证第2步和第3步是一个原子操作，要么都做，要么都不做。在第2步和第3步开启事务。

参考资料
[高性能MYSQL(第三版)](http://www.oreilly.com.cn/index.php?func=book&isbn=978-7-121-19885-4)
