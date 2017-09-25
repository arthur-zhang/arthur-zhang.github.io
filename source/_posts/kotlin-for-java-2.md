---
layout: post
title:  "Kotlin for Java Developer(2)—when语法"
date:   2017-06-21 21:17:43
categories: "Kotlin for Java Developer"
---

kotlin中没有 `switch case` 语法，取而代之的是引入了`when`语法，`when`语法会视情况，把语句翻译成Java概念中的`if-else`  `switch-case`  `三目运算符`

下面枚举了三种情况

- if-else

```java
fun checkValue(result: Any) {
    when (result) {
        "Value" -> {
            println("It's a value")
            println("Second statement")
        }
        is String -> {
            println("Excellent")
        }
        else -> {
            println("It came to this?")
        }
    }
}
```

反编译class文件以后的Java代码

```java
    public static final void checkValue(@NotNull final Object result) {
        Intrinsics.checkParameterIsNotNull(result, "result");
        if (Intrinsics.areEqual(result, (Object)"Value")) {
            System.out.println((Object)"It's a value");
            System.out.println((Object)"Second statement");
        }
        else if (result instanceof String) {
            System.out.println((Object)"Excellent");
        }
        else {
            System.out.println((Object)"It came to this?");
        }
    }
```

就是一个高级版`if-else`语法糖

- switch-case

```java
fun check(x: Int) {
    when (x) {
        1 -> print("x == 1")
        2 -> print("x == 2")
        else -> { // Note the block
            print("x is neither 1 nor 2")
        }
    }
}
```

对应于

```java
    public static final void check(final int x) {
        switch (x) {
            case 1: {
                System.out.print((Object)"x == 1");
                break;
            }
            case 2: {
                System.out.print((Object)"x == 2");
                break;
            }
            default: {
                System.out.print((Object)"x is neither 1 nor 2");
                break;
            }
        }
    }
```


- 三目表达式

```java
fun describe(obj: Any): String =
        when (obj) {
            1 -> "One"
            "Hello" -> "Greeting"
            is Long -> "Long"
            !is String -> "Not a string"
            else -> "Unknown"
        }
```

对应于

```java
    public static final String describe(final Object obj) {
        return Intrinsics.areEqual(obj, (Object)1) ? "One" : (Intrinsics.areEqual(obj, (Object)"Hello") ? "Greeting" : ((obj instanceof Long) ? "Long" : ((obj instanceof String) ? "Unknown" : "Not a string")));
    }
```

由此可见kotlin的语法更加具有表现力