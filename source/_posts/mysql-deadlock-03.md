---
layout: post
title:  "死锁案例第三波"
date:   2018-11-01 16:11:18
categories: MySQL
---

# 死锁案例第三波

前面两篇文章介绍了利用调试MySQL源码的方式来调试锁相关的信息，这里利用这个工具来解决一个比较好解决的问题，线上的表字段较多，这里简单成为了一个表

```sql
CREATE TABLE `t3` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` varchar(5) COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '',
  `b` varchar(5) COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),UNIQUE KEY `uk_a` (`a`), KEY `idx_b` (`b`) 
)
```

sql语句如下

```
t1
update t3 set b = '' where a = "1";

t2
update t3 set b = '' where b = "2";
```

我们先用之前debug的方式来看一下，这里两条语句分别加了哪些锁

### 第一条语句（通过唯一索引去更新记录）

> update t3 set b = '' where a = "1";

![](/media/15411264933560/15411276091511.jpg)

![](/media/15411264933560/15411276370193.jpg)

![](/media/15411264933560/15411276781006.jpg)

整理一下，加了3个X锁，顺序分别是

| 序号 | 索引 | 锁类型 |
| --- | :--- | --- |
| 1| uk_a |X  |
| 2| PRIMARY |X  |
|3 | idx_b |X  |

### 第二条语句

> update t3 set b = '' where b = "2";

![](/media/15411264933560/15411279273394.jpg)
![](/media/15411264933560/15411279606771.jpg)
![](/media/15411264933560/15411280062401.jpg)
整理一下，加了3个X锁，顺序分别是

| 序号 | 索引 | 锁类型 |
| --- | :--- | --- |
| 1| idx_b |X  |
| 2| PRIMARY |X  |
|3 | idx_b |X  |

两条语句从加锁顺序看起来就已经有构成死锁的条件了

![](/media/15411264933560/15411283107709.jpg)

手动是比较难模拟的，写个代码去跑一下

马上就出现了

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
181102 12:45:05
*** (1) TRANSACTION:
TRANSACTION 50AF, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 376, 2 row lock(s)
MySQL thread id 34, OS thread handle 0x70000d842000, query id 549 localhost 127.0.0.1 root Searching rows for update
update t3 set b = '' where b = "2"
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 67 page no 3 n bits 72 index `PRIMARY` of table `d1`.`t3` trx id 50AF lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 0000000050ae; asc     P ;;
 2: len 7; hex 03000001341003; asc     4  ;;
 3: len 1; hex 31; asc 1;;
 4: len 0; hex ; asc ;;

*** (2) TRANSACTION:
TRANSACTION 50AE, ACTIVE 0 sec updating or deleting
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1248, 3 row lock(s), undo log entries 1
MySQL thread id 35, OS thread handle 0x70000d885000, query id 548 localhost 127.0.0.1 root Updating
update t3 set b = '' where a = "1"
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 67 page no 3 n bits 72 index `PRIMARY` of table `d1`.`t3` trx id 50AE lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 0000000050ae; asc     P ;;
 2: len 7; hex 03000001341003; asc     4  ;;
 3: len 1; hex 31; asc 1;;
 4: len 0; hex ; asc ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 67 page no 5 n bits 72 index `idx_b` of table `d1`.`t3` trx id 50AE lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 1; hex 32; asc 2;;
 1: len 4; hex 80000001; asc     ;;

*** WE ROLL BACK TRANSACTION (1)
```

跟我们线上的死锁日志一模一样