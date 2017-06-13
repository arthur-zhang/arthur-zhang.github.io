---
layout: post
title:  "BlockingQueue带超时的批量取"
date:   2017-06-13 21:17:43
categories: 
---

我们常见的一种场景是合并上报，指定时间还没达到批量的阈值，有多少条报多少条，现在基于`BlockingQueue`实现了一种简单的方式去实习这个特性。

- Add 方法

```java
queue.offer(logItem, 10, TimeUnit.MILLISECONDS) // 如果队列已满，需要超时等待一段时间，使用此方法
queue.offer(logItem) // 如果队列已满，直接需要返回add失败，使用此方法
```

- 批量获取方法

`BlockingQueue`的批量取方法`drainTo()`不支持超时特性，但是注意到`poll()` 支持，结合这两者的特性，我们做了如下的改动

```java
public static <E> int batchGet(BlockingQueue<E> q,Collection<? super E> buffer, int numElements, long timeout, TimeUnit unit) throws InterruptedException {
    long deadline = System.nanoTime() + unit.toNanos(timeout);
    int added = 0;
    while (added < numElements) {
	    // drainTo非常高效，我们先尝试批量取，能取多少是多少，不够的poll来凑
        added += q.drainTo(buffer, numElements - added);
        if (added < numElements) {
            E e = q.poll(deadline - System.nanoTime(), TimeUnit.NANOSECONDS);
            if (e == null) {
                break;
            }
            buffer.add(e);
            added++;
        }
    }
    return added;
}
```

完整源代码如下：

```java
private static final int SIZE = 5000;
private static final int BATCH_FETCH_ITEM_COUNT = 50;
private static final int MAX_WAIT_TIMEOUT = 30;
private BlockingQueue<String> queue = new LinkedBlockingQueue<>(SIZE);
    
public boolean add(final String logItem) {
    return queue.offer(logItem);
}


public List<String> batchGet() {
    List<String> bulkData = new ArrayList<>();
    batchGet(queue, bulkData, BATCH_FETCH_ITEM_COUNT, MAX_WAIT_TIMEOUT, TimeUnit.SECONDS);
    return bulkData;
}

public static <E> int batchGet(BlockingQueue<E> q,Collection<? super E> buffer, int numElements, long timeout, TimeUnit unit) throws InterruptedException {
    long deadline = System.nanoTime() + unit.toNanos(timeout);
    int added = 0;
    while (added < numElements) {
        added += q.drainTo(buffer, numElements - added);
        if (added < numElements) {
            E e = q.poll(deadline - System.nanoTime(), TimeUnit.NANOSECONDS);
            if (e == null) {
                break;
            }
            buffer.add(e);
            added++;
        }
    }
    return added;
}
```