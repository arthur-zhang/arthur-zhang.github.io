---
layout: post
title:  "Java int long equals的坑"
date:   2016-09-01 21:17:43
categories: 
---
#### int long equals的坑

```
Integer s = new Integer(9);
Integer t = new Integer(9);
Long u = new Long(9);
System.out.println(s == u); // 编译错误
System.out.println(s == t); // false,指向不同对象
System.out.println(s.equals(u)); // false, Integer的equals方法首先判断是不是Integer类型,如果不是,直接return false

System.out.println(s.equals(new Integer(9))); // true

System.out.println(s.equals(9)); // true
System.out.println(u.equals(9)); // false
System.out.println(u.equals(9L)); // true

System.out.println(u == 9); // true
System.out.println(u == 9L); // true
```