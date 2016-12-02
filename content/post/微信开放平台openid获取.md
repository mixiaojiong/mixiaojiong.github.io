+++
title = "微信开放平台openid获取"
date = "2015-07-08T20:42:56+08:00"
tags = ["other"]
author = "xiaojiong"

+++

通过点击微信中自定义菜单，直接获取用户**openid(用户唯一标识)**：
> 首先在后台（开发者中心->接口权限表中查找是否有**网页授权获取用户基本信息**接口权限，如果没有需要申请。

### 设置自定义菜单
> 在http://mp.weixin.qq.com/debug/ 中选择自定义菜单-》自定义菜单接口创建-》json结构如下
```
{
    "button":[
    { 
        "type":"click",
            "name":"今日歌曲",
            "key":"V1001_TODAY_MUSIC"
    },
    {
        "name":"菜单",
        "sub_button":[
        {   
            "type":"view",
            "name":"test",
            "url":"https://open.weixin.qq.com/connect/oauth2/authorize?appid=wx520c15f417810387&redirect_uri=http%3A%2F%2Fchong.qq.com%2Fphp%2Findex.php%3Fd%3D%26c%3DwxAdapter%26m%3DmobileDeal%26showwxpaytitle%3D1%26vb2ctag%3D4_2030_5_1194_60&response_type=code&scope=snsapi_base&state=123#wechat_redirect"
        },
        {
            "type":"click",
            "name":"赞一下我们",
            "key":"V1001_GOOD"
        }]
    }]
}
```

### 网页授权的两种方式
* snsapi_base （用户无感知）这里我们选择这种
* snsapi_userinfo （如果用户没有关注公众号，需要点击授权页）

### 第一步获取code
> https://open.weixin.qq.com/connect/oauth2/authorize?appid=APPID&redirect_uri=REDIRECT_URI&response_type=code&scope=SCOPE&state=STATE#wechat_redirect 

### 第二步：通过code换取网页授权access_token
```
$code = $_REQUEST['code'];
$state = $_REQUEST['state'];
if (!Comm_Context::Cookie('openid')) {
    error_log("hehe\n", 3, '/tmp/a.log');                       
    $url = "https://api.weixin.qq.com/sns/oauth2/access_token?appid=wx8528b110df0f46bb&secret=578d8bdfdb6ff8964ab06c442f062db1&code={$code}&grant_type=authorization_code";
    $res = Comm_Curl::request($url);
    $res  = json_decode($res, true);
    Tool_Login::mkcookie('openid', $res['openid']);
}   
Comm_Tools::redirect($state);

```
### 关于微信内置的调试环境
> 在开放者工具->接口测试系统申请，可以申请测试环境（支持回调地址为IP）。
