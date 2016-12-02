+++
title = "linux 常用命令"
date = "2015-06-28T19:04:48+08:00"
tags = ["linux"]
author = "xiaojiong"

+++


### 去除代码bom头
	查找：grep -r -I -l $'^\xEF\xBB\xBF' ./
	去除：vim set nobomb
	
### sed 替换文件内容
	sed -ig 's/a/x/g' test.txt
	test.txt 中字母a替换为x
	
### 解压
	.tar 
	解包：tar xvf FileName.tar
	打包：tar cvf FileName.tar DirName
    （注：tar是打包，不是压缩！）
    ———————————————
    .gz
    解压1：gunzip FileName.gz
    解压2：gzip -d FileName.gz
    压缩：gzip FileName

    .tar.gz 和 .tgz
    解压：tar zxvf FileName.tar.gz
    压缩：tar zcvf FileName.tar.gz DirName
    ———————————————
    .bz2
    解压1：bzip2 -d FileName.bz2
    解压2：bunzip2 FileName.bz2
    压缩： bzip2 -z FileName

    .tar.bz2
    解压：tar jxvf FileName.tar.bz2
    压缩：tar jcvf FileName.tar.bz2 DirName
	
### 未完待续
