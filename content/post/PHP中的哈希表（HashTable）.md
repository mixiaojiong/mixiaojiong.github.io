+++
title = "PHP中的哈希表（HashTable）"
date = "2015-12-12T22:51:41+08:00"
tags = ["php"]
author = "xiaojiong"
+++


PHP中使用最为频繁的数据类型非字符串和数组莫属，PHP比较容易上手也得益于非常灵活的数组类型。 在开始详细介绍这些数据类型之前有必要介绍一下哈希表(HashTable)。 哈希表是PHP实现中尤为关键的数据结构。

哈希表在实践中使用的非常广泛，例如编译器通常会维护的一个符号表来保存标记，很多高级语言中也显式的支持哈希表。 哈希表通常提供查找(Search)，插入(Insert)，删除(Delete)等操作，这些操作在最坏的情况下和链表的性能一样为O(n)。 不过通常并不会这么坏，合理设计的哈希算法能有效的避免这类情况，通常哈希表的这些操作时间复杂度为O(1)。 这也是它被钟爱的原因。

### 基本概念

 - 键(key)：用于操作数据的标示，例如PHP数组中的索引，或者字符串键等等。
 - 槽(slot/bucket)：哈希表中用于保存数据的一个单元，也就是数据真正存放的容器。
 - 哈希函数(hash function)：将key映射(map)到数据应该存放的slot所在位置的函数。
 - 哈希冲突(hash collision)：哈希函数将两个不同的key映射到同一个索引的情况。

 哈希表可以理解为数组的扩展或者关联数组，数组使用数字下标来寻址，如果关键字(key)的范围较小且是数字的话， 我们可以直接使用数组来完成哈希表，而如果关键字范围太大，如果直接使用数组我们需要为所有可能的key申请空间。 很多情况下这是不现实的。即使空间足够，空间利用率也会很低，这并不理想。同时键也可能并不是数字， 在PHP中尤为如此，所以人们使用一种映射函数(哈希函数)来将key映射到特定的域中：

```
h(key) -> index
```

通过合理设计的哈希函数，我们就能将key映射到合适的范围，因为我们的key空间可以很大(例如字符串key)， 在映射到一个较小的空间中时可能会出现两个不同的key映射被到同一个index上的情况， 这就是我们所说的出现了冲突。 目前解决hash冲突的方法主要有两种：链接法和开放寻址法。

### 解决冲突

 - 链接法（PHP）
链接法通过使用一个链表来保存slot值的方式来解决冲突，也就是当不同的key映射到一个槽中的时候使用链表来保存这些值。 所以使用链接法是在最坏的情况下，也就是所有的key都映射到同一个槽中了，这样哈希表就退化成了一个链表， 这样的话操作链表的时间复杂度则成了O(n)，这样哈希表的性能优势就没有了， 所以选择一个合适的哈希函数是最为关键的。

 - 开放寻址法
使用开放寻址法是槽本身直接存放数据， 在插入数据时如果key所映射到的索引已经有数据了，这说明发生了冲突，这是会寻找下一个槽， 如果该槽也被占用了则继续寻找下一个槽，直到寻找到没有被占用的槽，在查找时也使用同样的策略来进行。

### 哈希表的实现

 1. 实现哈希函数
 2. 冲突的解决
 3. 操作接口的实现
 
### PHP中的应用
变量的作用域、函数表、类的属性、方法等，Zend引擎内部的很多数据都是保存在哈希表中的。

 - 哈希表结构
 
```
 typedef struct _hashtable { 
    uint nTableSize;        // hash Bucket的大小，最小为8，以2x增长。
    uint nTableMask;        // nTableSize-1 ， 索引取值的优化
    uint nNumOfElements;    // hash Bucket中当前存在的元素个数，count()函数会直接返回此值 
    ulong nNextFreeElement; // 下一个数字索引的位置
    Bucket *pInternalPointer;   // 当前遍历的指针（foreach比for快的原因之一）
    Bucket *pListHead;          // 存储数组头元素指针
    Bucket *pListTail;          // 存储数组尾元素指针
    Bucket **arBuckets;         // 存储hash数组
    dtor_func_t pDestructor;    // 在删除元素时执行的回调函数，用于资源的释放
    zend_bool persistent;       //指出了Bucket内存分配的方式。如果persisient为TRUE，则使用操作系统本身的内存分配函数为Bucket分配内存，否则使用PHP的内存分配函数。
    unsigned char nApplyCount; // 标记当前hash Bucket被递归访问的次数（防止多次递归）
    zend_bool bApplyProtection;// 标记当前hash桶允许不允许多次访问，不允许时，最多只能递归3次
#if ZEND_DEBUG
    int inconsistent;
#endif
} HashTable;
```
 - 数据容器：槽位

```
typedef struct bucket {
    ulong h;            // 对char *key进行hash后的值，或者是用户指定的数字索引值
    uint nKeyLength;    // hash关键字的长度，如果数组索引为数字，此值为0
    void *pData;        // 指向value，一般是用户数据的副本，如果是指针数据，则指向pDataPtr
    void *pDataPtr;     //如果是指针数据，此值会指向真正的value，同时上面pData会指向此值
    struct bucket *pListNext;   // 整个hash表的下一元素
    struct bucket *pListLast;   // 整个哈希表该元素的上一个元素
    struct bucket *pNext;       // 存放在同一个hash Bucket内的下一个元素
    struct bucket *pLast;       // 同一个哈希bucket的上一个元素
    // 保存当前值所对于的key字符串，这个字段只能定义在最后，实现变长结构体
    char arKey[1];              
} Bucket;
```

### Zend引擎哈希表结构和关系
![tool-editor](http://ww1.sinaimg.cn/large/68faff51jw1eyx9mnolj5j20k00dhmz3.jpg)

 - Bucket结构体维护了两个双向链表，pNext和pLast指针分别指向本槽位所在的链表的关系。
 - 而pListNext和pListLast指针指向的则是整个哈希表所有的数据之间的链接关系。 HashTable结构体中的pListHead和pListTail则维护整个哈希表的头元素指针和最后一个元素的指针。

> PHP中数组的操作函数非常多，例如：array_shift()和array_pop()函数，分别从数组的头部和尾部弹出元素。哈希表中保存了头部和尾部指针，这样在执行这些操作时就能在常数时间内找到目标。PHP中还有一些使用的相对不那么多的数组操作函数：next()，prev()等的循环中，哈希表的另外一个指针就能发挥作用了：pInternalPointer，这个用于保存当前哈希表内部的指针。 这在循环时就非常有用。

如图中左下角的假设，假设依次插入了Bucket1，Bucket2，Bucket3三个元素：

 1. 插入Bucket1时，哈希表为空，经过哈希后定位到索引为1的槽位。此时的1槽位只有一个元素Bucket1。其中Bucket1的pData或者pDataPtr指向的是Bucket1所存储的数据。此时由于没有链接关系。pNext，pLast，pListNext，pListLast指针均为空。同时在HashTable结构体中也保存了整个哈希表的第一个元素指针，和最后一个元素指针，此时HashTable的pListHead和pListTail指针均指向Bucket1。
 2. 插入Bucket2时，由于Bucket2的key和Bucket1的key出现冲突，此时将Bucket2放在双链表的前面。由于Bucket2后插入并置于链表的前端，此时Bucket2.pNext指向Bucket1，由于Bucket2后插入。Bucket1.pListNext指向Bucket2，这时Bucket2就是哈希表的最后一个元素，这是HashTable.pListTail指向Bucket2。
 3. 插入Bucket3，该key没有哈希到槽位1，这时Bucket2.pListNext指向Bucket3，因为Bucket3后插入。同时HashTable.pListTail改为指向Bucket3。

 

