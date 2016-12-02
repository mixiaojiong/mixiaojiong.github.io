+++
title = "redis+lua实现发号器"
date = "2016-01-25T21:58:21+08:00"
tags = [
    "redis",
    "lua",
]
author = "xiaojiong"

+++


### 业务场景

 - 拿微博举例：发布微博以后会产生(微博id)，而像微博这样庞大的系统你刚刚发布的微博很可能没有落地。这时你的粉丝看见了你的微博进行评论，需要一个唯一标识（微博id)与评论进行关联。
 - 你的主键是需要有意义的，比如你的分表是按日期划分，那么你的主键可能需要是这样的：2015121200000009。
 
### 设计方案
 
 - 号码设计：采用52位二进制数，其中高28位为秒级时间戳，接下来的4位做业务区分，再接下来的4位做随机值（一会说为什么）。最后16位为自增主键。也就说每秒可以产生65535个唯一ID。
 - 采用redis+lua的设计方案，不需要持久化。只需要cache，因为redis不具有计算能力，需要结合lua脚本的计算能力。

### 问题

 - 如果某一秒钟已经发出1000个号码，但是这一秒服务重启了（很可能宕机以后立即重启，一瞬间），那么会导致这1000个号码会重复发放。
 - 此服务是单点的。
 - phpredis扩展有整型溢出的bug

### 解决方案

 - 四位随机数会解决服务重启以后发送重复号码的概率。
 - 两台机器，一台发奇数，一台偶数。HA可以做轮询。
 - [phpredis扩展修复记](http://www.xiaojiong.net/2016/01/13/phpredis%E6%89%A9%E5%B1%95%E4%BF%AE%E5%A4%8D%E8%AE%B0/)

### 相关代码

```
-- biz 01 - 15
local biz = tonumber(KEYS[1])
-- random 01 - 15
local random = tonumber(KEYS[2])
-- id 16位
local id = redis.call("INCRBY", "impulse", 2)
if (id) then
    -- max odd
    if (id >= 65535 and (id % 2 == 1)) then
        redis.call("SET", "impulse", -1)
    end
    -- max even number
    if (id >= 65534 and (id %2 == 0)) then
        redis.call("SET", "impulse", 0)
    end
else
    redis.log(redis.LOG_NOTICE, "impulse id false")
    return false
end
id = id % 65536
-- sec 10位
local sec = redis.call("TIME")[1]
if (sec) then
    sec = sec - 1420041600
else 
    redis.log(redis.LOG_NOTICE, "impulse sec false")
    return false
end
return sec*16777216 + biz*1048576 + random*65536 + id
```

### 参考资料
[业务系统需要什么样的ID生成器](http://ericliang.info/what-kind-of-id-generator-we-need-in-business-systems/)
