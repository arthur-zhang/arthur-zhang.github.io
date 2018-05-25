---
layout: post
title:  "一次ZooKeeper Connection reset的排查过程"
date:   2018-05-25 16:11:18
categories: 网络
---


今天有个长得不够帅的人，碰到了用有线网一个连接ZooKeeper每次都被断开连接的鬼故事，换无线网就没问题。报的异常如下：

```java
java.io.IOException: Connection reset by peer
        at sun.nio.ch.FileDispatcher.read0(Native Method)
        at sun.nio.ch.SocketDispatcher.read(SocketDispatcher.java:21)
        at sun.nio.ch.IOUtil.readIntoNativeBuffer(IOUtil.java:233)
        at sun.nio.ch.IOUtil.read(IOUtil.java:200)
        at sun.nio.ch.SocketChannelImpl.read(SocketChannelImpl.java:236)
        at org.apache.zookeeper.ClientCnxnSocketNIO.doIO(ClientCnxnSocketNIO.java:68)
        at org.apache.zookeeper.ClientCnxnSocketNIO.doTransport(ClientCnxnSocketNIO.java:355)
        at org.apache.zookeeper.ClientCnxn$SendThread.run(ClientCnxn.java:1068)
```

今天排查了一下，首先，为什么ZooKeeper会断开连接，wireshark在本地抓包得到的是这样的。
ZooKeeper的ip是`172.18.155.14`

![](/media/15271761140187/15271762260957.jpg)


在Zk的那台机器上`sudo lsof -nP  -p 36146| grep ESTABLISHED`查看，发现本机`172.18.155.24`与zk建立了很多连接，`wc -l`发现有`57`个

因为ZooKeeper对每个客户端有一个连接的最大大小限制，默认是`60`，具体的值可以通过配置文件`conf/zoo.cfg`进行修改


```
# the maximum number of client connections.
# increase this if you need to handle more clients
maxClientCnxns=60
```

![](/media/15271761140187/15271764562147.jpg)

那本机`172.18.155.24`要重新启动一个jar包的程序，需要建立几个连接，马上达到了60的限制，被zk拒绝掉了。

事情发生的原因就这个。

但是为什么会有这么多`ESTABLISHED`的连接呢？在本地查看都没有开启那些端口，这就非常诡异了。

后来想到了tcp中有一个东西叫`half open`，极有可能是这个原因引起的。

它说的是这么一个事情：

```
TCP/IP详解卷一第二版449页

如果在未告知另一端的情况下通信的一端关闭或终止连接，那么就认为该条TCP连接处于半开状态。
这种情况发现在通信的一方的主机崩溃、电源断掉的情况下。
只要不尝试通过半开连接来传输数据，正常工作的一端将不会检测出另外一端已经崩溃。
```

可以很容易的模拟，在机器`A`上启动ZooKeeper，虚拟机`B`上用telnet上去`10.211.55.2:2181`

```
sudo lsof -Pnl -p pid_of_zk | grep ESTABLISHED
java    23738      502   28u    IPv6 0xbb2ca6140b2f1caf        0t0        TCP 10.211.55.2:2181->10.211.55.3:59486 (ESTABLISHED)
```

然后断掉虚拟机的网络，发现`A`机器上还是`ESTABLISHED`状态

如何避免这种情况，最好的方式是业务方自己维持心跳包，一段时间对端无响应可以做对应的处理

已经存在的连接怎么处理呢？

这就需要退出一个强大的`kill tcp`连接的工具了 [killcx](http://killcx.sourceforge.net/)

centos上安装过程比较麻烦，大概的过程是：

```
yum install  libpcap
yum install libpcap-devel


shell> perl -MCPAN -e shell

cpan> install Net::RawIP
cpan> install Net::Pcap
cpan> install NetPacket::Ethernet

然后用killcx杀掉连接
```







