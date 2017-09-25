---
layout: post
title:  "Kotlin for Java Developer(4)—object"
date:   2017-07-11 23:17:43
categories: "Kotlin for Java Developer"
---
# Kotlin for Java Developer(4)—object


代码胜万言，我们来看看kotlin中object的原理

```java
object ObjectDemo {
    val name: String = "hello"
}

fun main(args: Array<String>) {
    println(ObjectDemo.name)
}
```

对应Java中的代码如下：

```
// ObjectDemo.java
public final class ObjectDemo {
   private static final String name = "hello";
   public static final ObjectDemo INSTANCE;

   public final String getName() {
      return name;
   }

   private ObjectDemo() {
      INSTANCE = (ObjectDemo)this;
      name = "hello";
   }

   static {
      new ObjectDemo();
   }
}
// ObjectDemoKt.java
public final class ObjectDemoKt {
   public static final void main(String[] args) {
      String var1 = ObjectDemo.INSTANCE.getName();
      System.out.println(var1);
   }
}
```

很明显的`eager`单例模式

