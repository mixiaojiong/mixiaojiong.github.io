+++
title = "phpredis扩展修复记"
date = "2016-01-13T22:08:17+08:00"
tags = ["php"]
author = "xiaojiong"

+++


最近由于项目需要，我在用php+redis+lua开发项目的时候发现一个bug

```
<?php
    $redis = new Redis();
    $redis->connect('127.0.0.1', 6379);
    $res = $redis->eval('return 32140378*16777216');
    var_dump($res);
    实际的结果：1509949440
    正确的结果：539226064027648
```

### 定位问题
首先在定位问题在哪？我首先用redis在shell里直接执行lua脚本，发现结果正确。排除redis与lua兼容性的问题，那么问题可以确定是phpredis扩展的问题。于是我从pecl上下载最新的2.2.7版本，重新安装扩展，结果还是如此。

### 解决问题

 1. 首先想到的就是搜索引擎，在尝试各种关键字+各种搜索引擎无果。
 2. 去stackoverflow看看老外有什么好办法没。还是不行~
 3. 去github上提交问题 [issues-718](https://github.com/phpredis/phpredis/issues/718)。还是不行~
 4. 只好自己看phpredis扩展源码解决了，发现library.c 中 replay_info 为int去掉符号位，也就是说只要eval返回值大于2^31-1就会溢出~修改为long int，发现bugfix。

### github
为了以后其他人别踩这个坑，也为了开源事业做贡献，提交我的修复到：[pull-724](https://github.com/phpredis/phpredis/pull/724)。
我第一次在github上提交版本过程中特别感谢[michael-grunder](https://github.com/michael-grunder)对我的帮助。

### 鸣谢
[PHP扩展开发及内核应用](https://github.com/walu/phpbook)
[鸟哥的博客](http://www.laruence.com/)
