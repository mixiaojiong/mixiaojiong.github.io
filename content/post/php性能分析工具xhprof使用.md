+++
title = "php性能分析工具xhprof使用"
date = "2015-07-15T20:01:51+08:00"
tags = ["php"]
author = "xiaojiong"

+++


### 简介
XHProf 是一个轻量级的分层性能测量分析器。 在数据收集阶段，它跟踪调用次数与测量数据，展示程序动态调用的弧线图。 它在报告、后期处理阶段计算了独占的性能度量，例如运行经过的时间、CPU 计算时间和内存开销。 函数性能报告可以由调用者和被调用者终止。 在数据搜集阶段 XHProf 通过调用图的循环来检测递归函数，通过赋予唯一的深度名称来避免递归调用的循环。XHProf 包含了一个基于 HTML 的简单用户界面(由 PHP 写成)。 基于浏览器的用户界面使得浏览、分享性能数据结果更加简单方便。 同时也支持查看调用图。XHProf的报告对理解代码执行结构常常很有帮助。 比如此分层报告可用于确定在哪个调用链里调用了某个函数。XHProf 对两次运行进行比较（又名 "diff" 报告），或者多次运行数据的合计。对比、合并报告，很像针对单次运行的“平式视图”性能报告，就像“分层式视图”的性能报告。更多额外文档可以在 » facebook xhprof 上找到。

### 安装配置

```
$ wget https://github.com/facebook/xhprof/tarball/master -O xhprof.tar.gz
$ tar zxf xhprof.tar.gz
$ cd xhprof/extension/
$ phpize
$ ./configure --with-php-config=`/path/to/php-config`
$ make && make install
$ make test
#安装Graphviz（点击[View Full Callgraph]有图片展示）
$ wget http://www.graphviz.org/pub/graphviz/stable/SOURCES/graphviz-2.24.0.tar.gz
$ tar zxf graphviz-2.24.0.tar.gz
$ cd graphviz-2.24.0
$ ./configure
$ make
$ make install 

$ vim /etc/php.ini
; 内容为：
extension = xhprof.so
; 注意：output_dir 必须存在且可写
xhprof.output_dir = /tmp/xhpro

$ mkdir -p /tmp/xhpro/
$ chmod -R 777 /tmp/xhpro/
重启php-fpm 或者apache
```
### 使用
```
<?php
xhprof_enable(XHPROF_FLAGS_CPU+XHPROF_FLAGS_MEMORY);//加上这个参数可以使得xhprof显示cpu和内存相关的数据。

foo();//把要测量的函数用xhprof_enable和xhprof_disable包围起来。

$data = xhprof_disable();

//得到统计数据之后，以下的工作就是为页面显示做准备。
$xhprof_root = "/home/www/xhprof";//这里填写的就是你的xhprof的路径
//
include_once $xhprof_root."/xhprof_lib/utils/xhprof_lib.php";
include_once $xhprof_root."/xhprof_lib/utils/xhprof_runs.php";
//
$xhprof_runs = new XHprofRuns_Default();
$run_id = $xhprof_runs->save_run($data, "test");//第二个参数在接下来的地方作为命名空间一样的概念来使用
//
///**************************
//  访问<xhprof-ui-address>/index.php?run=$run_id&source=test就能够看到一个统计列表了。
//    
//      1. <xhprof-ui-address>其实就是http://localhost/xhprof_html
//        
//      2. 你会在/tmp里面找到一个类似这样的文件：50f7ed6689205.test.xhprof，第一个部分就是run_id,当然，程序里面的$run_id跟这个是一样的。第二个部分的test就是你在save_run里面的第二个参数，第三个部分就是文件后缀不用去管
//
//      3. 例子中，要看到统计列表需要访问的地址就是http://localhost/xhpro_html/index.php?run=50f7ed6689205&source=test
//                 **************************/
//
function foo(){
    $a = 100;
}
//此处跳转到显示html路径(xhprof/xhprof_html下)
echo "<script language='javascript'>window.location.href = 'http://192.168.1.132:9998/?run=" . $run_id . "&source=test';</script>";  
```

### html结果

![html](http://ww1.sinaimg.cn/large/68faff51jw1eu3958bk3uj21es0lzwlq.jpg)

### 图片分析结果

![pic](http://ww4.sinaimg.cn/large/68faff51jw1eu3958qsoij21270iu0xd.jpg)
