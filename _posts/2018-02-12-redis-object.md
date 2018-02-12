---
layout:     post
title:      "读书笔记 : redis数据结构"
subtitle:   "《redis设计与实现》"
date:       2018-02-12
author:     "lee"
header-img: "img/back.jpeg"
---


最近在看`redis设计与实现`,觉得此书讲的很清晰,对`redis`底层也有了更多的认识  
以下是根据书的第一部分,总结出来的读书笔记.主要是关于`redis`的内部数据结构.  

### redis对象

`redis`存储都是基于`键值对`.它的键值都分别是一个`redis对象`,`键`都是字符串对象,`值`可以是以下五种对象之一.
> `字符串对象` : 例如直接set,get,setnx  
`列表对象` : 例如lpush,lset,lrange  
`哈希对象` : 例如hset,hget  
`集合对象` : 例如sadd,smembers,srem  
`有序集合对象` : 例如zadd,zrange,zincrby  


redis对象的数据结构定义如下 :
```c
//redis对象  

typedef struct redisObject {

	//对象类型  

	unsigned type:4;

	//编码(具体底层实现的方式)

	unsigned encoding:4;
	
	//对象指针

	void *ptr;
	
} robj;

```
它通过`类型+编码`的方式,来实际确定一个数据的数据结构.  

也就是说,对于不同类型的对象,redis也会根据实际存储时的数据类型和大小,来进行合理的内存分配与数据结构分配.  
每个类型的对象都至少有两种底层实现方式.具体可以看下面.  

`数据类型`跟`底层实现数据结构编码`的关系如下 :

对象名称 | 对象类型(type) | 底层实现(encoding) | 说明
- | - | - | - 
字符串对象 | REDIS_STRING | REDIS_ENCODING_INT | 值是整数值,且可以用long来表示的时候
字符串对象 | REDIS_STRING | REDIS_ENCODING_EMBSTR | 值是小于32字节的字符串的时候 (只会进行一次内存分配,robj和sdshdr是连续的空间)
字符串对象 | REDIS_STRING | REDIS_ENCODING_RAW | 除了前两种情况之外
列表对象 | REDIS_LIST | REDIS_ENCODING_ZIPLIST | 列表包含元素小于512个,且元素的字符串长度都小于64字节时
列表对象 | REDIS_LIST | REDIS_ENCODING_LINKEDLIST | 除了上述之外
哈希对象 | REDIS_HASH | REDIS_ENCODING_ZIPLIST | 哈希对象保存的键值对数小于512个,且键值的长度都小于64字节
哈希对象 | REDIS_HASH | REDIS_ENCODING_HT | 除了上述之外
集合对象 | REDIS_SET | REDIS_ENCODING_INT | 元素都是整数值,且小于512个
集合对象 | REDIS_SET | REDIS_ENCODING_HT | 除了上述之外
有序集合对象 | REDIS_ZSET | REDIS_ENCODING_ZIPLIST | 有序集合保存的元素小于128,并且长度都小于64字节
有序集合对象 | REDIS_ZSET | REDIS_ENCODING_SKIPLIST | 除了上述之外. 并且此实现用跳跃表和字典同时实现,字典记录了成员和分值的映射关系


下面针对几种用到的底层数据结构,分别罗列出来 : 


#### `SDS` : 简单动态字符串(Simple Dynamic String)

```c
struct sdshdr {
	
	//buf数组中已使用字符的长度(不包括结尾的空字符)

	int len;
	
	//记录buf中未使用字节的数量

	int free;

	//字节数组,用于保存字符串

	char buf[];
}

```

理解 : 
1. embstr和raw,都用到了sdshdr数据结构.embstr通过一次内存分配,请求到robj和sdshdr;而raw会请求两次
2. 能兼容c的字符串,但是比c的字符串更安全好用(二进制安全,不会溢出等).可以按需分配内存
3. 获取长度的复杂度是o(1)



#### `链表`

```c
//链表节点

typedef struct listNode {
	
	//前置节点

	struct listNode *prev;

	//后置节点

	struct listNode *next;

	//值

	void *value;
	
} listNode;

//双端链表

typedef struct list {
	
	//头节点

	listNode *head;

	//尾节点

	listNode *tail;

	//链表长度

	unsigned long len;

	//复制一个链表节点的值

	void * (*dup)(void *ptr);

	//释放一个链表节点所保存的值

	void (*free)(void *ptr);

	//对比一个链表节点的值和另一个值是否相等

	int (*match)(void *ptr, void *key);
} list;
```

理解 : 
1. 无环的双端链表.
2. 自带的三个函数,多态,可以支持不同类型的节点值.
3. 获取链表长度的复杂度是o(1)




#### `字典`

```c
//哈希表节点

typedef struct dictEntry {
	
	//键

	void *key;

	//值(联合体) : 分配在同一内存,可以理解为,值取下面三个中的一种

	union {
		void *val;
		uint64_tu64;
		int64_ts64;
	} v

	//下一个哈希表节点 : 用于解决键冲突

	struct dictEntry *next;

} dictEntry;

//哈希表

typedef struct dictht {
	
	//table是数组,每个元素指向一个哈希表节点.每个数组元素上,如果有键冲突,会形成一个链表

	dictEntry **table;

	//table的大小

	unsigned long size;

	//size-1,用来计算索引

	unsigned long sizemask;

	//目前已有键值对的数量

	unsigned long used;

} dictht;

//字典

typedef struct dict {
	
	//特定的一些函数

	dictType *type;

	//特定函数需要的参数.多态,根据值的类型不同,操作函数的参数也不同.

	void *privdata;

	//哈希表

	dictht ht[2];

	//rehash会用到的索引值

	int trehashidx;

} dict;

//用于操作特定类型键值对的函数 (一组)

typedef struct dictType {
	
	//计算键的哈希值 (这里的const void * 确保了参数需要是一个常量指针)

	unsigned int (*hashFunction)(const void *key);

	//复制键

	void * (*keyDup)(void *privdata, const void *key);

	//复制值

	void * (*valDup)(void *privdata, const void *obj);

	//对比键

	int (*keyCompare)(void *privdata, const void *key1, const void *key2);

	//销毁键

	void (*keyDestructor)(void *privdata, void *key);

	//销毁值

	void (*valDestructor)(void *privdata, void *obj);

} dictType;


```
理解:
1. 字典是redis里广泛应用的数据结构,他的底层是哈希表
2. 字典里保存了两个哈希表,一般只用第一个哈希表.如果要对哈希表进行扩展或收缩,就会用到第二个哈希表来进行rehash
3. rehash的时候是渐进的,会用到rehashidx来进行记录(记录rehash进行到哪个key了)
4. 哈希表通过链地址法来解决键冲突
5. 字典的数据结构定义,采用了多态.包括里面的函数定义,都能支持不同类型的键值对,有各自的实现.


#### `跳跃表`

```c
//跳跃表节点

typedef struct zskiplistNode {
	
	//每个节点都有一个层的数组

	struct zskiplistLevel {

		//指向下一个节点

		struct zskiplistNode *forward;

		//到下一个节点的跨度

		unsigned int span;

	} level[];

	//后退指针

	struct zkiplistNode *backward;

	//分值

	double score;

	//robj是redis的通用obj定义

	robj *obj;
	
} zskiplistNode;

//跳跃表

typedef struct zskiplist {

	//指向头节点和尾节点

	struct zkiplistNode *head,*tail;

	//节点数

	unsigned long length;

	//层数最大的节点的层数

	int level;
	
} zskiplist;

```

理解:
1. 跳跃表在redis里被用在有序集合的底层实现
2. 大部分情况下,他的查找效率接近平衡树




#### `整数集合`

```c
typedef struct intset {

	//编码

	uint32_t encoding;

	//大小

	uint32_t length;
	
	//数组

	int8_t contents[];
	
} intset;

```

理解:
1. redis里,当一个集合只包含了整数,并且元素不多时,redis就采用整数集合来作为集合的底层数据结构
2. 编码决定了集合里元素的编码到底是int16还是int32还是int64
3. contents字段存储了整数集合里的元素,有序且不重复.具体的类型不是int8,而是根据实际encoding字段给出的类型来决定
4. 当插入一个大于或小于所有整数元素的值,而导致原来的int编码类型不支持时,intset就会进行升级,每个元素都会变成升级后的.并且不会降级
5. 这样的好处是可以根据需要灵活分配内存




#### `压缩列表`  

压缩列表:

整个列表占用内存的字节数 | 从开头到表尾节点的偏移量 | 包含的列表节点数 | 可以放多个节点 | 0xff,标记末端
- | - | - | - | - 
zlbytes | zltail | zllen | extryX | zlend

压缩列表节点:

previous_entry_length | encoding | content
- | - | -
前一个节点的字节长度 | 编码 | 字节数组 或 整数


理解:
1. 压缩列表(ziplist)是由一系列连续分配的内存块组成的数据结构,是列表键和哈希键的底层实现之一.
2. 如果一个列表键或哈希键包含的元素不多,并且是类型都是整数或者长度较小的字符串,则一般底层实现是ziplist
3. 压缩列表的节点里,每个节点都会记录上个节点的字节长度,便于从表尾遍历到表头(或者任意节点向前遍历)
4. previous_entry_length大小为1个字节,可能存在的一个问题,即如果所有节点的prev都是一个字节且接近255,更新节点的时候可能会引发连锁更新













