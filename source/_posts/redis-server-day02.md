---
layout: post
title:  "一起来写个redis server（day02）"
date:   2017-03-21 21:17:43
categories: 一起来写个redis-server
---



今天的目标：学习原生`NIO1`，最后用nio框架`netty`实现`redis`的`ping`命令解析和响应

## 1. NIO基础

### 1.1 实现一个阻塞的echo server，代码参考《Pro NIO2》

```java
public class BlockingEchoServer {
    public static void main(String[] args) {
        try {
            ServerSocketChannel channel = ServerSocketChannel.open();
            channel.bind(new InetSocketAddress("localhost", 6397));
            channel.configureBlocking(true); // 设置channel为阻塞模式
            
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            while (true) {
                SocketChannel socketChannel = channel.accept();
                System.out.println("Incoming connection from: " + socketChannel.getRemoteAddress());
                
                while (socketChannel.read(buffer) != -1) {
                    // flip将Buffer从写模式切换为读模式
                    buffer.flip();
                    socketChannel.write(buffer);
                    
                    // 准备下一次写入
                    if (buffer.hasRemaining()) {
                    // 如果buffer中还有未读数据，compact方法将所有未读数据拷贝到Buffer起始处，然后将position设置为最后一个未读元素后面
                        buffer.compact(); 
                    } else {
                        buffer.clear();
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
我们用telnet来测试一下
```
telnet 127.0.0.1  6397
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello
hello
```

### 1.2 实现一个非阻塞的echo server
原生NIO1代码比较啰嗦，比较长，详情参见github，NIO的学习，后面会有专门的文章来说明。

## 2. 拥抱netty，来实现redis的ping命令

netty使用的基本套路是这样的：

```java
public static void main(String[] args) throws InterruptedException {
    // 创建EventLoopGroup
    EventLoopGroup eventLoopGroup = new NioEventLoopGroup();
    try {
        // 创建ServerBootstrap
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(eventLoopGroup)
                // 注明使用的Nio channel
                .channel(NioServerSocketChannel.class)
                // 指定端口
                .localAddress(new InetSocketAddress(9736))
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel socketChannel) throws Exception {
                        socketChannel.pipeline().addLast(new CommandHandler());
                    }
                });
        ChannelFuture future = bootstrap.bind().sync();
        future.channel().closeFuture().sync();
    } finally {
        eventLoopGroup.shutdownGracefully().sync();
    }
}
```

具体的CommandHandler实现如下：
```java
    private static class CommandHandler extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            ByteBuf buf = (ByteBuf) msg;
            System.out.println(buf.toString(CharsetUtil.UTF_8));
            buf.resetReaderIndex();
            int read = buf.readByte();
            if (read == '*') {
                // 获取参数个数
                long numberArgs = Protocol.readLong(buf);
                byte[][] bytesArray = new byte[(int) numberArgs][];
                for (int i = 0; i < numberArgs; i++) {
                    if (buf.readByte() == '$') {
                        // 获取命令长度
                        int size = (int) Protocol.readLong(buf);
                        bytesArray[i] = new byte[size];
                        buf.readBytes(bytesArray[i]);
                    }
                }
                // 判断是否是ping命令
                if (Arrays.equals(PING.getBytes(), bytesArray[0])) {
                    System.out.println("is ping cmd");
                    // 拼接response
                    ctx.write(Unpooled.wrappedBuffer("+".getBytes()));
                    ctx.write(Unpooled.wrappedBuffer("pong".getBytes()));
                    ctx.write(Unpooled.wrappedBuffer(CRLF));
                    ctx.flush();
                } else {
                    ctx.write(Unpooled.wrappedBuffer("+".getBytes()));
                    ctx.write(Unpooled.wrappedBuffer(CRLF));
                    ctx.flush();
                }
            }
        }
        
        @Override
        public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        }
        
        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            cause.printStackTrace();
            ctx.close();
        }
    }
```
同样，我们用`redis-cli`来测试，可以看到解析成功，且返回了正确的命令格式

```shell
redis-cli -p 9736
127.0.0.1:9736> ping
pong
```







