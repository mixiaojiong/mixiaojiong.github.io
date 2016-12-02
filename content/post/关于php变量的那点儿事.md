+++
title = "关于php变量的那点儿事"
date = "2015-12-18T22:22:46+08:00"
tags = ["php"]
author = "xiaojiong"

+++


PHP属于弱类型语言：一个变量可以表示任意的数据类型。

PHP之所以成为一个简单而强大的语言，很大一部分的原因是它拥有弱类型的变量。 但是有些时候这也是一把双刃剑，使用不当也会带来一些问题。就像仪器一样，越是功能强大， 出现错误的可能性也就越大。

在官方的PHP实现内部，所有变量使用同一种数据结构(zval)来保存，而这个结构同时表示PHP中的各种数据类型。 它不仅仅包含变量的值，也包含变量的类型。这就是PHP弱类型的核心。
### 一、PHP变量类型

 - 标量类型： boolean、integer、float(double)、string
 - 复合类型： array、object
 - 特殊类型： resource、NULL

### 二、PHP变量存储结构
前面提到变量的值存储在zvalue_value联合体中，结构体定义如下：
```
typedef union _zvalue_value {
    long lval;                  /* long value */
    double dval;                /* double value */
    struct {
        char *val;
        int len;
    } str;
    HashTable *ht;              /* hash table value */
    zend_object_value obj;
} zvalue_value;
```

> 使用联合体而不是用结构体是出于空间利用率的考虑，因为一个变量同时只能属于一种类型。 如果使用结构体的话将会不必要的浪费空间，而PHP中的所有逻辑都围绕变量来进行的，这样的话， 内存浪费将是十分大的。这种做法成本小但收益非常大。

 - 字符串string

```
struct {
    char *val;
    int len;
} str;
```
这里字符串使用的是一个结构体，额外存储了一个长度值，所以导致strlen()函数可以在常数时间内获取到字符串的长度。这种是典型的牺牲空间换时间的做法。

 - 对象
 （略）
 - 数组
 PHP中数组的实现是哈希表+链表的方式存储。参见：

	**[PHP中的哈希表](http://www.xiaojiong.net/2015/12/12/PHP%E4%B8%AD%E7%9A%84%E5%93%88%E5%B8%8C%E8%A1%A8%EF%BC%88HashTable%EF%BC%89/)**
	
	**[数据结构--双向链表](http://www.xiaojiong.net/2015/12/07/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%EF%BC%8D%EF%BC%8D%E5%8F%8C%E5%90%91%E9%93%BE%E8%A1%A8/)**
