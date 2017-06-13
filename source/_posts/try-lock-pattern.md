---
layout: post
title:  "Lock在try中的正确姿势"
date:   2017-03-10 21:17:43
categories: Java再入门
---

在使用Java Lock时，有两种使用的可能

```java
1. 
Lock l = ...;
l.lock();
try {
    // access the resource protected by this lock
} finally {
    l.unlock();
}

2.
Lock l = ...;
try {
    l.lock();
    // access the resource protected by this lock
} finally {
    l.unlock();
}
```

区别就在于 lock 放在 try 里面，还是放在 try 外面
我们推荐的是第一种方式，**把 lock 放在 try 外面**
理由是：第二种方式中，如果 l.lock() 抛出了异常，未能 lock 成功，依然会调用 unlock 方法，没有任何意义。