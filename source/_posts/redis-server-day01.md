---
layout: post
title:  "一起来写个redis server（day01）"
date:   2017-03-15 21:17:43
categories: 一起来写个redis-server
---



我们来一步一步用`Java`写一个`redis`的服务器，短期目标是实现所有的单机版本功能，长期目标是实现集群、类codis proxy功能。纯业余学习项目，一起来看看网络编程、NIO、并发、分布式的一些知识。

github地址 `https://github.com/arthur-zhang/redis-server`

今天的目标实现用`redis-cli`连接我们的服务器，完成`redis-cli`发送`ping`命令，我们返回`pong`


先用`tcpdump`抓个真正`redis-cli`和`redis-server` ping命令的包

```
    127.0.0.1.62054 > 127.0.0.1.6379: Flags [P.], cksum 0xfe36 (incorrect -> 0xcfd1), seq 14:28, ack 8, win 12457, options [nop,nop,TS val 1573721279 ecr 1573665837], length 14
	0x0000:  0200 0000 4500 0042 3b6c 4000 4006 0000  ....E..B;l@.@...
	0x0010:  7f00 0001 7f00 0001 f266 18eb 317b 8f7c  .........f..1{.|
	0x0020:  59da f12b 8018 30a9 fe36 0000 0101 080a  Y..+..0..6......
	0x0030:  5dcd 14bf 5dcc 3c2d 2a31 0d0a 2434 0d0a  ]...].<-*1..$4..
	0x0040:  7069 6e67 0d0a                           ping..

    127.0.0.1.6379 > 127.0.0.1.62054: Flags [P.], cksum 0xfe2f (incorrect -> 0x34dd), seq 9758:9765, ack 189, win 12752, options [nop,nop,TS val 1607541840 ecr 1607541835], length 7
	0x0000:  0200 0000 4502 003b afd6 4000 4006 0000  ....E..;..@.@...
	0x0010:  7f00 0001 7f00 0001 18eb f266 59db 1741  ...........fY..A
	0x0020:  317b 902b 8018 31d0 fe2f 0000 0101 080a  1{.+..1../......
	0x0030:  5fd1 2450 5fd1 244b 2b50 4f4e 470d 0a    _.$P_.$K+PONG..
```

结合[redis通信协议](https://redis.io/topics/protocol) 我们可以看到请求和响应如下

请求
`2A 31 0D 0A 24 34 0D 0A 70 69 6E 67 0D 0A` 对应 `*1\r\n$4\r\nping\r\n`

响应
`2B 50 4F 4E 47 0D 0A` 对应 `+PONG\r\n`

请求协议
```
*<参数数量> CR LF
$<参数 1 的字节数量> CR LF
<参数 1 的数据> CR LF
...
$<参数 N 的字节数量> CR LF
<参数 N 的数据> CR LF

```
redis的响应回复

- 状态回复（status reply）的第一个字节是 "+"
- 错误回复（error reply）的第一个字节是 "-"
- 整数回复（integer reply）的第一个字节是 ":"
- 批量回复（bulk reply）的第一个字节是 "$"
- 多条批量回复（multi bulk reply）的第一个字节是 "*"
这里的 *1\r\n$4\r\nping\r\n
```
┌───────────────┐
│argument count │
└───────────────┘
        ▲
        │
        │
  ┌───┬─┴─┬────┬───┬──┬────┬────┬────┐
  │ * │ 1 │\r\n│ $ │4 │\r\n│ping│\r\n│
  └───┴───┴────┴───┴┬─┴────┴────┴────┘
                    │
                    │
                    │
                    ▼
        ┌──────────────────────────┐
        │   argument#1 byte size   │
        └──────────────────────────┘
```

基本代码如下，监听6397，顺序处理请求，用`redis-cli`请求我们的server

```
redis-cli -p 6397
127.0.0.1:6397> ping
pong
```
部分代码如下

```java
public class RedisServer {
    private static final String PING = "ping";
    
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(6397);
        
        while (true) {
            System.out.println("process incoming call");
            Socket socket = serverSocket.accept();
            InputStream in = socket.getInputStream();
            OutputStream out = socket.getOutputStream();
            
            int read = in.read();
            // 开始符
            if (read == '*') {
                // 参数个数，第一个参数为命令的名字
                long numberArgs = Protocol.readLong(in);
                // 命令和命令参数部分
                byte[][] bytesArray = new byte[(int) numberArgs][];
                for (int i = 0; i < numberArgs; i++) {
                    if (in.read() == '$') {
                        bytesArray[i] = Protocol.readBytes(in);
                    }
                }
                // 判断是否是ping命令
                if (Arrays.equals(PING.getBytes(), bytesArray[0])) {
                    System.out.println("is ping cmd");
                    // 返回pong
                    Protocol.writeResponsePong(out);
                } else {
                    socket.close();
                }
            }
        }
    }
}
```




