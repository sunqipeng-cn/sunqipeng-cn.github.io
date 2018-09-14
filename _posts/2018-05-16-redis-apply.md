---
layout: post
title:  "Redis中各数据类型的应用场景"
date:   2018-05-16 23:06:05
categories: 分布式
tags: redis
author: sqp
---

* content
{:toc}

# Redis中各数据类型的应用场景

## String

利用INCR函数的原子性做计数器  
>商品维度计数（喜欢数，评论数，鉴定数，浏览数）  
用户维度计数（动态数、关注数、粉丝数、喜欢商品数、发帖数，微博数）  

使用redis计数器防止并发请求--INCR key  
![avator](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/71039317.jpg)

限速器模式  
![avatar](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/2453472.jpg)
![avator](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/87767866.jpg)

实现用户上线次数统计
BITCOUNT key [start] [end]  
计算给定字符串中，被设置为 1 的比特位的数量。  
一般情况下，给定的整个字符串都会被进行计数，通过指定额外的 start 或 end 参数，可以让计数只在特定的位上进行。  
start 和 end 参数的设置和 GETRANGE 命令类似，都可以使用负数值：比如 -1 表示最后一个位，而 -2 表示倒数第二个位，以此类推。  
不存在的 key 被当成是空字符串来处理，因此对一个不存在的 key 进行 BITCOUNT 操作，结果为 0 。  

假设现在我们希望记录自己网站上的用户的上线频率，比如说，计算用户 A 上线了多少天，用户 B 上线了多少天，诸如此类，以此作为数据，从而决定让哪些用户参加 beta 测试等活动 —— 这个模式可以使用 SETBIT 和 BITCOUNT 来实现。  
举个例子，如果今天是网站上线的第 100 天，而用户 peter 在今天阅览过网站，那么执行命令 SETBIT peter 100 1 ；如果明天 peter 也继续阅览网站，那么执行命令 SETBIT peter 101 1 ，以此类推。  
当要计算 peter 总共以来的上线次数时，就使用 BITCOUNT 命令：执行 BITCOUNT peter ，得出的结果就是 peter 上线的总天数。  

``` shell
redis> BITCOUNT bits
(integer) 0
redis> SETBIT bits 0 1
(integer) 0
redis> BITCOUNT bits
(integer) 1
redis> SETBIT bits 3 1
(integer) 0
redis> BITCOUNT bits
(integer) 2
```

性能与内存  
前面的上线次数统计例子，即使运行 10 年，占用的空间也只是每个用户 10*365 比特位(bit)，也即是每个用户 456 字节。对于这种大小的数据来说， BITCOUNT 的处理速度就像 GET 和 INCR 这种 O(1) 复杂度的操作一样快。

## Hash

存储部分变更数据-如用户信息等  
![avatar](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/50109879.jpg)
![avatar](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/45326923.jpg)
![avatar](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/72643507.jpg)

优势:  
- 节省内存
- 不会带来序列化和返序列化问题
- 不会带来并发修改控制的问题

>注意: hgetall的性能问题,尤其是和管道一起使用  

## List

取最新N个数据的操作  
![avator](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/31212177.jpg)
![avator](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/61092191.jpg)

比如twitter的关注列表，粉丝列表等都可以用Redis的list结构来实现。  
消息队列：可以利用Lists的PUSH操作，将任务存在Lists中，然后工作线程再用POP操作将任务取出进行执行。  
如果入队端一直在塞数据，而出队端没有消费数据，或者是入队的频率大而多，出队端的消费频率慢会导致内存暴涨。  
![avator](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/58076245.jpg)

## Zset

数据自动排重 比如：在微博应用中，可以将一个用户所有的关注人存在一个集合中，将其所有粉丝存在一个集合。  
Redis还为集合提供了求交集、并集、差集等操作，可以非常方便的实现如共同关注、共同喜好、二度好友等功能，对上面的所有集合操作，你还可以使用不同的命令选择将结果返回给客户端还是存集到一个新的集合中。  
![avator](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/72448384.jpg)
![avator](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/25292352.jpg)
![avator](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/15436004.jpg)
![avator](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/18923117.jpg)

## SortedSet

和set相比，sorted set增加了一个权重参数score，使得集合中的元素能够按score进行有序排列.  

1. 比如一个存储全班同学成绩的sorted set，其集合value可以是同学的学号，而score就可以是其考试得分，这样在数据插入集合的时候，就已经进行了天然的排序。
2. 可以用sorted set来做带权重的队列，比如普通消息的score为1，重要消息的score为2，然后工作线程可以选择按score的倒序来获取工作任务。让重要的任务优先执行。
3. 排行榜应用：

![avator](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/60826471.jpg)
![avator](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/63621016.jpg)

``` shell
zadd news 201610022301 '{"title" : "new1", "time": 201610022301}'  
zadd news 201610022302 '{"title" : "new2", "time": 201610022302}'  
zadd news 201610022303 '{"title" : "new3", "time": 201610022303}'  
zadd news 201610022304 '{"title" : "new4", "time": 201610022304}'
zadd news 201610022305 '{"title" : "new5", "time": 201610022305}'
```

实际场景中new1、news2…等等都应该是一个json对象,包含新闻标题、时间、作者、类型、图片URL…等等信息。  
然后当用户请求新闻时,我们会使用这个命令查询出最新几条记录:  

``` shell
ZREVRANGEBYSCORE news +inf -inf LIMIT 0 2
```

上面的例子中,我们从news频道查询出来了2条最新记录。如果用户翻页,我们会使用最新记录中时间最大的记录做为参数进行分页。依次查询第二页,第三页。

``` shell
ZREVRANGEBYSCORE news +inf -inf LIMIT 2 2
ZREVRANGEBYSCORE news +inf -inf LIMIT 4 2
```

![avator](http://pf1gfkwtz.bkt.clouddn.com/18-9-14/2409062.jpg)

