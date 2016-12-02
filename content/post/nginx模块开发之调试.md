+++
title = "nginx模块开发之调试"
date = "2016-06-06T05:29:00+08:00"
tags = ["nginx"]
author = "xiaojiong"

+++


### nginx 模块开发之调试

这里以nginx稳定版本1.8.1为例。

### 下载与安装

wget http://nginx.org/download/nginx-1.8.1.tar.gz
./configure --prefix=~/Code/dst --with-debug --add-module=../module/test

### 前台单进程方式

```
daemon off;#关闭守护进程，使之在前台运行
master_process off;  #关闭主进程，使只有一个进程
```


### gdb调试:


```
~/Code/dst/sbin gdb nginx
GNU gdb (GDB) 7.9.1
Copyright (C) 2015 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-apple-darwin15.0.0".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from nginx...done.
(gdb) l handler
33	    ngx_http_core_loc_conf_t *corecf;
34	    corecf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
35	    corecf->handler = handler;
36	    return NGX_CONF_OK;
37	};
38	static ngx_int_t handler(ngx_http_request_t *req) {
39	    u_char html[1024] = "<h1>This is Test Page!</h1>";
40	    req->headers_out.status = 200;
41	    int len = sizeof(html) - 1;
42	    req->headers_out.content_length_n = len;
(gdb) b 42
Breakpoint 1 at 0x100079d37: file ../module/test/ngx_http_test_module.c, line 42.
(gdb) run
Starting program: /Users/xiaojiong/Code/dst/sbin/nginx
```

此时可以发现当前终端gdb被阻塞。新开一个终端执行：

```
curl localhost:8081
```

这时候gdb的终端显示为:

```
(gdb) run
Starting program: /Users/xiaojiong/Code/dst/sbin/nginx

Breakpoint 1, handler (req=0x101804450) at ../module/test/ngx_http_test_module.c:42
42	    req->headers_out.content_length_n = len;
(gdb) p req
$1 = (ngx_http_request_t *) 0x101804450
```


### 后台守护进程与worker方式

一般nginx在生产环境都是master为守护进程，然后多个woker处理请求的方式。所以调试的时候也采用这种方式。

配置文件：

```
#daemon off;
#master_process off;
worker_processes  1; #设置为1是为了方便调试。
```

gdb调试:

```
~/Code/dst gdb sbin/nginx
GNU gdb (GDB) 7.9.1
Copyright (C) 2015 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-apple-darwin15.0.0".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from sbin/nginx...done.
(gdb) run
Starting program: /Users/xiaojiong/Code/dst/sbin/nginx
[Inferior 1 (process 2018) exited normally]
(gdb) attach 2021
Attaching to program: /Users/xiaojiong/Code/dst/sbin/nginx, process 2021
0x00007fff8fcf0eca in kevent () from /usr/lib/system/libsystem_kernel.dylib
(gdb) l handler
33	    ngx_http_core_loc_conf_t *corecf;
34	    corecf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
35	    corecf->handler = handler;
36	    return NGX_CONF_OK;
37	};
38	static ngx_int_t handler(ngx_http_request_t *req) {
39	    u_char html[1024] = "<h1>This is Test Page!</h1>";
40	    req->headers_out.status = 200;
41	    int len = sizeof(html) - 1;
42	    req->headers_out.content_length_n = len;
(gdb) b 42
Breakpoint 1 at 0x100079d37: file ../module/test/ngx_http_test_module.c, line 42.
(gdb) c
Continuing.

Breakpoint 1, handler (req=0x101001050) at ../module/test/ngx_http_test_module.c:42
42	    req->headers_out.content_length_n = len;
(gdb) p req
$1 = (ngx_http_request_t *) 0x101001050
(gdb) detach
Detaching from program: /Users/xiaojiong/Code/dst/sbin/nginx, process 2021
```


attach 2021, 2021为我机器上的worker进程号，你可以用ps aux | grep nginx查看，detach 为退出调试。
