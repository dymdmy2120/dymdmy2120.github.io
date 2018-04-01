---
layout: post
title: "mysql索引"
date: 2018-03-30
categories: 数据库
excerpt: InnoDB存储数据的索引和算法
---

* content
{:toc}




# InnoDB存储引擎索引

## B+ Tree索引


包括聚集索引和辅助索引，注意这个b 是指balance意思不是binary, B+ Tree是从平衡二叉树演化而来的，但是并不是二叉树。

当给定一个具体值时，B+ Tree并不能定位到该值对应的行，只能定位到该值所在的页，然后加载到内存中，再在内存中进行查找。

## 全文索引

可以对数据表中所有字段的值建立一个索引，这样可以快速定位到出现某个值的所有行记录


## 哈希索引

InnoDB中会根据某个字段查询的次数，会自动进行对key-value建立一个hash索引，这个索引是自动生成，不能人工干预。


# 数据结构与算法

## 二分查找

	#循环
	
	def binarySearchByLoop(list,value) :
	
		size = len(list)
		start = 0
		to = size-1
		while(start <= to) :
			mid = (start+to)/2	
			if(list[mid] == value) : return mid
			if(value > list[mid]) :
				start = mid+1
			if(value < list[mid]) :
				to = mid-1
		return -1
	
	#递归
	
	def binarySearchByRecurse(list,start,to,value) :
	
		if(start > to) : return -1
		mid = (start+to)/2
		if(list[mid] == value) : return mid
		if(list[mid] > value) :
			return binarySearchByRecurse(list,start,mid-1,value)
	
		if(list[mid] < value) :
			return binarySearchByRecurse(list,mid+1,to,value)



## 二叉查找树和平衡二叉树

二叉树特点是：左孩子节点比父节点值要小，右孩子节点都比父节点要大。

当二叉查找树的所有节点都在同一边的情况下，查找的效率和顺序查找相当，所以引进了平衡二叉树，任何节点的两个子树的高度最大差为1

InnoDB中有两种存放行记录的格式：Compact压缩型的和Redundant冗余型的

# B+ Tree

B+ Tree的结构

![props](http://dymdmy2120.github.com//static/2018-03-images/btree-example.png)


## B+ Tee插入

根据叶子节点和索引节点目前存放数据的多少，会使用不用的插入算法，保证B+ Tree平衡。以下就是不同情况下，进行的插入操作：

![props](http://dymdmy2120.github.com//static/2018-03-images/btree-insert.png)

由于B+ Tree结构主要用于磁盘，对页的多次拆分，就会进行多次磁盘I/O，所以会影响性能，B+ Tree也可以像平衡二叉树旋转功能，如果本页已经满了，并不会急着拆页，而是会检查兄弟节点，然后将本身的数据移到兄弟节点上,做旋转操作。

![props](http://dymdmy2120.github.com//static/2018-03-images/btree-rotation.png)

## B+ Tee删除

B+ Tree会根据填充因子进行控制对页的删除并且合并，合并后页内的数据仍然有序，该 fill factor可以设置默认为50%,也就是当页内的记录小于该页容量的50%时就会删除该页，将其数据移到其他页中。

同样删除也存在以下几种情况：

![props](http://dymdmy2120.github.com//static/2018-03-images/btree-delete.png)

# B+ Tree索引

B+ Tree中的高扇出性，也就是树的分支比较多，一般有100个分支，树的高度为3～4层，那么容下100万～1亿的数据量，一般的磁盘操作1s可以进行100次，所以根据某个键锁定记录时大概需要3～4次磁盘I/O 需要 0.03～0.04秒。

数据中的B+ Tree索引分为 聚集索引和辅助索引， 之间的不同是 聚集索引的叶子节点存放的是整行的具体记录信息，而非聚集索引存放的是key+rowId, 后面再用rowId进行根据聚集索引再查询一次。

## 聚集索引

就是数据本身会根据主键(如果没有定义则找唯一键，否则会随机产生6位的rowId)进行排序的。 数据页之间通过双向链表进行链接的，可以过寻找数据页中的记录找到整行信息，索引优化器首先会使用聚集索引，例如根据id进行范围的查找，由于数据是根据主键排序的，逻辑上是有序的，因此可以知道需要读取哪些页，并且对磁盘也是顺序读取的。

非叶子内的记录也是通过链表进行链接的，里面存放的是主键的值和指向下一页的偏移量。

聚集索引的结构：


![props](http://dymdmy2120.github.com//static/2018-03-images/btree-struct.png)

聚集索引如果按照物理顺序存放记录则成本会非常大(当删除了某些数据时，这些空间都不能重复利用导致消耗更多的空间)，而是逻辑上物理记录是有序的，页与页之间通过双向链表链接的，而页内的记录也是通过链表链接起来的。

聚集索引对于根据主键的排序查找和范围查找非常快。 例如：

select * from t where order by id limit 10;

由于数据页之间是通过双向链表，所以很快知道最后一页的，然后加载到内存中，取出最新的10条记录。 虽然使用了order by 但是并没有对查出的结果进行在内存中排序，因为聚集索引已经根据主键排好序了。

另一个范围查询(range query) :

select * from t where id > 10 and id <1000;

可以先找到id=10的页加载到内存中，然后遍历其内部记录，后面会不断加载兄弟节点到内存，直到满足条件。

## 辅助索引

和聚集索引一样也是B+ Tree结构，不同的就是叶子结点中行记录为 key+rowId，当根据辅助索引查找数据时，需要根据辅助索引定位到对应叶子结点中对应行的主键id,然后进行一次聚集索引，最终获取行记录信息，总共的磁盘I/O为：3次+3次=6次

结构为：

![props](http://dymdmy2120.github.com//static/2018-03-images/btree-second.png)

![props](http://dymdmy2120.github.com//static/2018-03-images/bree-second-a.png)


## B+ Tree索引的分裂

上面说到如果某个页的行记录数已经满了，则插入某条记录之前会进行分裂该页数据，以页记录的中间位置的记录为分裂记录，分成两个页，这里就会有个问题，对于自动增长的id来说，前面一页永远有部分都是空的，浪费空间，而对于随机的主键则可以采取中间分隔方式。 否则就可以根据要插入的记录为split record进行分割。

分割过程为：

![props](http://dymdmy2120.github.com//static/2018-03-images/btree-split.png)


## B+ Tree索引的管理

创建索引和删除索引两种方式： alter table 和 create drop

alter table t add key|index key_name(column_name)


alter table drop key_name


create index index_name(column_name) on t

drop index index_name on t 

alter table t add key index_b(b(100)) 表示对b列中的前100个字符进行索引

通过 show index from t 进行查询表t的索引的信息：其中有个信息 Cardinality这个值越和当前记录数越靠近，该索引效果就很明显，表示该列唯一的记录数。 这个值只是一个预估值，并不准，因为为了保证效率这个值并不是实时统计的，而是当记录超过一定数，或者修改页次数达到一定数，就会随机选择8个数据页统计其中唯一记录数b 则推测出所有唯一数据： （c1+...c8）*A/8 (A为总的数据页) 。如果想要一个准确值时，可以通过 analyze table t 分析一次表。

InnoDB如何在数据库运行中进行对索引的增加、删除呢？

在5.5版本之前，当创建新的索引时，首先会创建新的表然后加上索引，然后将数据导入新表中，最后drop原来表，并重新命名新表名。其间都不能处理新的事物和读请求，这是无法忍受的。 后面就有了Fast Index Creation，就是给表加上了SHARE锁，期间只能接受读请求，不能处理事务。

在5.6版本出现了 Online DDL 其中也可以通过配置 参数 old_alter_table 设定在创建和删除索引时对表加锁的级别：

NONE:为不加任何锁，可以获得最大并发读，还有 SHARE:只能读 EXCLUSIVE:不能读和写 DEFAULT:根据事务的并发性来进行判断使用哪种锁的级别。

在创建和删除索引时，同时支持读和写的原理是： 在创建索引期间，像INSERT UPDATE DELETE这类DDL操作都是操作缓冲中的数据，当索引构建完成了，就会把重做日志应用到新的索引上，优化器是不会选择正在创建的索引。

# B++ Tree索引的使用

## 联合索引

联合索引是由多个字段组成的索引键，此时键的数量大于等于2，同样也是按照键值的顺序存放的，例如定义联合索引 (a,b) 首先根据a进行排好序，当a相同时再进行根据b排序。 查找时必须知道a才能应用索引。

![props](http://dymdmy2120.github.com//static/2018-03-images/btree-multi-key.png)


联合索引；首先会根据第一个字段建立索引，在第一个字段相同的情况下，再继续根据第二个字段建立索引。 结构为：

![props](http://dymdmy2120.github.com//static/2018-03-images/union-index.png)

例子：

alter table t add key index_a_b(a,b)

select * from t where a=1 and b=2 很显然此查找可以使用联合索引。

如果还存在 单键索引 a，那么 select * from t where a = 1这个查询的优化器会选择单键索引列，因为，同等大小的页，单键消耗空间<多键消耗的空间，也就是说同一页中单键存放行记录会更多，进而有更少的磁盘I/O。

select * from t where a = 1 order by b 此次查询优化器会选择联合索引，因为在a相同情况下根据b进行排序了(注如果a<1时，通过explain看sql执行计划中 extra：using filesort 把选择的结果在内存中重新排序一次) 所以直接返回根据b排好序的数据。

对于 联合索引 (a,b,c)

以下可以直接使用联合索引的结果

select * from t where a = 1 order by b

slect * from t where a = 1 and b = 2 order by c

以下就不能使用联合索引的结果，因为(a,c）并不是有序的

select * from t where a = 1 order by c

## 覆盖索引

explain 分析查询计划 有个字段 extra：

using filesort:表示order by 并没有使用索引内部的排序结果，在内存中重新做了排序

using index:表示该查询使用了覆盖索引

using where: 在查询过程中使用了where后面条件，例如 where a = 1 limit 3 就会有 using where

using index condition:后面会将到对索引的一次优化 Index Condition Pushed,意思就是在扫描索引的同时将where后面条件也带上，避免无效的数据在数据库Server层进行过滤，提高效率。

NULL: 表示走正常的流程，从辅助索引获取主键然后到聚集索引再找一次。


查找过程中，直接使用辅助索引的记录即可返还结果，无需再通过主键到聚集索引查找一次， 另一好处是：辅助索引不包含整行的信息，故大小远小于聚集索引，因此可以减少大量的磁盘I/O

对于联合索引 (a,b)

以下都会使用覆盖索引获取数据，extra为using index

select b where a = 1

select id,b where a = 1

还有一种使用统计表中的数量时 count(*)会使用覆盖索引

select count(*) from t 如果t中有辅助索引，则优化器会使用， 因为辅助索引远小于聚集索引，可以减少磁盘I/O操作。

select count(*) from t where b<1 and b>10 此时会使用联合索引 (a,b) extra为 using where using index

## 优化器选择不使用索引的情况

一般是Range查询，优化器会选择不会使用辅助索引。

当存在辅助索引 order_id  主键为 id

explain select * from t where order_id < 1000 and order_id > 100000

发现 key为PRIMARY extra:using where ，也就是优化器使用了聚集索引然后进行全表扫描，最后在内存过滤掉不符合条件的数据，为了不选择使用order_id辅助索引，由于直接使用聚集索引时，是顺序的访问磁盘数据的，而如果使用辅助索引则会通过id继续查找一次，那么时离散性读取磁盘数据，顺序>随机读(机械磁盘随机需要寻道)。如果已经知道使用的是固态硬盘的话，可以使用关键字 FORCE INDEX 强制使用某个索引 select * from t FORCE INDEX(orderId) where order_id < 1000 and order_id > 100000

## Multi Range Read

含义：就是将通过辅助索引查找到的主键id集合按照主键进行排好序，保证是有顺序的访问聚集索引。 其中还可以对查询条件拆分成键值对

例如：联合索引 (a,b)

select * from a<10 and a >1 and b=8 如果不使用MRR那么先使用联合索引获得a的范围数据然后根据b的条件过滤，假如有大量的数据都是b!=8则使用MRR优化就可以提高性能了， 拆分成 (1,8) (2,8) ... (9,8)


## Index Condition Pushed

就是对于 where后面的条件过滤不在Sever层进行过滤了，而是在扫描索引时带上过滤条件(也就是过滤操作在存储引擎层完成) 大大提高数据库性能。

**注意：** where后面的过滤条件要在索引覆盖的范围之内


# 哈希索引

给定一个被缓存的页，如何能在内存为128G空间中快速找到，并不能每次都遍历所有内存进行查找，而是通过O(1)的哈希算法获取。

InnoDB innodb_buffer_pool_size 缓冲池的大小，相当于640个页 一个页16K ,缓冲内存的哈希表来说 需要 640*2 = 1280 但是需要质数，比1280大的质数为1399， 槽的大小为1399，用户查询是 某个表空间 space_id 的某个连续的16K的页 offset为页的偏移量。key的值为 space_id左移20位加上 space_id和offset

Key = space_id << 20 + sapce_id + offset,然后通过key对槽位取余。

InnoDB采用的是自适应的哈希，无需人工干预，但是对于Range query无能为力

# 全文搜索

全文搜索就是把某列的所有内容，根据分词的特性建立索引，例如文章数据库，对详情字段建立一个全文索引。 InnoDB并不支持全文索引，InnoDB1.2x才支持 MyISAM也支持

## 倒排索引

有两种形式：

下面的单词可以理解为根据某规则进行分词后的单词，如果英文则根据空间分隔的

1、inverted file list 表现形式：{单词,documentId}

2、full inverted file list 表现形式：{单词,{documentId,在文档中具体的位置}}

![props](http://dymdmy2120.github.com//static/2018-03-images/full-text.png)


![props](http://dymdmy2120.github.com//static/2018-03-images/full-text-info1.png)

![props](http://dymdmy2120.github.com//static/2018-03-images/full-text-info2.png)

如果想要查看单个文档出现some单词次数为两次文档id，则可以通过full inverted file list查看，因为其存储了该单词在文档中的位置。
