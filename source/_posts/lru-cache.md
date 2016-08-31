---
layout: post
title:  "LRC cache 实现"
date:   2016-08-31 21:17:43
categories: leetcode java lru
---

```
public void get() {     LinkedHashMap<String , Integer> map = new LinkedHashMap<>(10, 0.75f, true);     map.put("k1", 1);     map.put("k2", 2);     map.put("k3", 3);      System.out.println("before get: " + map);     int v2 = map.get("k2");     System.out.println("after get : " + map); }
```

输出：
```
before get: {k1=1, k2=2, k3=3}
after get : {k1=1, k3=3, k2=2}
```