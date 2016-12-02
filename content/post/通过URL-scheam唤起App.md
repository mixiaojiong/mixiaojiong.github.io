+++
title = "通过URL scheam唤起App"
date = "2015-08-04T20:51:20+08:00"
tags = ["other"]
author = "xiaojiong"

+++

URI scheam 是什么？详情见https://en.wikipedia.org/wiki/URI_scheme。通俗的讲：

* http:// 打开的是网址。
* qq://打开的是qq：
* mail:://打开的是邮箱

### 实现方式
1.1 安卓：为Android应用的启动Activity设置一个Schema，如下：

``` 
<data android:host="splash" android:scheme="youselfname"/> 
```

1.2 苹果：参考

```
http://objcio.com/blog/2015/05/21/the-complete-tutorial-on-ios-slash-iphone-custom-url-schemes/
```

2.用户点击浏览器中的链接时，在动态创建一个不可见的iframe，并且让这个iframe去加载步骤1中的Schema
3.如果在指定的时间内没有跳转成功，则当前页跳转到apk的下载地址（或下载页）
4.html代码

```
<html>
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">

    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black" />

    <title>scheam</title>
    <meta id="viewport" name="viewport" content="width=device-width,initial-scale=1.0,minimum-scale=1.0,maximum-scale=1.0,minimal-ui">
</head>
<body>
    <div>
        <!--a id="J-call-app" href="javascript:;" class="label">立即打开&gt;&gt;</a-->
        <input id="J-download-app-adr" type="hidden" name="storeurl" value="downloadUrl">
        <input id="J-download-app-ios" type="hidden" name="storeurl" value="https://itunes.apple.com/us/app/yi-ke/id911325680?l=zh&ls=1&mt=8">
    </div>

    <script>
        (function()
        {
            var ua = navigator.userAgent.toLowerCase();
            var t;
            var config = {
                /*scheme:必须*/
                scheme_IOS: 'youselfname://',
                scheme_Adr: 'youselfname//splash',
                download_url: document.getElementById(
                    'J-download-app-adr').value,
                timeout: 600
            };
            config.download_url = ua.indexOf('os') > 0 ? document.getElementById('J-download-app-ios').value :
                    config.download_url;

            function openclient()
            {
                var startTime = Date.now();

                var ifr = document.createElement('iframe');
                

                ifr.src = ua.indexOf('os') > 0 ? config.scheme_IOS :
                    config.scheme_Adr;
                ifr.style.display = 'none';
                document.body.appendChild(ifr);

                var t = setTimeout(function()
                {
                    var endTime = Date.now();

                    if (!startTime || endTime - startTime <
                        config.timeout + 200)
                    {
                        window.location = config.download_url;
                    }
                    else
                    {

                    }
                }, config.timeout);

                window.onblur = function()
                {
                    clearTimeout(t);
                }
            }
            openclient();
            /*
            window.addEventListener("DOMContentLoaded", function()
            {
                document.getElementById("J-call-app").addEventListener(
                    'click', openclient, false);

            }, false);
            */
        })()
    </script>
</body>
</html>

```
