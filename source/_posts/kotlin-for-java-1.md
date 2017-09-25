---
layout: post
title:  "Kotlin for Java Developer(1)—Main函数"
date:   2017-06-20 21:17:43
categories: "Kotlin for Java Developer"
---

用Intellij Idea创建一个kotlin项目，新建一个MainDemo.kt文件，输入`main`回车，就自动回生成main入口函数

```java
MainDemo.kt

package com.cvte.kt.basic
fun main(args: Array<String>) {
    print("Hello World")
}
```

运行一下，就会输出`Hello World`

我们来对比一下，Java版本的`Hello World`

```java
MainDemoJava.java
public class MainDemoJava {
    public static void main(String[] args) {
        System.out.println("Hello World");
    }
}
```

我们发现，kotlin版本的main函数没有包裹在类中，至少现在看起来如此。
那它是如何做到可以编译运行的呢？
我们看一下MainDemo.kt编译后的输出文件

```
-rw-r--r--  1 Arthur  staff   1.0K  6 20 08:50 MainDemoKt.class
```

发现会生成`一个文件名+Kt+.class`的class文件，反编译一下看看

```java
public final class MainDemoKt
{
    public static final void main(@NotNull final String[] args) {
        Intrinsics.checkParameterIsNotNull((Object)args, "args");
        System.out.print("Hello World");
    }
}
```

那我们看到，kotlin其实就是一个语法糖嘛，没什么神奇的，但是又超级好用，后面会有一个系列讲，kotlin到底有哪些好用的语法，以及是如何是如何实现的
