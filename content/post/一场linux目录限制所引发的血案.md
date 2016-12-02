+++
title = "一场linux目录限制所引发的血案"
date = "2015-10-14T20:18:45+08:00"
tags = ["linux"]
author = "xiaojiong"

+++

我们公司最近出现一个BUG，没有上传过用户头像的用户，无法上传头像。而已经上传过头像的用户，可以更新。
简单介绍一下我们这块的业务：
我们上传头像采取的文件目录存储结构为/data/personal/968/avater/a.jpg其中968为uid。
### 1.定位问题
查找日志发现是因为无法在/data/personal/下创建新目录。
### 2.解决问题
根据业务可以对uid进行hash。也就是目录了变为/data/personal/hash/968/avater/a.jpg
### 3.为什么会无法创建目录
当无法在/data/personal/下创建新目录时，我查看了一下当时/data/personal/下一共有31998个目录。为什么是31998呢？开始google之。发现我们的服务器采用的是ext3文件格式。Linux为了cpu的搜索效率而规定的,要想改变数目限制需要重新编译内核。我看到在kernel代码中有这样的：

```
include/linux/ext2_fs.h:#define EXT2_LINK_MAX           32000
include/linux/ext3_fs.h:#define EXT3_LINK_MAX           32000
```

ext3文件系统一级子目录的个数默认为31998(个)，准确地说是32000个。为什么说31998个呢？这是因为mkdir创建一个目录时，目录下默认就会创建两个子目录的，一个是.目录（代表当前目录），另一个是..目录（代表上级目录）。这两个子目录是删除不掉的，“ rm . ” 会得到“rm: cannot remove `.' or `..'”的提示。所以32000-2=31998。
