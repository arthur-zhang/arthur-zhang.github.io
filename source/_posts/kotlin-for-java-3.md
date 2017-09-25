---
layout: post
title:  "Kotlin for Java Developer(3)—local return与inline"
date:   2017-06-21 23:17:43
categories: "Kotlin for Java Developer"
---
# Kotlin for Java Developer(3)—local return与inline


- local return其实说的是个比较简单的事情，在lambda中return，到底是应该退出`外层调用函数`，还是退出`lambda函数`的问题

以下面这段代码为例

```java
fun test3() {
    val nums = 1..100
    nums.forEach {
        if (it % 5 == 0) {
            return
        }
    }
    println("reach here") // 不被打印
}
```

那到底最后面的`”reach here”`会不会被打印呢？，答案是否定的
如果我们只是想退出`foreach`，还是有办法的，就是现在要说的`local return`，

示例代码如下

```java
fun test3() {
    val nums = 1..100
    nums.forEach MyLabel@ {
        if (it % 5 == 0) {
            return@MyLabel
        }
    }
    println("reach here") // 被打印
}
```

当然lambda函数名也是也是可以用来当return label的

```java
fun test3() {
    val nums = 1..100
    nums.forEach {
        if (it % 5 == 0) {
            return@forEach
        }
    }
    println("reach here") // 被打印
}
```

当然换成匿名函数，是没有这个问题的，因为return退出的是匿名函数本身

```java
fun test4() {
    val nums = 1..100
    nums.forEach(
            fun(x): Unit {
                if (x % 5 == 0) {
                    return
                }
            })
    println("reach here") // 被打印
}
```

为什么lambda return跳出的是外层函数而不是lambda函数自身呢？

我们先来看一下`forEach`高阶函数的实现

```java
_Collections.kt

public inline fun <T> Iterable<T>.forEach(action: (T) -> Unit): Unit {
    for (element in this) action(element)
}
```

我们有注意到新的关键字`inline`

```java
fun sayHello() {
    val a = 0
    val b = 1
    var c = a + b
    println(c)
}

fun main(args: Array<String>) {
    print("start")
    sayHello()
    print("end")
}
```

```java
public static final void sayHello() {
        final int a = 0;
        final int b = 1;
        final int c = a + b;
        System.out.println(c);
    }
    
    public static final void main(@NotNull final String[] args) {
        Intrinsics.checkParameterIsNotNull((Object)args, "args");
        System.out.print((Object)"start");
        sayHello();
        System.out.print((Object)"end");
    }
```

加了inline以后
```java
inline fun sayHello() {
    val a = 0
    val b = 1
    var c = a + b
    println(c)
}
```

```java
    public static final void main(@NotNull final String[] args) {
        Intrinsics.checkParameterIsNotNull((Object)args, "args");
        System.out.print((Object)"start");

        final int a = 0;
        final int b = 1;
        final int c = a + b;

        System.out.println(c);
        System.out.print((Object)"end");
    }
```

在kotlin中，函数就是对象，当你调用某个函数的时候，就会创建相关的对象。
这就是空间上的开销
当你去调用某个函数的时候，虚拟机会去找到你调用函数的位置。然后执行函数，然后再回到你调用的初始位置。
这就是时间上的开销
ok，现在，我直接把调用的函数里面的代码放到我调用的地方，省去寻找调用函数的位置的时间。

既然如此，如果函数特别长，其实不适合内联

在kotlin中，return 只可以用在`有名字的函数`，或者`匿名函数中`，使得该函数执行完毕。
而针对lambda表达式，你不能直接使用return
有一个例外，就是lambda是个内联函数