---
layout: post
title:  "JVM中 &lt;init&gt; &lt;clinit&gt;的区别"
date:   2016-02-23 21:17:43
categories: jvm
---
二者区别：
- &lt;init&gt;是实例的构造器方法,用于非静态变量的初始化
- &lt;clinit&gt;是类或者接口静态初始化方法

```
class X {

   static Log log = LogFactory.getLog(); // <clinit>

   private int x = 1;   // <init>

   X(){
      // <init>
   }

   static {
      // <clinit>
   }

}
```
