---
layout: post
title:  "null强转为原始类型的隐式空指针异常（陷阱与缺陷）"
date:   2017-01-17 21:17:43
categories: Java再入门
---


把`null`强转为原始类型会抛出空指针异常，比较容易出错

常见的场景

```java
String ip = "172.19.1.1";
Integer port = null;
// 经过一段逻辑，port，没有被赋值，调用下面的函数就会抛出空指针异常
foo(ip, port);

void foo(String ip, int port) {}
```

再比如

```java
Map<String, Object> map = new HashMap<>();
// 经过了漫长的逻辑，map被put进去了很多数据
int id = (Integer) map.get("id");  // 抛出 NullPointerException
System.out.println(id);

```

再比如：

```java
Map<Integer, Integer> numberAndCount = new HashMap<>();
int[] numbers = {3, 5, 7, 9, 11, 13, 17, 19, 2, 3, 5, 33, 12, 5};
for (int i : numbers) {
    Integer count = numberAndCount.get(i);
    numberAndCount.put(i, ++count); // NullPointerException here
}
```
