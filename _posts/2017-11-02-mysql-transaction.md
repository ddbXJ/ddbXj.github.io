---
layout:     post
title:      "mysql锁和事务分析"
subtitle:   "mysql,spring transaction"
date:       2017-11-02
author:     "lee"
header-img: ""
---


业务里经常会碰到,查询一条数据,然后对这条数据`update`,如果业务方法里有异常,则方法里所有数据回滚.很容易想到的是`Spring框架`封装好的事务,加个`@Transactional`注解就完事.但是如果是单纯的`udpate`一条数据还好说,但对于`select for update`的场景,事务注解就不管用了.

对应的伪代码如下
```java
@Transactional
public void methodA() {
    ...
    //1. select xxx where ...
    //2. update xxx where ...
    ...
}

```

直接加`@Transactional`注解,其实就是`Spring框架`封装的事务管理器,在mysql里开了一个事务*(InnoDB引擎,默认事务隔离级别是`可重复读`)*,然后根据条件,先`select`数据,再`update`数据
 
这个时候,第一句的`select`并不是**排他锁**(x锁),所以如果有并发更新同一条记录的请求的话,这个`select`在别的事务里面照样能够读到,然后在`update`操作的时候,会出现争用,最后就导致先抢到锁的线程的更新丢失,所以在这样的场景,简单加个事务注解是不能解决问题的.

有一种不推荐的解决方案,也就是对读操作就加锁,这时候需要把`select`语句写成`select xxx where ... for update`*(前提是`select`的`where`语句需要有索引)*.这时候事务里,会对这条记录的`select`操作加x锁.

但是,对db加读锁,哪怕只是行锁*(而且innodb自称行锁效率很高)*,总归是不安全的.

所以,这种场景,个人还是建议用分布式锁,比如通过**redis**实现的分布式锁*(`jedis.setnx`)*,通过对查询条件*(比如某个userid)*加分布式锁,抢占到锁的才进行查询和更新,抢不到的可以直接返回,不阻塞数据读取.



p.s. 推荐两篇感觉讲的很清晰的文章

[SELECT FOR UPDATE（转）](http://www.cnblogs.com/chenwenbiao/archive/2012/06/06/2537508.html)

[mysql事务和锁InnoDB](http://www.cnblogs.com/zhaoyl/p/4121010.html)