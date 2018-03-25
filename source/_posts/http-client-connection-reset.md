---
layout: post
title:  "一次 http client 偶发性Connection Reset的除错分析记录"
date:   2018-01-30 01:10:43
categories: "tcp"
---


线上http client偶发性报`Connection Reset`的问题

```java
[2018-01-24 09:38:22,162][http-nio-0.0.0.0-8084-exec-1782][ERROR][com.masaike.care.common.utils.HttpUtils:288] Connection reset
java.net.SocketException: Connection reset
    at java.net.SocketInputStream.read(SocketInputStream.java:209)
    at java.net.SocketInputStream.read(SocketInputStream.java:141)
    at org.apache.http.impl.io.SessionInputBufferImpl.streamRead(SessionInputBufferImpl.java:136)
    at org.apache.http.impl.io.SessionInputBufferImpl.fillBuffer(SessionInputBufferImpl.java:152)
    at org.apache.http.impl.io.SessionInputBufferImpl.readLine(SessionInputBufferImpl.java:270)
    at org.apache.http.impl.conn.DefaultHttpResponseParser.parseHead(DefaultHttpResponseParser.java:140)
    at org.apache.http.impl.conn.DefaultHttpResponseParser.parseHead(DefaultHttpResponseParser.java:57)
    at org.apache.http.impl.io.AbstractMessageParser.parse(AbstractMessageParser.java:260)
    at org.apache.http.impl.DefaultBHttpClientConnection.receiveResponseHeader(DefaultBHttpClientConnection.java:161)
    at org.apache.http.impl.conn.CPoolProxy.invoke(CPoolProxy.java:138)
    at org.apache.http.protocol.HttpRequestExecutor.doReceiveResponse(HttpRequestExecutor.java:271)
    at org.apache.http.protocol.HttpRequestExecutor.execute(HttpRequestExecutor.java:123)
    at org.apache.http.impl.execchain.MainClientExec.execute(MainClientExec.java:254)
    at org.apache.http.impl.execchain.ProtocolExec.execute(ProtocolExec.java:195)
    at org.apache.http.impl.execchain.RetryExec.execute(RetryExec.java:86)
    at org.apache.http.impl.execchain.RedirectExec.execute(RedirectExec.java:108)
    at org.apache.http.impl.client.InternalHttpClient.doExecute(InternalHttpClient.java:186)
    at org.apache.http.impl.client.CloseableHttpClient.execute(Unknown Source)
    at org.apache.http.impl.client.CloseableHttpClient.execute(Unknown Source)
    at com.masaike.care.common.utils.HttpUtils.sendHttpRequest(HttpUtils.java:197)
    at com.masaike.care.remote.impl.service.AdminServiceApiImpl.getCodeV2(AdminServiceApiImpl.java:271)
    at com.masaike.care.common.impl.service.AccountServiceImpl.getDynamicCode(AccountServiceImpl.java:4305)
    at com.masaike.care.common.impl.service.AccountServiceImpl.sendVerificationCode(AccountServiceImpl.java:3344)
```

拿到这个问题，我的第一反应是连接失效导致的，因为类似的问题，在redis连接池已经出现过（具体可以参见我的博客 http://arthur-zhang.github.io/2017/11/25/redis-unexpected-end-of-stream/ ）

问题来了，什么时候连接会被断掉？一种最常见的可能是`nginx`的`keepalive`，有关系的有两个参数`keepalive_timeout`和`keepalive_requests`

``` 
keepalive_timeout 默认值75s;

The first parameter sets a timeout during which a keep-alive client connection will stay open on the server side. The zero value disables keep-alive client connections. 

keepalive_requests 默认值 100;
Sets the maximum number of requests that can be served through one keep-alive connection. After the maximum number of requests are made, the connection is closed.
```

我们来实测一下
首先来测试一下`keepalive_timeout`

```
http {
    keepalive_timeout  10s;
}
```

写个代码测试一下

```java
String url = "http://test.me.com/rest-ip.json";
JSONObject obj = HttpUtils.sendHttpRequest(url, new HashMap<>(), "GET");
System.out.println(obj);
System.in.read();
```
tcpdump抓包如下

![](/images/nginx-keepalvie1.png)


可以看到10s以后，`nginx`就`主动`断掉了此http1.1的keep-alive连接

那么一种可能的bug产生方式就是我拿到了一个即将过期的的连接，在执行http请求前的一瞬间刚好被nginx断掉了

怎么模拟呢？自然想到的方式是，连建立连接，然后等连接失效，然后用这个失效的连接去发起http请求，肯定会失败


```java
String url = "http://test.me.com/rest-ip.json";
 
for (int i = 0; i < Integer.MAX_VALUE; i++) {
    System.out.println("\n##########>>>>>>>>" + i);
    JSONObject a = HttpUtils.sendHttpRequest(url, new HashMap<>(), "GET");
    System.out.println(new Date() + ">>" + a);
}
```


采用`debug`的方式，可以轻松的复现，方法之一是先访问一次（这个时候会从连接池申请一条连接），然后再次从连接池获取连接，debug断点等超时，然后等nginx超时发送fin包，然后debug继续，用这条有问题的连接去请求nginx服务器，果然抛出了我们想要的`Connection Reset`

![](/images/connection-reset1.png)

查看wireshark，可以看到Nginx果然回了一个RST包

![](/images/connection-reset2.png)


然后，最外层的`HttpUtils.sendHttpRequest`并没有执行抛出我们想要的异常，正常执行了。
仔细看，原来刁刁的apache项目不是盖的，它是有retry的！

![](/images/connection-reset3.png)

这个时候，其实是比较困惑的，难道不是因为这个原因嚒。

继续debug一下，发现它有一个是否需要retry的判断，原来retry也是有条件的，不是所有的都需要retry，那点进去看一下

```java
    public boolean retryRequest(
            final IOException exception,
            final int executionCount,
            final HttpContext context) {
        // 判断重试次数
        if (executionCount > this.retryCount) {
            // Do not retry if over max retry count
            return false;
        }
        // 判断是不是在异常黑名单
        if (this.nonRetriableClasses.contains(exception.getClass())) {
            return false;
        } else {
            for (final Class<? extends IOException> rejectException : this.nonRetriableClasses) {
                if (rejectException.isInstance(exception)) {
                    return false;
                }
            }
        }
        final HttpClientContext clientContext = HttpClientContext.adapt(context);
        final HttpRequest request = clientContext.getRequest();

        if(requestIsAborted(request)){
            return false;
        }

        if (handleAsIdempotent(request)) {
            // Retry if the request is considered idempotent
            return true;
        }

        if (!clientContext.isRequestSent() || this.requestSentRetryEnabled) {
            // Retry if the request has not been sent fully or
            // if it's OK to retry methods that have been sent
            return true;
        }
        // otherwise do not retry
        return false;
    }

```

这个时候，英语好不好就有用了，`handleAsIdempotent`，Idempotent这个单词太熟悉不过了，`幂等`。原来幂等才需要retry，那么它怎么知道需不需要retry？

```java
    protected boolean handleAsIdempotent(final HttpRequest request) {
        return !(request instanceof HttpEntityEnclosingRequest);
    }
```
我们印象中，GET请求应该是幂等的，POST请求应该不是幂等的，那么我们来试一哈，改一下代码

测试以后确实，`GET`请求`handleAsIdempotent`返回的是`true`，POST请求`handleAsIdempotent`返回的是`false`

```java
        String url = "http://test.me.com/rest-ip.json";

        for (int i = 0; i < Integer.MAX_VALUE; i++) {
            System.out.println("\n##########>>>>>>>>" + i);
            JSONObject a = HttpUtils.sendHttpRequest(url, new HashMap<>(), "POST");
            System.out.println(new Date() + ">>" + a);
        }
```

再来debug一下，`HttpUtils.sendHttpRequest`终于挂了

![](/images/connection-reset4.png)

然后查了下，我们线上挂掉的接口，正好都是POST请求。

其实所有以上的分析，大概总结为：

因为某些原因（不一定是因为keepalive），httpclient的连接池拿到了一条失效的连接去发起http请求，发生了Connection Reset，然后因为是POST请求，它认为可能不是幂等的，没有进行retry，导致了最终的程序异常

番外篇：

其实apache这个httpclient写得挺牛叉的，
1、对幂等请求可以retry
2、它在每个http连接发起之前，都会进行失效检查（默认行为，可关闭），类似于redis等连接池中的`testOnBorrow`，可能会有最多30ms的延时。4.3版本以上已废弃这种方式，采用了更高效的实现方式。

```java
if (config.isStaleConnectionCheckEnabled()) {
  // validate connection
  if (managedConn.isOpen()) {
      this.log.debug("Stale connection check");
      if (managedConn.isStale()) { /// 进行失效检测
          this.log.debug("Stale connection detected");
          managedConn.close();
      }
  }
}
```

但是这些都无法保证，真正执行的时候，http连接还是好的（这个其实是无解的），但是也没什么太大问题，这种情况，retry一次基本上就ok了。

怎么解这个问题呢？很简单，自己实现一个retry的逻辑，在`Connection reset`的情况下进行retry，不管是不是幂等的接口。

改完，测试，确实ok了。



