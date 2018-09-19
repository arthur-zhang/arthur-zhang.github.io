---
layout: post
title:  "一次URL长度导致的414 400异常分析"
date:   2018-04-27 16:11:18
categories: Tomcat
---

出现了两个案例：
1. 接口返回400
2. 接口返回414

# 先看第一个问题：状态码返回400

线上出现了几次接口400的异常，排查起来比较有意思，通过nginx代理过去就会400，通过ip直连过去，就不会出现，机智的我用tcpdump抓了一个包，搞下来看了一下，居然没有发现两种请求有什么不同（当然是我没发现）

我猜可能是刚好触发了tomcat的header最大大小的限制了，放到我自己编译的tomcat中开启debug模式，果然出现了异常

![](/images/15248405028586.jpg)


看到是Request header is too large异常，那到底是多大？




```java
 @Override
    public boolean parseHeaders()
        throws IOException {
        if (!parsingHeader) {
            throw new IllegalStateException(
                    sm.getString("iib.parseheaders.ise.error"));
        }

        HeaderParseStatus status = HeaderParseStatus.HAVE_MORE_HEADERS;

        do {
            status = parseHeader();
            // Checking that
            // (1) Headers plus request line size does not exceed its limit
            // (2) There are enough bytes to avoid expanding the buffer when
            // reading body
            // Technically, (2) is technical limitation, (1) is logical
            // limitation to enforce the meaning of headerBufferSize
            // From the way how buf is allocated and how blank lines are being
            // read, it should be enough to check (1) only.
            if (pos > headerBufferSize
                    || buf.length - pos < socketReadBufferSize) {
                throw new IllegalArgumentException(
                        sm.getString("iib.requestheadertoolarge.error"));
            }
        } while ( status == HeaderParseStatus.HAVE_MORE_HEADERS );
        if (status == HeaderParseStatus.DONE) {
            parsingHeader = false;
            end = pos;
            return true;
        } else {
            return false;
        }
    }

```

大小跟headerBufferSize有关，这个值最终是由决定

```java
AbstractHttp11Protocol.java

/**
* Maximum size of the HTTP message header.
*/
private int maxHttpHeaderSize = 8 * 1024;
public int getMaxHttpHeaderSize() { return maxHttpHeaderSize; }
```

可以看到是8192

这个值可以通过tomcat的配置文件进行修改，因为我觉得没什么修改的必要，已经够大了，就不放出来了。

那回到上面的问题，为什么经过了nginx就好出问题呢？

仔细看了wireshark的包，发现了真相，原理nginx会增加一个`X-Forwarded-Url`的header，会重复一遍url，我们本来的header在7k左右，然后有了这个header，瞬间double到了14k，超过了tomcat的header最大大小，被tomcat直接拒绝了。

![](/images/15248408539742.jpg)


如果你还不会从源码调试tomcat，那么这篇文章就对你有帮助啦

下面这个链接是可以直接用Intellj打开并进行debug的，基于tomcat8.0，做了一些细微的修改以便编译可以通过。

https://github.com/arthur-zhang/tomcat-code-reading


续上
近期发现 url 长度达到 2500 就跪了，跟我们之前看到的理论上支持 4000 左右（8192/2）相差比较远，抓包研究了一下，发现 Nginx 又丧心病狂的把 url 重复了三遍

![](/images/tomcat_url_3_times.png)

导致最多支持的url长度大概为 2700 以下，本例中是 2500，因为要算上其它的 header

## 第二个问题，返回414状态码
其实这个很好理解，一个请求到后端接口内部，经历了两层组件 `Nginx` 和 `Tomcat` 容器
上面那个是 `Tomcat` 限制了能处理的最大header大小，经过了一层 Nginx 导致 header 大了三倍左右，导致一个能经过 Nginx 的请求，在 Tomcat 层被拒绝掉，返回`400`状态码

那么Nginx层大小是怎么样的呢？
默认值也是8k

```
Syntax: large_client_header_buffers number size;
Default:    
large_client_header_buffers 4 8k;
Context:    http, server
Sets the maximum number and size of buffers used for reading large client request header. A request line cannot exceed the size of one buffer, or the 414 (Request-URI Too Large) error is returned to the client. A request header field cannot exceed the size of one buffer as well, or the 400 (Bad Request) error is returned to the client. Buffers are allocated only on demand. By default, the buffer size is equal to 8K bytes. If after the end of request processing a connection is transitioned into the keep-alive state, these buffers are released.
```

写一个代码从url长度8192往下找到第一个非414的请求，发现不太对

![](/images/nginx414-1.png)

为啥是`8295`？而不是8k(8192)
仔细看一下文档
`A request line cannot exceed the size of one buffer, or the 414 (Request-URI Too Large) error is returned to the client.`

意思是`request line`不能超过一个buffer的大小(8k)，否则会抛出 `414` 异常

什么是request line？
`http`的`RFC上`有写
`Request-Line   = Method SP Request-URI SP HTTP-Version CRLF`
也就是只包含 `method + uri + http-version CRLF`

我们把对应的部分拷贝出来，看下长度，不包含CRLF的长度是8190。
我们可以大致计算一下GET请求的URI在nginx上最大不能超过多少：
`8192 - (GET ).length - ( HTTP/1.1).length - (CRLF).length = 8192 - 4 - 9 - 2 = 8177`

跟我们用代码试出来的大小一样


