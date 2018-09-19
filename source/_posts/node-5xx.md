---
layout: post
title:  "keepalive导致NodeJS服务502问题排查"
date:   2018-06-19 16:11:18
categories: 网络
---


最近加了http状态码的监控以后，发现线上有很多`502`的告警，这些请求并没有到后端node那里，

`Nginx`的error日志报错如下

```
2018/06/13 23:57:59 [error] 21230#0: *90124956 recv() failed (104: Connection reset by peer) while reading response header from upstream, client: 2.2.2.2, server: masaike.com, request: "POST /apis.json?action=USER_GET_PUBLIC_PROPERTIES&timestamp=1528905480372 HTTP/1.1", upstream: "http://10.22.22.22:22222/apis.json?action=USER_GET_PUBLIC_PROPERTIES&timestamp=1528905480372", host: "masaike.com""
```

抓了几个包，

![](/media/15293339434072/15293350477158.jpg)


看了几个出现问题的包以后，发现都是过了`5s`，`Nginx`向`node`发请求，`node`回了一个`RST`包，Nginx收到RST包以后，认为后端有问题，返回给客户端502

怀疑是Node的keepalive时间造成了，去搜了一下，默认值真的是5s: [node文档](https://nodejs.org/api/http.html#http_request_settimeout_timeout_callback)

```
server.keepAliveTimeout#
Added in: v8.0.0
<number> Timeout in milliseconds. Default: 5000 (5 seconds).
```

这个502出现的原因是：

5s超时时间到的瞬间，Node想断掉这个空闲连接，Nginx并不知道，还是发了HTTP请求过来，因为Node这个时候，可能已经发了FIN包，或者准备发FIN包，直接回了RST给Nginx，让它放弃这个连接，Nginx拿到RST，以为后端挂了，返回给客户端502


这里有好几个问题
1. 为什么node keepalive的时间到了，没有发送FIN包给Nginx
2. Nginx有没有参数可以解决后端keepalive超时的问题
3. 会不会对应用造成影响

第一个问题，这里的没有出现FIN包，不代表所有的都没有，我抓的另外一个包是node先回了Nginx FIN包，后回的RST。这里没出现FIN包，我猜可能的原因是，在FIN报还没发出去之前，下一个HTTP请求到来，系统应该会发送FIN和RST，在这个情况下，FIN包可有可无。

一般情况下，都是客户端想断开连接，这里的情况不太一样，是服务端主动断开的连接，这种情况下，会比较恶心，因为服务端发送FIN包以后，表示自己再也不会发送数据给客户端了，这种情况下，客户端再发送的请求，它是没有办法回应的，对它来说，回RST包是更好的选择，告诉客户端，放弃吧。

对比了一下`node`和`tomcat`的502情况，发现Java的特别少，怀疑两个http服务的默认超时时间不一样，经过试验，确实如此，tomcat的默认超时时间是60s，把出现的概率减小了不少。

把线上的node的server.keepAliveTimeout调整为60s以后，502的问题基本消失了（一天5个以内），改之前大概十几分钟就会出现一次

![](/media/15293339434072/15293783802240.jpg)


![](/media/15293339434072/15293783454467.jpg)


那么怎么彻底解决这个问题呢？

各路大神其实早就遇到了这个问题，淘宝的`tengine`的`ngx_http_upstream_keepalive_module
`很早就增加这个`patch`，为`upstream`增加了一个`keepalive_timeout`的指令

详见：[淘宝ngx_http_upstream_keepalive_module](http://tengine.taobao.org/document_cn/http_upstream_keepalive_timeout_cn.html)

```
ngx_http_upstream_keepalive_module
长连接超时设置

增加nginx后端长连接的超时支持:

upstream backend {
    server 127.0.0.1:8080;
    keepalive 32;
    keepalive_timeout 30s; # 设置后端连接的最大idle时间为30s
}
```

这样子，只要把这里的超时时间设置得比业务服务器小一点就可以了，这样就可以在业务服务器超时之前，Nginx先主动断掉空闲连接，下次过来重新建连接。

实测情况：

```
upstream express_server {
        server 10.211.55.2:3000;
        keepalive 16;
        keepalive_timeout 3s;
}
server {
        listen 80;
        server_name test.ya.centos001;
        location /test {
                proxy_http_version 1.1;
                proxy_redirect off;
                proxy_set_header Connection "";
                proxy_pass http://express_server/;
        }
}

curl test.ya.centos001/test
```


![](/media/15293339434072/15293777948818.jpg)


过了3s，Nginx就主动先断掉了连接，防止了后续的悲剧

就目前的情况来看，引入这个Nginx模块之前，暂时没有完美的办法解决。


第三个问题，会不会对实际用户造成影响？

it dependends

实测在GET请求下，Nginx会进行重试

```
172.18.101.241 - - [19/Jun/2018:16:18:14 +0800] "GET /node_keepalive_test?delay=488 HTTP/1.1" 200 207 "-" "okhttp/3.10.0" "-" "172.18.101.241:3000, 172.18.101.241:3000" "0.008" "0.001, 0.007"
```

在POST请求下，Nginx会直接返回502

```
172.18.98.18 - - [20/Jun/2018:12:05:37 +0800] "POST /users/?delay=477 HTTP/1.1" 502 179 "-" "okhttp/3.10.0" "-" "172.18.98.18:3000" "0.002" "0.002" test.ya.psd010
```

Nginx文档：

```
Syntax: proxy_next_upstream error | timeout | invalid_header | http_500 | http_502 | http_503 | http_504 | http_403 | http_404 | non_idempotent | off ...;
Default:    
proxy_next_upstream error timeout;
Context:    http, server, location

error
an error occurred while establishing a connection with the server, passing a request to it, or reading the response header;
timeout
a timeout has occurred while establishing a connection with the server, passing a request to it, or reading the response header
```

默认情况下它会在error和timeout的时候会重试，我们这个场景下，是一个error(reading response header from upstream)，但是只会重试幂等的请求（Nginx认为POST, LOCK, PATCH是非幂等的，其它的是幂等的）

所以出现了上面的情况

怎么样复现?

```java
var client = OkHttpClient.Builder().connectionPool(ConnectionPool(1, 10, TimeUnit.MINUTES))
        .build()

fun run(url: String, delay: Int): String {
    val request = Request.Builder()
            .url(url)
            .build()

    val response = client.newCall(request).execute()
    println("delay: $delay, status : ${response.code()}")
    return response.body()!!.string()
}
fun runPost(url: String, delay: Int): String {
    val request = Request.Builder()
            .url(url)
            .post(RequestBody.create(MediaType.parse("application/json"), "{}"))
            .build()

    val response = client.newCall(request).execute()
    println("delay: $delay, status : ${response.code()}")
    return response.body()!!.string()
}
fun main(args: Array<String>) {
    var delay = 502
    while (true) {
        runPost("http://test.ya.psd010/users/?delay=$delay", delay)
//        run("http://test.ya.psd010/users/?delay=$delay", delay)
        delay -= 1
        TimeUnit.MILLISECONDS.sleep(delay.toLong())
        if (delay < 470) {
            delay = 502
        }

    }
}
```