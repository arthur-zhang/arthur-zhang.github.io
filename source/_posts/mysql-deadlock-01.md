---
layout: post
title:  "一个简单加锁顺序不一致导致 MySQL update 死锁异常"
date:   2018-10-15 16:11:18
categories: MySQL
---


一个简单加锁顺序不一致导致 MySQL update 死锁异常

死锁日志如下

```
2018-10-11T01:51:52.053896Z 7977662 [Note] InnoDB:
*** (1) TRANSACTION:

TRANSACTION 854095654, ACTIVE 1 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 5 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 1
MySQL thread id 7976601, OS thread handle 139858374461184, query id 11116500072 10.111.101.39 seewo updating

update ep_dynamic_timeline set is_like=1, updated_at='2018-10-11 09:51:51.977' where dynamic_uid='0da4341bff974ccab5a58a9af1a351a1' and user_uid='8497721440b54fda903889a6270e6835' and is_like<>1

2018-10-11T01:51:52.053933Z 7977662 [Note] InnoDB: *** (1) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 427 page no 137513 n bits 288 index uni_dynamic_user of table `seewo_easi_care_broadcast`.`ep_dynamic_timeline` trx id 854095654 lock_mode X locks rec but not gap waiting

2018-10-11T01:51:52.053974Z 7977662 [Note] InnoDB: *** (2) TRANSACTION:

TRANSACTION 854095641, ACTIVE 1 sec starting index read
mysql tables in use 1, locked 1
1240 lock struct(s), heap size 123088, 110 row lock(s), undo log entries 55
MySQL thread id 7977662, OS thread handle 139858575738624, query id 11116500504 10.111.101.36 seewo updating
update ep_dynamic set created_at='2018-10-11 09:51:45', is_deleted=1, uid='0da4341bff974ccab5a58a9af1a351a1', updated_at='2018-10-11 09:51:51.921', audio_duration=0, audio_url='', content='5p6X6Zuo5qyj77yM546L5oWn546y77yM5ZGo5qKm55G2', creator_uid='758585f422b442128a9274b530826c55', like_count=0, need_reply=0, picture_url='https://store-g1.seewo.com/FuTb5kxrFCRyz_Z9TL0jldoNg0NM?imageMogr2/crop/1560x2080', reply_expired_time='2018-10-11 23:00:56', show_role=0, title='', type=1, web_content='' where id=14079229
2018-10-11T01:51:52.054007Z 7977662 [Note] InnoDB: *** (2) HOLDS THE LOCK(S):

RECORD LOCKS space id 427 page no 137513 n bits 288 index uni_dynamic_user of table `seewo_easi_care_broadcast`.`ep_dynamic_timeline` trx id 854095641 lock_mode X locks rec but not gap
2018-10-11T01:51:52.054023Z 7977662 [Note] InnoDB: *** (2) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 514 page no 594639 n bits 104 index PRIMARY of table `seewo_easi_care_broadcast`.`ep_dynamic` trx id 854095641 lock_mode X locks rec but not gap waiting
2018-10-11T01:51:52.054042Z 7977662 [Note] InnoDB: *** WE ROLL BACK TRANSACTION (1)
```

对应代码

```java
接口1
PUT /api/v1/dynamics_masaike/0da4341bff974ccab5a58a9af1a351a1/like
 @Override
    @Transactional
    public void setDynamicLikeCount(String uid, String dynamicUid, Long likeCount) {
        dynamicMapper.increaseLikeCountByUid(dynamicUid, likeCount);
        dynamicTimelineDao.setLikeState(dynamicUid, uid, true, new Date());
    }

接口2：09:51:51.911

 DELETE /api/v1/dynamics_masaike/0da4341bff974ccab5a58a9af1a351a1

    @RequestMapping(value = "/{uid}", method = RequestMethod.DELETE)
    public DataMap delete(@PathVariable String uid) {
        dynamicService.logicDeleteByUid(uid);
        return DataMap.success();
    }

    @Override
    @Transactional
    public void logicDeleteByUid(String uid) {
        Assert.hasText(uid, "uid must not be null, empty, or blank");
        dynamicDao.logicDeleteByUid(uid);
        dynamicTimelineDao.logicDeleteByDynamicUid(uid);
        classHeadlineService.deleteClassHeadlineByDynamicId(uid);
        dynamicClassDao.logicDeleteByDynamicUid(uid);
        dynamicShareMapper.removeByDynamicUid(uid);
        clearDynamicTimelineVoFromCacheByUid(uid);
    }
    

```

数据准备
表结构
```
CREATE TABLE `ep_dynamic` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `uid` varchar(32) NOT NULL COMMENT '业务uid',
  `type` int(11) NOT NULL DEFAULT '1' COMMENT '动态类型：1为公告，2为作业，3为光荣榜，4为班级成绩，5为学生成绩，6为成长手册',
  `title` varchar(256) NOT NULL DEFAULT '' COMMENT '标题',
  `content` varchar(8192) DEFAULT NULL,
  `picture_url` varchar(2048) NOT NULL DEFAULT '' COMMENT '图片地址字符串，多个地址用","分隔',
  `audio_url` varchar(128) NOT NULL DEFAULT '' COMMENT '音频url',
  `audio_duration` bigint(20) unsigned NOT NULL DEFAULT '0' COMMENT '音频时长',
  `web_content` varchar(1024) NOT NULL DEFAULT '' COMMENT '外链内容',
  `need_reply` tinyint(1) unsigned NOT NULL DEFAULT '0' COMMENT '是否需要回复，0是不需要，1是需要',
  `reply_expired_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '回复截止时间',
  `show_role` tinyint(2) unsigned NOT NULL DEFAULT '0' COMMENT '可见的角色',
  `creator_uid` varchar(32) NOT NULL DEFAULT '' COMMENT '创建者uid',
  `created_at` datetime NOT NULL COMMENT '创建时间',
  `updated_at` datetime NOT NULL COMMENT '更新时间',
  `is_deleted` tinyint(1) NOT NULL DEFAULT '0' COMMENT '删除标识',
  `like_count` bigint(20) unsigned NOT NULL DEFAULT '0' COMMENT '点赞数',
  `deadline` date NOT NULL DEFAULT '2017-06-01' COMMENT '作业截止日期',
  `isDeleted` tinyint(1) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_uid` (`uid`) USING BTREE,
  KEY `idx_type` (`type`)
) ENGINE=InnoDB AUTO_INCREMENT=5528976 DEFAULT CHARSET=utf8 COMMENT='广播站动态表'

CREATE TABLE `ep_dynamic_timeline` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `uid` varchar(32) NOT NULL COMMENT '业务uid',
  `user_uid` varchar(32) NOT NULL DEFAULT '' COMMENT '用户uid',
  `dynamic_uid` varchar(32) NOT NULL DEFAULT '' COMMENT '动态uid',
  `is_like` tinyint(1) NOT NULL DEFAULT '0' COMMENT '是否点赞',
  `is_read` tinyint(1) NOT NULL DEFAULT '0' COMMENT '是否已读',
  `read_time` datetime DEFAULT NULL COMMENT '浏览时间',
  `type` int(11) NOT NULL DEFAULT '1' COMMENT '动态类型：1为公告，2为作业，3为光荣榜，4为成长手册，5为班级成绩，6为学生成绩',
  `creator_uid` varchar(32) NOT NULL,
  `created_at` datetime NOT NULL COMMENT '创建时间',
  `updated_at` datetime NOT NULL COMMENT '更新时间',
  `is_deleted` tinyint(1) NOT NULL DEFAULT '0' COMMENT '删除标识',
  `replied` tinyint(1) unsigned NOT NULL DEFAULT '0' COMMENT '是否回复，0是否，1是',
  `replied_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '回复时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_uid` (`uid`) USING BTREE,
  UNIQUE KEY `uni_dynamic_user` (`dynamic_uid`,`user_uid`),
  KEY `idx_user_uid_type` (`user_uid`,`type`) USING BTREE,
  KEY `indx_user_id_id` (`user_uid`,`id`)
) ENGINE=InnoDB AUTO_INCREMENT=974727627 DEFAULT CHARSET=utf8 COMMENT='广播站动态时间线表'
```


#### 事务1

```
set autocommit = 0;

BEGIN;

UPDATE ep_dynamic
SET like_count = like_count + 1
WHERE UID = '0da4341bff974ccab5a58a9af1a351a1'
    AND is_deleted = 0;

UPDATE ep_dynamic_timeline
SET is_like = 1,
    updated_at = '2018-10-11 09:51:51.977'
WHERE dynamic_uid = '0da4341bff974ccab5a58a9af1a351a1'
    AND user_uid = '8497721440b54fda903889a6270e6835'
    AND is_like <> 1 ;
    
ROLLBACK;
```

#### 事务2

```
SET autocommit = 0;
BEGIN;

UPDATE ep_dynamic_timeline
SET is_deleted=1
WHERE dynamic_uid='0da4341bff974ccab5a58a9af1a351a1';

UPDATE ep_dynamic
SET created_at='2016-12-30 09:38:55',
    is_deleted=1,
    UID='0da4341bff974ccab5a58a9af1a351a1',
        updated_at='2018-10-13 13:09:38.463',
        audio_duration=3000,
        audio_url='',
        content='',
        creator_uid='ced40d39cc0a4d23a28f47c4d218e5a0',
        like_count=0,
        need_reply=0,
        picture_url='',
        reply_expired_time='2018-06-26 09:19:43',
        show_role=0,
        title='',
        TYPE=1,
             web_content=''
WHERE id=91921;


ROLLBACK;
```

可以看到是加锁顺序恰好相反导致的


|  time |  t1| t2 |
| --- | --- | --- |
| 1 | UPDATE ep_dynamic SET like_count = like_count + 1 WHERE UID = '0da4341bff974ccab5a58a9af1a351a1' |  |
| 2 |  |  UPDATE ep_dynamic_timeline SET is_deleted=1 WHERE dynamic_uid='0da4341bff974ccab5a58a9af1a351a1'|
|3||UPDATE ep_dynamic SET created_at='2016-12-30 09:38:55', is_deleted=1, UID='0da4341bff974ccab5a58a9af1a351a1', updated_at='2018-10-13 13:09:38.463', audio_duration=3000, audio_url='', content='',                                          creator_uid='ced40d39cc0a4d23a28f47c4d218e5a0', like_count=0, need_reply=0, picture_url='', reply_expired_time='2018-06-26 09:19:43', show_role=0, title='', TYPE=1, web_content='' WHERE id=91921|
|4|UPDATE ep_dynamic_timeline SET is_like = 1, updated_at = '2018-10-11 09:51:51.977' WHERE dynamic_uid = '0da4341bff974ccab5a58a9af1a351a1' AND user_uid = '8497721440b54fda903889a6270e6835' AND is_like <> 1 ;||


备注：模拟所需数据
```
INSERT INTO `ep_dynamic` (`id`, `uid`, `type`, `title`, `content`, `picture_url`, `audio_url`, `audio_duration`, `web_content`, `need_reply`, `reply_expired_time`, `show_role`, `creator_uid`, `created_at`, `updated_at`, `is_deleted`, `like_count`, `deadline`, `isDeleted`) VALUES 
	(91921,'0da4341bff974ccab5a58a9af1a351a1',1,'','','','',3000,'',0,'2018-06-26 09:19:43',0,'ced40d39cc0a4d23a28f47c4d218e5a0','2016-12-30 09:38:55','2018-10-13 13:09:38',0,0,'2017-06-01',0);
	
INSERT INTO `ep_dynamic_timeline` (`id`, `uid`, `user_uid`, `dynamic_uid`, `is_like`, `is_read`, `read_time`, `type`, `creator_uid`, `created_at`, `updated_at`, `is_deleted`, `replied`, `replied_time`) VALUES 
	(146062,'bc46ba1a2de148f39fcc78089307695c','8497721440b54fda903889a6270e6835','0da4341bff974ccab5a58a9af1a351a1',0,0,NULL,3,'','2017-01-01 17:59:37','2018-01-20 08:53:09',0,0,'2018-06-28 17:35:25');
	
SELECT *
FROM ep_dynamic
WHERE UID='0da4341bff974ccab5a58a9af1a351a1';


SELECT *
FROM ep_dynamic_timeline
WHERE dynamic_uid = "0da4341bff974ccab5a58a9af1a351a1"
    AND user_uid = '8497721440b54fda903889a6270e6835';
```




