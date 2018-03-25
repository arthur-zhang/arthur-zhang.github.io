---
layout: post
title:  "JVM DNS缓存的理解误区"
date:   2018-03-25 14:17:43
categories: JVM
---

最近有一个MySQL主从切换会不会影响业务，以及影响业务多久的问题，从我们在网上google得到资料都会告诉你跟`networkaddress.cache.ttl` `networkaddress.cache.negative.ttl`这两个值有关，这两个值在`jre/lib/security/java.security`中进行定义，打开看一下，发现默认情况下
`networkaddress.cache.ttl`被注释掉了，`networkaddress.cache.negative.ttl`值是10

```
#
# The Java-level namelookup cache policy for successful lookups:
#
# any negative value: caching forever
# any positive value: the number of seconds to cache an address for
# zero: do not cache
#
# default value is forever (FOREVER). For security reasons, this
# caching is made forever when a security manager is set. When a security
# manager is not set, the default behavior in this implementation
# is to cache for 30 seconds.
#
# NOTE: setting this to anything other than the default value can have
#       serious security implications. Do not set it unless
#       you are sure you are not exposed to DNS spoofing attack.
#
#networkaddress.cache.ttl=-1
```

我们看到第一句`default value is forever`，就以为会永久缓存，但是实际表现的行为并不是这样。我们切换DNS解析，断掉MySQL的连接，Java业务在不重启的情况下会自动重连新的ip，根本不是永久缓存。那到底缓存是不是永久缓存，如果不是，缓存多久，就值得我们去研究了。

`不然我们以后切DNS解析，都需要慎重考虑JVM对DNS缓存带来对业务方的的负面影响了`

#### 1.先来做一个实验，验证是不是永久缓存

实验步骤是模拟主从切换。 在本机搭建dnsmasq服务（brew install 就好），写一个Java的查数据的程序使用域名`test1.ya.me:3306`去连接数据，2s查询一次数据库。
跑一段时间，切换DNS，并没有生效，因为是使用的连接池，后续并没有建连接，在MySQL那里kill连接以后，发现马上重连到了新的ip上。

也就是`缓存策略一定不是永久缓存`

#### 2.到底是什么策略

经过debug，跟到了`java.net.InetAddress`里面这个函数

![](/images/15219572407358.jpg)

现在就很清楚了，policy等于30，也就是缓存30s过期

从dns解析协议层面，我们再抓包看一下，到底是不是这样的。
这个时候我写了一个不带连接池的数据库代码，每次都去新建连接（未释放）

```java
public class MyDataSource implements DataSource{

    public Connection getConnection() throws SQLException {
        Properties properties = new Properties();
        properties.put("username", "masaike");
        properties.put("password", "masaike@me");
        Connection conn = DriverManager.getConnection(
                "jdbc:mysql://test1.ya.me:3306/masaike?useUnicode=true&characterEncoding=UTF-8&rewriteBatchedStatements=true&useSSL=false",
                properties);
        return conn;
    }
}
```

![](/images/15219574662441.jpg)

这下就很清楚了
`30s，缓存会过期，如果需要新建连接的话，就会去解析DNS，拿到新的ip`。

那个文档里其实有写这个
> When a security manager is not set, the default behavior in this implementation is to cache for 30 seconds.

只是我们不知道，到底这个if else会到哪一个。

那么对于我们后续的工作的意义有如下几个

- 所有网络请求，优先考虑使用连接池

对于频繁建立连接、断开连接的应用，需要把DNS缓存时间设置长一点，避免频繁的DNS解析（比如不用连接池技术的http连接库，线上很多http调用慢，也有一部分可能是这个原因）

- 切换DNS后，需要断掉以前的所有连接，以免因为连接池导致未连接到新的ip上





