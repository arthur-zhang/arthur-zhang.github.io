---
layout: post
title:  "BloomFilter基础"
date:   2018-05-03 16:11:18
categories: 数据结构
---


# Bloom过滤器

# 它要解决什么问题

假设我们有一个`1000W`行的手机号黑名单数据库，现在有一个需求，给一批用户发送短信推送，要求用户手机号不在黑名单数据库里面。

如何判断`A`在集合`B`中？

![](/media/15252661702341/15252706234780.jpg)

有哪些数据结构可以表示一个`set`？

- Self-balancing binary search trees 
- Tries 
- Hash tables 
- Simple arrays 
- Linked lists of the entries

上面的数据结构大都要求把实际数据存储到数据节点中，有些不够快、有些太大了

`Bloom过滤器`粉墨登场

核心原理是： 

> ⼏个hash函数, 计算出n个结果, 并在`ﬁlter`数据中将对应的位置为1

![](/media/15252661702341/15252667059379.jpg)

# 数据查询
查询是否在结果集中, 要⽤同样的hash算法, ⽐对是否所有结果位都为`1`
- 如果返回`1`，数据`可能`在集合中
- 如果返回`0`，数据`绝逼不可能`在集合中

![](/media/15252661702341/15252702421880.jpg)


# 使用场景
看到下面这些，你就可以放心大胆的用啦

- `Google BigTable`, `Apache HBase` 以及 `Apache Cassandra` 使⽤bloomﬁlter检查字段是否存在.
- `Google Chrome` 浏览器, 使⽤bloomﬁlter检查恶意⽹址.
- `Squid` 代理服务器使⽤bloomﬁlter检查缓存是否存在.
- `Bitcoin` ⽐特币使⽤bloomﬁlter提升数据钱包的同步效率.
- `MongoDB` 使⽤bloomﬁlter检查数据是否位于某个节点.
- `Exim` 使⽤bloomﬁlter防范⾼频次的恶意邮件. 
- 许多搜索引擎⼚商使⽤bloomﬁlter帮助爬⾍防⽌多次爬取同⼀个url地址.

# 使用的局限性
不支持删除操作，如要支持，需要使用`counting bloomﬁlter ﬁlters`的变种

# 误判的概率
- `m`是数组的位数
- `k`是哈希函数的个数
- `n`是要插入的原始的个数

下面是一个神奇的公式

![](/media/15252661702341/15253106809572.jpg)


合理的`k`值
![](/media/15252661702341/15253103089937.jpg)

![](/media/15252661702341/15253218057406.jpg)


![](/media/15252661702341/15253217865131.jpg)


# 实际演示
http://billmill.org/bloomfilter-tutorial/






