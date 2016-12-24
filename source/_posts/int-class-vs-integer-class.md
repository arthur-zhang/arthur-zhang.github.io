---
layout: post
title:  "int.class和Integer.class"
date:   2016-12-24 21:17:43
categories: Java再入门
---


下面的代码输出什么？

(⊙o⊙)…先不要怀疑`int.class`这样写是不是有语法错误。。。

```java      
    Class<Integer>  a = int.class;
    Class<Integer>  b = Integer.class;
    System.out.println(a == b);
    
    输出 false
```

下面来分析一下为什么。

先来看字节码


```java
 Object a = int.class;
 Object b = Integer.class;

int.class
 0: getstatic     #2   // Field java/lang/Integer.TYPE:Ljava/lang/Class;
 
Integer.class
4: ldc           #3   // class java/lang/Integer
```
---




可见

`int.class=Integer.TYPE`

实际上是`Integer`类的一个静态常量`TYPE`，最终的值是`Class.getPrimitiveClass("int")`

```java
    /**
     * The {@code Class} instance representing the primitive type
     * {@code int}.
     *
     * @since   JDK1.1
     */
    public static final Class<Integer>  TYPE = (Class<Integer>) Class.getPrimitiveClass("int");

```

`Integer.class`这个大家就比较清楚了，是`java.lang.Integer`的class对象，实际上代表 `Class.forName("java.lang.Integer")`

你可能会想，知道这个有毛用？这么偏门的知识。

在反射中，搞错了这两个，你可能要调试n长时间，找不到方法等，不要问我是怎么知道的。
