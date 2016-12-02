+++
title = "vim小技巧"
date = "2015-06-28T19:09:55+08:00"
tags = ["vim"]
author = "xiaojiong"

+++


### 块操作
	1.如何在一段代码的每一行前插入a?
		<c-v>选中第一行的第一个字符，j->I->a>Esc
	2.如何在一段代码的每一行尾插入;?
		<c-v>选中第一行的最后一个字符，j->$->A->;->Esc

### 善于用.重复命令
	举例：xiaojiong is good man!
		现在光标在is中的i上，要删除is good,dw->.
		好处是如果只想删除is活着想多删一个单词，直接按u或者.就好了。
		
### 视图模式
	<html>aaa</html> 光标在><符号之间，如何选取><符号之间的内容。vit
	
### 去除windows ^M
    :%s/^M$//g 
    ^M 注意要用 Ctrl + V Ctrl + M 来输入

### 未完待续
