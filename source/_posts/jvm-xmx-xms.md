---
layout: post
title:  "JVM的Xmx Xms分析"
date:   2017-09-07 21:17:43
categories: Jvm
---


这个文章试图解答以下两个困扰许久的问题

- 为什么JVM进程物理内存远大于Xmx设置的值，比如设置了-Xmx512m，用top看到 JVM 实际内存占用超过1G？
- 设置-Xms到底有什么用，设置-Xms=2048m，用top看到JVM占用实际很小？

# JVM进程物理内存远大于Xmx
第一个问题，其实很好理解，Xmx指定的是最大堆的大小，也就是我们熟悉的新生代和老生代的最大值，不包括其他（持久代、JIT CodeCache，堆外内存、JNI code、Metaspace、线程栈）

# -Xms到底有什么用

第二个问题，会稍微复杂一点


做个简单的实验：

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello World!");
        try {
            Thread.sleep(1000000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```


关注两个输出

```
java -cp . -Xms512m -Xmx512m -XX:NativeMemoryTracking=detail Hello
```

#### top输出

```
top -p $pid
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 4527 ya        20   0 2808408  39608  10576 S   0.0  2.1   0:00.98 java
```

RES值其实很小

#### jcmd输出

```
jcmd  5641 VM.native_memory summary

Total: reserved=1876589KB, committed=577621KB
-                 Java Heap (reserved=524288KB, committed=524288KB)
                            (mmap: reserved=524288KB, committed=524288KB)

-                     Class (reserved=1059949KB, committed=8045KB)
                            (classes #389)
                            (malloc=3181KB #137)
                            (mmap: reserved=1056768KB, committed=4864KB)

-                    Thread (reserved=12388KB, committed=12388KB)
                            (thread #13)
                            (stack: reserved=12336KB, committed=12336KB)
                            (malloc=38KB #64)
                            (arena=14KB #24)

-                      Code (reserved=249632KB, committed=2568KB)
                            (malloc=32KB #297)
                            (mmap: reserved=249600KB, committed=2536KB)

-                        GC (reserved=22625KB, committed=22625KB)
                            (malloc=3465KB #111)
                            (mmap: reserved=19160KB, committed=19160KB)

-                  Compiler (reserved=132KB, committed=132KB)
                            (malloc=1KB #22)
                            (arena=131KB #3)

-                  Internal (reserved=5904KB, committed=5904KB)
                            (malloc=5872KB #1325)
                            (mmap: reserved=32KB, committed=32KB)

-                    Symbol (reserved=1356KB, committed=1356KB)
                            (malloc=900KB #66)
                            (arena=456KB #1)

-    Native Memory Tracking (reserved=140KB, committed=140KB)
                            (malloc=86KB #1367)
                            (tracking overhead=54KB)

-               Arena Chunk (reserved=174KB, committed=174KB)
                            (malloc=174KB)
```

上面的输出，也可以解释第一个问题，除了heap的占用，JVM还有很多其他的开销，比如线程、GC等上面所说的

我们来关注一个重要的信息
                      
```
Java Heap (reserved=524288KB, committed=524288KB)
    (mmap: reserved=524288KB, committed=524288KB)
```

我们看到`reserved`和`committed`都是`512M`

我们改一下启动参数，把Xms改小到128m

```shell
java -cp . -Xms128m -Xmx512m -XX:NativeMemoryTracking=detail Hello
```

重新用jcmd来查看一下

```
Total: reserved=1873968KB, committed=180508KB
-                 Java Heap (reserved=524288KB, committed=131072KB)
                            (mmap: reserved=524288KB, committed=131072KB)
```
看到reserved的等于Xmx，committed的等于Xms

我们来看一下原理，

> 为了让资源得到最大合理利用，OS在物理内存之上搞一层虚拟地址，同一台机器上每个进程可访问的虚拟地址空间大小都是一样的，为了屏蔽掉复杂的到物理内存的映射，该工作os直接做了，当需要物理内存的时候，当前虚拟地址又没有映射到物理内存上的时候，就会发生缺页中断，由内核去为之准备一块物理内存，所以即使我们分配了一块1G的虚拟内存，物理内存上不一定有一块1G的空间与之对应
（抄袭于其它博客）

虚拟地址内存的几个状态：
- Free：不解释了
- Reserved：这段地址空间被保留，以便未来进程需要的时候，可以获取到，这部分内存只有在committed以后才可以被访问到
- Committed：可以被进程访问到的内存，意味着在paging file里分配了这么多的page frames，Committed内存只要在进程使用的时候才会被真正加载到主内存，也被称为 on-demand paging

比如我们设置 -Xms512m -Xmx2048m
- Xmx：
JVM告诉操作系统，我的堆可能需要多达2G的内存，你要给我留着，不同意的话，我是有脾气的，我就不启动了。操作系统如果爱你的话，就会承诺，当JVM需要增加堆而尝试分配额外内存的时，这些内存是可以获取到的
- Xms：
JVM启动时分配的初始内存，但是要注意这是committed虚拟地址空间，只有用到的时候，才会变成物理内存

为什么Oracle推荐设置`Xmx`和`Xms`为相等：

主要是为了避免Xms用完GC以后，需要重新向操作系统申请内存，设置为一样，就省去了这个过程


参考链接

https://blogs.oracle.com/buck/repost%3a-why-is-my-jvm-process-larger-than-max-heap-size
https://plumbr.eu/blog/memory-leaks/why-does-my-java-process-consume-more-memory-than-xmx     
https://www.ibm.com/developerworks/library/j-memusage/
http://lovestblog.cn/blog/2015/08/21/rssxmx/


