---
layout: post
title:  "一次 tomcat 400 异常"
date:   2018-04-27 16:11:18
categories: Tomcat
---

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




