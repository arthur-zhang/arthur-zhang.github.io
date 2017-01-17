---
layout: post
title:  "重新认识Arrays.asList（冷知识）"
date:   2016-12-24 21:17:43
categories: Java再入门
---


我们在把数组转list时，最常看到的实用方式是`Arrays.asList`

比如

```java
String[] receivers  = new String [] {
                "tom","jerry"
        };
List<String> list =  Arrays.asList(receivers);
```

然后看下`Arrays.asList`的源码

```java
public static <T> List<T> asList(T... a) {
    return new ArrayList<>(a);
}
```

是不是跟你想的一样，这个方法把`array`转为了`java.util.ArrayList`？

我都这么说了，肯定不是的啦，不信我们来测试一下。

```java
public static void main(String[] args) {
    // 正常
    List<String> foo = foo();

    // 正常
    List<String> bar = bar();

    // 正常
    java.util.ArrayList<String> foo1 = (java.util.ArrayList<String>) foo();

    // 挂掉 java.lang.ClassCastException: java.util.Arrays$ArrayList cannot be cast to java.util.ArrayList
    java.util.ArrayList<String> bar1 = (java.util.ArrayList<String>) bar();

    System.out.println(foo().getClass()); // class java.util.ArrayList
    System.out.println(bar().getClass()); // class java.util.Arrays$ArrayList

}

public static List<String> foo() {
    List<String> list = new ArrayList<>();
    list.add("tom");
    return list;
}
public static List<String> bar() {
    return Arrays.asList("tom");
}
```

现在我们知道了`Arrays.asList`返回的并不是`java.util.ArrayList`,而是Arrays的静态内部类`Arrays.ArrayList`，源码如下：

```
public class Arrays {
    public static <T> List<T> asList(T... a) {
        return new ArrayList<>(a);
    }
    private static class ArrayList<E> extends AbstractList<E> {
    }
}
```

可能你又要想了，有什么关系，不影响我使用。加入你想再add或删除一个元素，那么就悲剧了


```java
List<String> bar = Arrays.asList("hello");
bar.add("world"); // java.lang.UnsupportedOperationException
```

因为Arrays.ArrayList的内部实现是一个`固定大小`的数组，正如它文档所说

> Returns a fixed-size list backed by the specified array

那么要把array转换为一个可变大小的list，要怎么处理？

```java
ArrayList<String> arrayList = new ArrayList<String>(Arrays.asList(arr));
```







