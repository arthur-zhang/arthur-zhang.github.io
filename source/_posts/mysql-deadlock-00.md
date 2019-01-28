---
layout: post
title:  "如何通过调试MySQL源码的方式来理解加锁过程和死锁"
date:   2018-10-26 16:11:18
categories: MySQL
---

这篇文章主要讲的是如何通过调试MySQL源码，知道一条SQL真正会拿哪些锁，不再抓虾，瞎猜或者何登成大神没写过的场景就不知道如何处理了。

通过好多个深夜艰难的单步调试，终于找到了一个理想的断点，可以看到所有的获取锁的过程

具体方式：使用CLion+MySQL官方源码

代码在`lock0lock.c`的`static enum db_err lock_rec_lock()` 函数中，这个函数会显示，获取锁的过程，以及获取锁成功与否的情况

以一个实际的死锁例子来看

死锁日志如下

```
*** (1) TRANSACTION:
TRANSACTION 1106, ACTIVE 50 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 376, 2 row lock(s)
MySQL thread id 5, OS thread handle 0x70000a22c000, query id 30 localhost root Updating
UPDATE xx_dynamicc_student_relation SET state = 2, update_time = now() WHERE student_uid IN ('f91792488255466aa00971fa78306c5f') AND dynamic_uid =      '11c62d70d69743118d6f382e43df4bec' AND state = 1 AND is_deleted = 0 and 1=1
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 8 page no 4 n bits 72 index `uidx_dynamic_student` of table `d1`.`xx_dynamicc_student_relation` trx id 1106 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 30; hex 313163363264373064363937343331313864366633383265343364663462; asc 11c62d70d69743118d6f382e43df4b; (total 32 bytes);
 1: len 30; hex 663931373932343838323535343636616130303937316661373833303663; asc f91792488255466aa00971fa78306c; (total 32 bytes);
 2: len 4; hex 00000001; asc     ;;

*** (2) TRANSACTION:
TRANSACTION 1107, ACTIVE 41 sec starting index read
mysql tables in use 1, locked 1
3 lock struct(s), heap size 376, 2 row lock(s)
MySQL thread id 3, OS thread handle 0x70000a26f000, query id 31 localhost root Updating
UPDATE xx_dynamicc_student_relation SET state = 2, update_time = now() WHERE student_uid IN ('f91792488255466aa00971fa78306c5f') AND dynamic_uid =      '11c62d70d69743118d6f382e43df4bec' AND state = 1 AND is_deleted = 0 and 2=2
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 8 page no 4 n bits 72 index `uidx_dynamic_student` of table `d1`.`xx_dynamicc_student_relation` trx id 1107 lock mode S
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 30; hex 313163363264373064363937343331313864366633383265343364663462; asc 11c62d70d69743118d6f382e43df4b; (total 32 bytes);
 1: len 30; hex 663931373932343838323535343636616130303937316661373833303663; asc f91792488255466aa00971fa78306c; (total 32 bytes);
 2: len 4; hex 00000001; asc     ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 8 page no 4 n bits 72 index `uidx_dynamic_student` of table `d1`.`xx_dynamicc_student_relation` trx id 1107 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 30; hex 313163363264373064363937343331313864366633383265343364663462; asc 11c62d70d69743118d6f382e43df4b; (total 32 bytes);
 1: len 30; hex 663931373932343838323535343636616130303937316661373833303663; asc f91792488255466aa00971fa78306c; (total 32 bytes);
 2: len 4; hex 00000001; asc     ;;

*** WE ROLL BACK TRANSACTION (2)
```

经过排查代码，是由下面的代码并发执行导致的

```
INSERT IGNORE INTO xx_dynamic_student_relation (id, dynamic_uid, student_uid, parent_uids, parent_names , state, create_time, update_time, is_deleted, performance_id) VALUES (NULL, '11c62d70d69743118d6f382e43df4bec', 'f91792488255466aa00971fa78306c5f', '["3d08530b61d3465eb9e9e1079c5da773"]',          '["凌秋燕"]' , 1, '2018-10-23 21:03:46.193', '2018-10-23 00:00:00.333', 0, NULL) ;

UPDATE xx_dynamic_student_relation SET state = 2, update_time = now() WHERE student_uid IN ('f91792488255466aa00971fa78306c5f') AND dynamic_uid =      '11c62d70d69743118d6f382e43df4bec' AND state = 1 AND is_deleted = 0 and 1=1;
```

表结构

```
CREATE TABLE `xx_dynamic_student_relation` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `dynamic_uid` char(32) NOT NULL DEFAULT '' ,
  `student_uid` char(32) NOT NULL DEFAULT '',
  `parent_uids` varchar(1000) DEFAULT NULL ,
  `parent_names` varchar(1000) DEFAULT NULL ,
  `state` tinyint(2) unsigned NOT NULL DEFAULT '0',
  `performance_id` varchar(1000) DEFAULT NULL,
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `update_time` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00' ,
  `is_deleted` tinyint(4) NOT NULL DEFAULT '0' ,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uidx_dynamic_student` (`dynamic_uid`,`student_uid`)
)
```

可以来实验一下唯一索引冲突的情况下，`INSERT IGNORE`在大多数情况下会加什么锁（有一些特殊情况，需要特殊讨论）

可以看到`mode=2(LOCK_S)`，对`uidx_dynamic_student`加了`LOCK_S`锁（哈哈，不要信网上的各种猜测文章，实测最靠谱）

![](/media/15405380720231/15405441529754.jpg)


那`UPDATE xx_dynamic_student_relation`会加哪些锁？
第一步： 
![](/media/15405380720231/15405513861581.jpg)

第二步：
![](/media/15405380720231/15405514551399.jpg)

可以看到，跟网上说的差不多，
对唯一索引`uidx_dynamic_student`加`X`锁（1027=LOCK_REC_NOT_GAP + LOCK_X)，然后对主键索引`PRIMARY`加`X`锁

现在就非常清楚了

|  t1 |t2  |备注  |
| --- | --- | --- |
|  INSERT IGNORE INTO | - | t1成功获得uk的S锁 DB_SUCCESS|
|  -|  INSERT IGNORE INTO|  t2成功获得uk的S锁 DB_SUCCESS|
|  UPDATE | - | t1尝试获得uk的X锁，但没有成功，处于等待状态 DB_LOCK_WAIT |
| - |  UPDATE |  t2尝试获得uk的X锁，发现死锁产生 DB_DEADLOCK|
|-|Deadlock|t2释放S锁|
|成功|- | -|

对于之前何登成大神博客里面的内容(http://hedengcheng.com/?p=771)， 可以做实验逐个验证

---

#### id主键+RC

结论： `只对主键加X锁`

`delete from table_1 where id = 10;`
![](/media/15405380720231/15405687503875.jpg)

加锁过程如下：

| 字段 | 值 | 备注 | 
| --- | --- | --- | 
| mode |1027  | LOCK_REC_NOT_GAP + LOCK_X   |  
|index|primary||   

---

#### id唯一索引+RC
结论：`对唯一索引加X锁，对主键索引加X锁`

`delete from t1 where id = 10`

![](/media/15405380720231/15405688422552.jpg)

加锁过程如下：

|序号|heap_no|变量值|锁状态|
| :---|--- | --- | --- | 
|1|7|uk_id X锁|DB_SUCCESS_LOCKED_REC|
|2|5|PRIMARY X锁|DB_SUCCESS_LOCKED_REC|
|3|7|uk_id X锁|DB_SUCCESS|

---

#### id非唯一索引+RC
结论：`对普通索引加X锁，对主键加X锁`
`delete from table_3 where id = 10`;

![](/media/15405380720231/15405689179406.jpg)

加锁过程如下：

| index | heap_no |mode| 锁类型 | 
| --- | --- |---| --- | 
|idx_id|7|1027|X|
|primary|3|1027|X|
|idx_id|7|1027|X|
|idx_id|8|1027|X|
|primary|2|1027|X|
|idx_id|8|1027|X|

#### id无索引+RC
作为练习题，不列举了
![](/media/15405380720231/15405689547518.jpg)

---

#### 备注
锁有关的常量

```
LOCK_IS 0
LOCK_IX 1
LOCK_S 2
LOCK_X 3

LOCK_REC_NOT_GAP 1024
```

相关源代码

```c
/*********************************************************************//**
Tries to lock the specified record in the mode requested. If not immediately
possible, enqueues a waiting lock request. This is a low-level function
which does NOT look at implicit locks! Checks lock compatibility within
explicit locks. This function sets a normal next-key lock, or in the case
of a page supremum record, a gap type lock.
@return	DB_SUCCESS, DB_SUCCESS_LOCKED_REC, DB_LOCK_WAIT, DB_DEADLOCK,
or DB_QUE_THR_SUSPENDED */
static
enum db_err
lock_rec_lock(
/*==========*/
	ibool			impl,	/*!< in: if TRUE, no lock is set
					if no wait is necessary: we
					assume that the caller will
					set an implicit lock */
	ulint			mode,	/*!< in: lock mode: LOCK_X or
					LOCK_S possibly ORed to either
					LOCK_GAP or LOCK_REC_NOT_GAP */
	const buf_block_t*	block,	/*!< in: buffer block containing
					the record */
	ulint			heap_no,/*!< in: heap number of record */
	dict_index_t*		index,	/*!< in: index of record */
	que_thr_t*		thr)	/*!< in: query thread */
{
	ut_ad(mutex_own(&kernel_mutex));
	ut_ad((LOCK_MODE_MASK & mode) != LOCK_S
	      || lock_table_has(thr_get_trx(thr), index->table, LOCK_IS));
	ut_ad((LOCK_MODE_MASK & mode) != LOCK_X
	      || lock_table_has(thr_get_trx(thr), index->table, LOCK_IX));
	ut_ad((LOCK_MODE_MASK & mode) == LOCK_S
	      || (LOCK_MODE_MASK & mode) == LOCK_X);
	ut_ad(mode - (LOCK_MODE_MASK & mode) == LOCK_GAP
	      || mode - (LOCK_MODE_MASK & mode) == LOCK_REC_NOT_GAP
	      || mode - (LOCK_MODE_MASK & mode) == 0);

	/* We try a simplified and faster subroutine for the most
	common cases */

	enum db_err	err;
	/* We try a simplified and faster subroutine for the most
	common cases */
	switch (lock_rec_lock_fast(impl, mode, block, heap_no, index, thr)) {
		case LOCK_REC_SUCCESS:
			err = DB_SUCCESS;
			return(err);
		case LOCK_REC_SUCCESS_CREATED:
			err = DB_SUCCESS_LOCKED_REC;
			return(err);
		case LOCK_REC_FAIL:
			err = (lock_rec_lock_slow(impl, mode, block,
									  heap_no, index, thr));
			return (err);
	}

	ut_error;
	err=DB_ERROR;
	return(err);
}
```