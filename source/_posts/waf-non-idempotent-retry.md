---
layout: post
title:  "阿里云WAF非幂等请求重试的坑"
date:   2018-06-21 16:11:18
categories: 网络
---


# 阿里云WAF非幂等请求重试的坑

这段时间遇到了一个诡异的问题，用`curl`或`PostMan`去调用阿里云（经过了WAF）上一个需要很长时间返回的后端接口（可能在半小时以上），发现后端居然会处理三次

![](/media/15295844321816/15295874011181.jpg)


这种问题，排查的思路是什么呢？

- 第一，当然是看客户端是不是有重试，wireshark抓包如下

![](/media/15295844321816/15295869649989.jpg)

发现并没有重试，事情就变得比较有意思了。

另外
`curl`在默认情况下，只有`connect`的超时时间（2min），并没有`read`的超时时间，也就是`curl`在`connect`成功以后会永久等待。可以通过参数值来设置超时时间

```
-m, --max-time <seconds>
    Maximum  time  in  seconds that you allow the whole operation to
    take.  This is useful for preventing your batch jobs from  hang‐
    ing  for  hours due to slow networks or links going down.  Since
    7.32.0, this option accepts decimal values, but the actual time‐
    out will decrease in accuracy as the specified timeout increases
    in decimal precision.  See also the --connect-timeout option.

    If this option is used several times, the last one will be used.
```

另外，这里的`keepalive`心跳包也比较有意思，后面再讲

- 第二，在`Nginx`机器抓包，验证，真的有请求多次

![](/media/15295844321816/15295874991093.jpg)

发现每到2分钟`Nginx`就主动断掉了连接，重新建连接，发起请求

- 第三，在不经过`WAF`的情况下，调用看会不会有问题

怎么样不经过WAF呢？在阿里云内网，指定`care.cctv.com`到一台nginx上就可以了，主要通过`curl`的`--resolve`来实现

```
curl -v -X POST \
http://care.cctv.com/init/daily \
--resolve care.cctv.com:80:2.2.2.2
```

![](/media/15295844321816/15295876827707.jpg)

为什么是`900s`？

`Nginx`配置大概如下：

```
upstream  express_server{
        server localhost:3000;
        keepalive 16;
}

server {
        listen       80;
        server_name test.ya.local;

        access_log  /usr/local/var/log/nginx/host.test.ya.me.access.log  main buffer=64k flush=1s;


        location /{
                proxy_pass http://express_server;
                proxy_redirect      off;
                proxy_set_header                X-Forwarded-Url         "$scheme://$host$request_uri";
                proxy_http_version  1.1;
                proxy_set_header    Connection "";
                proxy_set_header    Cookie $http_cookie;
                proxy_set_header    Host $host;
                proxy_set_header    X-Real-IP $remote_addr;
                proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_connect_timeout           900;
                proxy_send_timeout              900;
                proxy_read_timeout              900;
        }
}
```

在`Nginx`配置这里，`read`的超时时间就是`900s`

阿里云`WAF`坑的地方，在于对于非幂等的`POST`请求，依然进行了重试，完全没有节操

