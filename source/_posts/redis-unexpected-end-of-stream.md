---
layout: post
title:  "一次 Redis Unexpected end of stream 的除错分析记录"
date:   2017-11-25 21:17:43
categories: "redis"
---

    
这几天线上出现了偶发性jedis的异常，堆栈如下

```java
redis.clients.jedis.exceptions.JedisConnectionException: Unexpected end of stream.
    at redis.clients.util.RedisInputStream.ensureFill(RedisInputStream.java:199)
    at redis.clients.util.RedisInputStream.readByte(RedisInputStream.java:40)
    at redis.clients.jedis.Protocol.process(Protocol.java:151)
    at redis.clients.jedis.Protocol.read(Protocol.java:215)
    at redis.clients.jedis.Connection.readProtocolWithCheckingBroken(Connection.java:340)
    at redis.clients.jedis.Connection.getBinaryBulkReply(Connection.java:259)
    at redis.clients.jedis.Connection.getBulkReply(Connection.java:248)
    at redis.clients.jedis.Jedis.get(Jedis.java:153)
```

看jedis的源码如下：

```java
  private void ensureFill() throws JedisConnectionException {
    if (count >= limit) {
      try {
        limit = in.read(buf);
        count = 0;
        if (limit == -1) {
          // 这里抛了异常
          throw new JedisConnectionException("Unexpected end of stream.");
        }
      } catch (IOException e) {
        throw new JedisConnectionException(e);
      }
    }
  }
```

看起来是连接关闭的问题，但是为什么会出现呢？
我们看了当时出问题的时间点附近的redis连接数，发现了点东西

![](/images/15115949120236.jpg)


对应业务告警如下
![](/images/15115950836709.jpg)

时间点上比较接近


但是又不完全对的上时间

![](/images/15115951848199.jpg)


![](/images/15115952364777.jpg)

也就是峰值过后的第十分钟才出现，查看了jedis的配置文件如下：

```
redis.maxTotal = 5000000  // 值的合理性先不管他
redis.maxIdle = 1000      // 值的合理性先不管他
redis.minEvictableIdleTimeMillis = 300000    // 连接空闲大于5min的
redis.numTestsPerEvictionRun = 3             // 每次检查3个连接
redis.timeBetweenEvictionRunsMillis = 60000  // 每60s执行一次
redis.hostName = redis-master-1.gz.cvte.cn
redis.port = 6379
redis.timeout = 2147483647
redis.usePool = true
```

一开始，我们是怀疑jedis使用的commons-pool的空闲连接回收策略导致的，按上面的配置文件
每60s执行一次，每次检查3个连接，检查的条件是存活了5min

那我就是写代码模拟一下这种情况，连接暴涨之后，因为回收策略的问题，导致的连接使用的问题，但是我不管怎么配置，都不能模拟出`Unexpected end of stream`的情况，那我就怀疑到底是不是因为回收的问题，因为上面的回收配置来看，10分钟也释放不了那么多空闲连接（1次/min * 10min * 3 = 30）

那十分钟这个到底怎么来的，困扰了我几个小时，后来我去查看了一些redis的配置，好像想起来了什么
我们redis的配置文件设置的是`600s`超时，也就是`10`分钟

```
127.0.0.1:6379> config get timeout
1) "timeout"
2) "600"
127.0.0.1:6379>
```

看起来非常像了，那我再模拟一下，我把本地的redis的超时时间设置成2s，然后，我来试一下

```java
fun main(args: Array<String>) {
    val jedis = RedisHelper.getInstance().get()
    try {
        jedis.get("helo")
        Thread.sleep(3000)
        jedis.get("hello")
    } catch (ex: Exception) {
        ex.printStackTrace()
    }
    System.`in`.read()
}
```

激动得我差点留下了悔恨的泪水

```java
redis.clients.jedis.exceptions.JedisConnectionException: Unexpected end of stream.
    at redis.clients.util.RedisInputStream.ensureFill(RedisInputStream.java:199)
    at redis.clients.util.RedisInputStream.readByte(RedisInputStream.java:40)
    at redis.clients.jedis.Protocol.process(Protocol.java:151)
    at redis.clients.jedis.Protocol.read(Protocol.java:215)
    at redis.clients.jedis.Connection.readProtocolWithCheckingBroken(Connection.java:340)
    at redis.clients.jedis.Connection.getBinaryBulkReply(Connection.java:259)
    at redis.clients.jedis.Connection.getBulkReply(Connection.java:248)
    at redis.clients.jedis.Jedis.get(Jedis.java:153)
```


2s 连接空闲，redis 回了一个`fin`包，来关闭这个连接

![](/images/15115965826180.jpg)

那我们再拿来用，就抛了`JedisConnectionException: Unexpected end of stream`的异常


当然你会说，我们实际使用的例子，用完一次就会换回去池子里，不会拿了连接，发一次命令，过一会再发

其实这个不是个什么问题，本质上是一样的，不信我们来试一下，这次我们把连接池大小设置的小一点，比如1

```java
fun main(args: Array<String>) {
    val context = ClassPathXmlApplicationContext("classpath:*.xml")
    context.start()
    val redisTemplate: RedisTemplate<Serializable, Serializable> = context.getBean(RedisTemplate::class.java) as RedisTemplate<Serializable, Serializable>
    redisTemplate.opsForValue().get("hallo")
    Thread.sleep(3000)
    redisTemplate.opsForValue().get("hallo")
    System.`in`.read()
}
```
嗯，出现了跟线上一模一样的堆栈

```java
Exception in thread "main" org.springframework.data.redis.RedisConnectionFailureException: Unexpected end of stream.; nested exception is redis.clients.jedis.exceptions.JedisConnectionException: Unexpected end of stream.
    at org.springframework.data.redis.connection.jedis.JedisExceptionConverter.convert(JedisExceptionConverter.java:67)
    at org.springframework.data.redis.connection.jedis.JedisExceptionConverter.convert(JedisExceptionConverter.java:41)
    at org.springframework.data.redis.PassThroughExceptionTranslationStrategy.translate(PassThroughExceptionTranslationStrategy.java:37)
    at org.springframework.data.redis.FallbackExceptionTranslationStrategy.translate(FallbackExceptionTranslationStrategy.java:37)
    at org.springframework.data.redis.connection.jedis.JedisConnection.convertJedisAccessException(JedisConnection.java:242)
    at org.springframework.data.redis.connection.jedis.JedisConnection.get(JedisConnection.java:1220)
    at org.springframework.data.redis.core.DefaultValueOperations$1.inRedis(DefaultValueOperations.java:46)
    at org.springframework.data.redis.core.AbstractOperations$ValueDeserializingRedisCallback.doInRedis(AbstractOperations.java:57)
    at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:207)
    at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:169)
    at org.springframework.data.redis.core.AbstractOperations.execute(AbstractOperations.java:91)
    at org.springframework.data.redis.core.DefaultValueOperations.get(DefaultValueOperations.java:43)
    at JedisTest5Kt.main(JedisTest5.kt:24)
Caused by: redis.clients.jedis.exceptions.JedisConnectionException: Unexpected end of stream.
    at redis.clients.util.RedisInputStream.ensureFill(RedisInputStream.java:199)
    at redis.clients.util.RedisInputStream.readByte(RedisInputStream.java:40)
    at redis.clients.jedis.Protocol.process(Protocol.java:151)
    at redis.clients.jedis.Protocol.read(Protocol.java:215)
    at redis.clients.jedis.Connection.readProtocolWithCheckingBroken(Connection.java:340)
    at redis.clients.jedis.Connection.getBinaryBulkReply(Connection.java:259)
    at redis.clients.jedis.BinaryJedis.get(BinaryJedis.java:244)
    at org.springframework.data.redis.connection.jedis.JedisConnection.get(JedisConnection.java:1218)
    ... 7 more
```


我们怎么避免上面的情况出现呢？
目前来看，牺牲一部分性能开启`testOnBorrow`是比较稳妥的，就是每次拿连接，都去ping一下，看连接是否可用，不可用就重建连接，我可以试一下，开了以后，程序完全正常

上面配置还有哪些问题呢？

1. 连接设置的过大，推荐200以下
2. 空闲连接检查设置的不合理，现在看起来好像没有太大作用
3. 客户端最大timeout设置的过大

