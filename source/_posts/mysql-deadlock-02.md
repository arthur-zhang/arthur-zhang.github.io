---
layout: post
title:  "一次批量插入死锁问题分析记录"
date:   2018-11-01 16:11:18
categories: MySQL
---

线上最近出现了批量`insert`的死锁，百思不得姐。死锁记录如下

```
2018-10-26T11:04:41.759589Z 8530809 [Note] InnoDB: 
*** (1) TRANSACTION:

TRANSACTION 1202026765, ACTIVE 0 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 1
MySQL thread id 8532863, OS thread handle 139858337453824, query id 16231472122 10.111.10.143 seewo update
INSERT IGNORE INTO xx_performance_type_label_relation(label_id, performance_type_id, type, create_time)
    VALUES
      
      ('bb0394e670644168a998a93a3ed521bc', '06b96ee0bab84d71bb17bf9645d3aa54', 1, now())
     , 
      ('bb0394e670644168a998a93a3ed521bc', '27d82e2331b241e1a9c9c0a74ec21099', -1, now())
     , 
      ('bb0394e670644168a998a93a3ed521bc', '3100b5978fb24f56b327d25732a7d7a7', 1, now())
     , 
      ('bb0394e670644168a998a93a3ed521bc', '435a1e19ce6e4e5bbb84240b3b34cf03', 1, now())
     , 
      ('bb0394e670644168a998a93a3ed521bc', '447fe27199ca40e289ef2834469d9a78', 1, now())
     , 
      ('bb0394e670644168a998a93a3ed521bc', '87a52c4d00844b5bb9eb75e8fe34202a', 1, now())
     , 
      ('bb0394e670644168a998a93a3ed521bc', 'c6a0e26983bd4fae837d5ee2f4efeef8', 1, now())
2018-10-26T11:04:41.759635Z 8530809 [Note] InnoDB: *** (1) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 505 page no 9912 n bits 288 index uk_performance_type_id_label_id of table `masaike`.`xx_performance_type_label_relation` trx id 1202026765 lock_mode X locks gap before rec insert intention waiting
2018-10-26T11:04:41.759674Z 8530809 [Note] InnoDB: *** (2) TRANSACTION:

TRANSACTION 1202026764, ACTIVE 0 sec inserting
mysql tables in use 1, locked 1
3 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 1
MySQL thread id 8530809, OS thread handle 139858469242624, query id 16231472119 10.111.10.153 seewo update
INSERT IGNORE INTO xx_performance_type_label_relation(label_id, performance_type_id, type, create_time)
    VALUES
      
      ('bb0394e670644168a998a93a3ed521bc', '06b96ee0bab84d71bb17bf9645d3aa54', 1, now())
     , 
      ('bb0394e670644168a998a93a3ed521bc', '27d82e2331b241e1a9c9c0a74ec21099', -1, now())
     , 
      ('bb0394e670644168a998a93a3ed521bc', '3100b5978fb24f56b327d25732a7d7a7', 1, now())
     , 
      ('bb0394e670644168a998a93a3ed521bc', '435a1e19ce6e4e5bbb84240b3b34cf03', 1, now())
     , 
      ('bb0394e670644168a998a93a3ed521bc', '447fe27199ca40e289ef2834469d9a78', 1, now())
     , 
      ('bb0394e670644168a998a93a3ed521bc', '87a52c4d00844b5bb9eb75e8fe34202a', 1, now())
     , 
      ('bb0394e670644168a998a93a3ed521bc', 'c6a0e26983bd4fae837d5ee2f4efeef8', 1, now())
2018-10-26T11:04:41.759713Z 8530809 [Note] InnoDB: *** (2) HOLDS THE LOCK(S):

RECORD LOCKS space id 505 page no 9912 n bits 288 index uk_performance_type_id_label_id of table `masaike`.`xx_performance_type_label_relation` trx id 1202026764 lock mode S
2018-10-26T11:04:41.759753Z 8530809 [Note] InnoDB: *** (2) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 505 page no 9912 n bits 288 index uk_performance_type_id_label_id of table `masaike`.`xx_performance_type_label_relation` trx id 1202026764 lock_mode X locks gap before rec insert intention waiting
2018-10-26T11:04:41.759784Z 8530809 [Note] InnoDB: *** WE ROLL BACK TRANSACTION (2)
```

第一反应是批量insert，insert的顺序不一样导致的死锁。但是这个在这里是不成立的。原因有两点
1. 出现问题的批量插入SQL中顺序是一模一样的，在顺序一样的情况下，只会进行插入等待（implicit lock转explicit X锁）下面有实验
2. 如果是因为批量插入顺序不一致带来的死锁日志，打印的结果不是等待插入意向锁（insert intention waiting），下面有实验

现在采用一个简化的表，做实验

```
CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` varchar(5)  NOT NULL DEFAULT '',
  `b` varchar(5)  NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_name` (`a`,`b`)
) ENGINE=InnoDB;
```

### 实验 01
> 在记录不存在的情况下，两个同样顺序的批量insert同时执行，第二个会进行锁等待状态

首先`truncate t1;`

| t1 | t2 |  |
| --- | --- | --- |
| begin; | begin; |  |
|insert ignore into t1(a, b)values("1", "1");||成功|
||insert ignore into t1(a, b)values("1", "1");|锁等待状态|

可以看到目前锁的状态

```
mysql> select * from information_schema.innodb_locks;
+-------------+-------------+-----------+-----------+------------+------------+------------+-----------+----------+-----------+
| lock_id     | lock_trx_id | lock_mode | lock_type | lock_table | lock_index | lock_space | lock_page | lock_rec | lock_data |
+-------------+-------------+-----------+-----------+------------+------------+------------+-----------+----------+-----------+
| 31AE:54:4:2 | 31AE        | S         | RECORD    | `d1`.`t1`  | `uk_name`  |         54 |         4 |        2 | '1', '1'  |
| 31AD:54:4:2 | 31AD        | X         | RECORD    | `d1`.`t1`  | `uk_name`  |         54 |         4 |        2 | '1', '1'  |
+-------------+-------------+-----------+-----------+------------+------------+------------+-----------+----------+-----------+
```

在我们执行事务`t1`的insert时，没有在任何锁的断点处出现，这跟MySQL插入的原理有关系

> `insert 加的是隐式锁。什么是隐式锁？隐式锁的意思就是没有锁`

在t1插入记录时，是不加锁的。这个时候事务t1还未提交的情况下，事务t2尝试插入的时候，发现有这条记录，t2尝试获取S锁，会判定记录上的事务id是否活跃，如果活跃的话，说明事务未结束，会帮t1把它的隐式锁提升为显式锁（X锁）

源码如下
![](/media/15406248994609/15410360059022.jpg)

t2获取`S`锁的结果:`DB_LOCK_WAIT`

![](/media/15406248994609/15410362344198.jpg)

### 实验02

> 批量插入顺序不一致的导致的死锁日志不是等待插入意向锁


|  t1|t2  |  |
| --- | --- | --- |
| begin |  |  |
|insert into t1(a, b)values("1", "1");||成功|
||insert into t1(a, b)values("2", "2");|成功|
|insert into t1(a, b)values("2", "2");||t1尝试获取S锁，把t2的隐式锁提升为显式X锁，进入DB_LOCK_WAIT|
||insert into t1(a, b)values("1", "1");|t2尝试获取S锁，把t1的隐式锁提升为显式X锁，产生死锁|

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
181101  9:48:36
*** (1) TRANSACTION:
TRANSACTION 3309, ACTIVE 215 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 376, 2 row lock(s), undo log entries 2
MySQL thread id 2, OS thread handle 0x70000a845000, query id 58 localhost root update
insert into t1(a, b)values("2", "2")
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 55 page no 4 n bits 72 index `uk_name` of table `d1`.`t1` trx id 3309 lock mode S waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 1; hex 32; asc 2;;
 1: len 1; hex 32; asc 2;;
 2: len 4; hex 80000002; asc     ;;

*** (2) TRANSACTION:
TRANSACTION 330A, ACTIVE 163 sec inserting
mysql tables in use 1, locked 1
3 lock struct(s), heap size 376, 2 row lock(s), undo log entries 2
MySQL thread id 3, OS thread handle 0x70000a888000, query id 59 localhost root update
insert into t1(a, b)values("1", "1")
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 55 page no 4 n bits 72 index `uk_name` of table `d1`.`t1` trx id 330A lock_mode X locks rec but not gap
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 1; hex 32; asc 2;;
 1: len 1; hex 32; asc 2;;
 2: len 4; hex 80000002; asc     ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 55 page no 4 n bits 72 index `uk_name` of table `d1`.`t1` trx id 330A lock mode S waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 1; hex 31; asc 1;;
 1: len 1; hex 31; asc 1;;
 2: len 4; hex 80000001; asc     ;;

*** WE ROLL BACK TRANSACTION (2)
```

到目前为止，已经陷入了僵局，完全没法复现死锁的情况。看了代码，发现在insert之前有一个delete，但是delete与insert不在一个事务里面，也就是delete提交以后，才进行批量insert，真正出问题的地方在批量insert的地方。一开始就排除了delete对后面的影响，难道不在一个事务，也会有影响？

写了一个代码去模拟，有很大概率会复现

```java
fun test() {
    dao.delete() // 对应delete from
    // sleep for 10ms
    dao.insert() // 对应insert ignore
}
```

```
begin;
delete from t1 where a = '25'
commit;

begin;
INSERT ignore INTO `t1` (`a`, `b`) VALUES('25','1')
commit
```
这个代码在两个线程同时调用的时候，非常容易死锁。

后来翻遍了网上先关的死锁案例，有一个关于purge删除的过程，好像有一点启发。

> 如果标记为删除，说明事务已经提交，还没来得及 purge，这时后面的事务加`S`锁等待；

在源码中打印一些日志。
1.在`storage/innobase/row/row0ins.c`的`row_ins_set_shared_rec_lock`增加日志，可以看到对唯一索引增加`S`锁的过程

```
if (dict_index_is_clust(index)) {
    err = lock_clust_rec_read_check_and_lock(
        0, block, rec, index, offsets, LOCK_S, type, thr);
} else {
    err = lock_sec_rec_read_check_and_lock(
        0, block, rec, index, offsets, LOCK_S, type, thr);
    // 增加如下日志
    fprintf(stderr, "row_ins_set_shared_rec_lock %s %lu %d\n" , index->name, type, err);
}
```

2.在`lock_rec_enqueue_waiting`增加日志，可以看到锁等待的情况

```
static
enum db_err
lock_rec_enqueue_waiting(
{
	fprintf(stderr, "lock_rec_enqueue_waiting::::: %s %lu\n" , index->name, type_mode);
}
```

日志大概如下

```
row_ins_set_shared_rec_lock uk_name 0 9 (t1获取S锁成功）
row_ins_set_shared_rec_lock uk_name 0 9 (t2获取S锁成功）

lock_rec_enqueue_waiting::::: uk_name 2563（t1 X锁进如锁等待）
lock_rec_enqueue_waiting::::: uk_name 2563（t2 X锁进如锁等待）
```
其中2563=2048+512+3=LOCK_INSERT_INTENTION+LOCK_GAP+LOCK_X

这个过程跟非常经典的三个事务同时insert，一个回滚，剩下的两个事务一个成功，一个死锁，其实是一模一样的原理。

### 实验03 

> 三个insert，一个回滚造成的死锁

insert语句都是`insert ignore into t1(a, b)values("1", "1");`以下省略

| t1 | t2 |t3  |备注|
| --- | --- | --- |---|
| begin | begin | begin ||
|insert|||成功
||insert||把t1的隐式锁提升为X锁，t2进入进入S锁等待|
|||insert|t3进入进入S锁等待|
|rollback;|||t1回滚以后，释放X锁，t2和t3同时拿到了S锁|
||ok|deadlock|t2和t3都想拿插入意向锁X锁，造成死锁条件|
死锁日志，跟我们案例中的一模一样
```
------------------------
LATEST DETECTED DEADLOCK
------------------------
181101 23:22:59
*** (1) TRANSACTION:
TRANSACTION 5032, ACTIVE 11 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1248, 2 row lock(s), undo log entries 1
MySQL thread id 5, OS thread handle 0x70000d736000, query id 125 localhost root update
insert ignore into t1(a, b)values("1", "1")
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 56 page no 4 n bits 584 index `uk_name` of table `d1`.`t1` trx id 5032 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 139 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 3; hex 313031; asc 101;;
 1: len 3; hex 313031; asc 101;;
 2: len 4; hex 800007b1; asc     ;;

*** (2) TRANSACTION:
TRANSACTION 5033, ACTIVE 6 sec inserting
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1248, 2 row lock(s), undo log entries 1
MySQL thread id 6, OS thread handle 0x70000d779000, query id 126 localhost root update
insert ignore into t1(a, b)values("1", "1")
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 56 page no 4 n bits 584 index `uk_name` of table `d1`.`t1` trx id 5033 lock mode S locks gap before rec
Record lock, heap no 139 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 3; hex 313031; asc 101;;
 1: len 3; hex 313031; asc 101;;
 2: len 4; hex 800007b1; asc     ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 56 page no 4 n bits 584 index `uk_name` of table `d1`.`t1` trx id 5033 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 139 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 3; hex 313031; asc 101;;
 1: len 3; hex 313031; asc 101;;
 2: len 4; hex 800007b1; asc     ;;

*** WE ROLL BACK TRANSACTION (2)
```

目前来看，得到的结论是：
> 一个已提交但是未purge掉的记录会造成后续插入获取S共享锁，两个事务同时获取S锁，然后尝试获取插入意向锁，造成死锁

网上大神梳理的insert流程

* 首先对插入的间隙加插入意向锁（Insert Intension Locks）
    * 如果该间隙已被加上了 GAP 锁或 Next-Key 锁，则加锁失败进入等待；
    * 如果没有，则加锁成功，表示可以插入；

* 然后判断插入记录是否有唯一键，如果有，则进行唯一性约束检查
    * 如果不存在相同键值，则完成插入
    * 如果存在相同键值，则判断该键值是否加锁
        * 如果没有锁， 判断该记录是否被标记为删除
        * 如果标记为删除，说明事务已经提交，还没来得及 purge，这时加 S 锁等待；
        * 如果没有标记删除，则报 1062 duplicate key 错误；
    * 如果有锁，说明该记录正在处理（新增、删除或更新），且事务还未提交，加 S 锁等待；
* 插入记录并对记录加 X 记录锁；


以下为参考文档
* http://www.aneasystone.com/archives/2017/12/solving-dead-locks-three.html
* http://www.aneasystone.com/archives/2018/06/insert-locks-via-mysql-source-code.html
* http://hedengcheng.com/?p=844#_Toc378337498

