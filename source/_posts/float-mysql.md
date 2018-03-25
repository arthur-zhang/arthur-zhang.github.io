---
layout: post
title:  "MySQL float"
date:   2017-09-07 21:17:43
categories: MySQL
---

先来看几个吃惊的现象

```
CREATE TABLE t1 (
  c1 float(15,2) DEFAULT NULL
)
insert into t1 values(123456789);
select * from t1

t1
123456792.00
```

妈的，居然连整数都不对了，简直颠覆

```
0100 1100 1110 1011 0111 1001 1010 0011

0100 1100 1110 1011 0111 1001 1010 0011
```



