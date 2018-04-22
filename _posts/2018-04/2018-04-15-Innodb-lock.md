---
layout: post
title: "mysql锁"
date: 2018-04-15
categories: 数据库
excerpt: InnoDB锁
---

* content
{:toc}




# InnoDB存储引擎锁

## Lock和Latch区别


Lock锁定的对象是事务，锁定数据库中对象，如表、页、行记录，当事务commit或rollback锁就会释放(释放时间可能和事务隔离级别有关)，具有死锁机制检测

Latch锁定对象是线程，为了保护内存数据结构例如缓冲池中的LRU。没有死锁检测，按照加锁顺序加锁

## Innodb存储引擎中的锁

**锁的类型：**

S 共享锁 只允许其他事务加S锁，读一行数据，如果加X锁则等待

X 排他锁 允许事务更新或删除一行数据，如果加X锁和S锁则等待

S和X指的都是行锁

意向锁：

为了能支持多个粒度加锁操作，所以才有意向锁，如果想对行记录加锁之前必须要对表、页加意向锁，若任意一个部分导致等待，则需要等待粗粒度锁的完成。

>如果给事务A给十亿数据加表锁，那么需要遍历每条数据看是否有加锁，但是有意向锁，事务B会在给索引记录加锁X之前会给表加上意向锁IX，然后事务A发现加的锁和意向锁冲突了，则等待事务B释放。

> 说白了意向锁的主要作用是处理行锁和表锁之间的矛盾，能够显示“某个事务正在某一行上持有了锁，或者准备去持有锁”


**一致性非锁定读**
 
 当前行加了X锁，但是读取时并不会加锁，而是会读取快照(MVCC 多版本并发控制，采用了undo log进行实现的，undo log保存数据行记录的多个版本),不同的隔离级别方式不一样， REPEATABLE READ 会读取开始事务时的那个版本，而 READ COMMITED
 每次都会读取最新版本数据
 
 **一致性锁定读**
 
	 select ... for update
	 
	 select ... lock in share mode
 
**自增长与锁**
 
 每个自增长表中都有有个自动增长器，使用 select max(auto_inc_col) from t for update ,其中使用 AUTO-INC Locking 锁技术来保证主键自动增长，并不是等到事务完成后才会释放该锁，而是在完成对自增长值插入的SQL后立即释放。
 
 **外键与锁**

在Innodb中为了避免表锁，如果外键没有索引则会自动加上锁，而oracle则不会。 每次在子表中对外键的修改或删除、插入时都需要到父表中查询一次看是否存在， 这个时候会在父表的主键加上S LOCK,注意这里不能使用一致性非锁定读，因为每次都是从开始事务时的快照读的。

## 查看具体事务、锁、锁等待信息

在 information_schemas 数据库中三张表中

 INNODB_TRX  存放事务相关的信息 事务在运行还是锁等待
 
 INNODB_LOCKS  存放有关锁的信息，锁id 事务id 锁的模式(X S) 锁的类型等，具体到那个表锁了哪几行数据
 
 INNODB_LOCK_WAITS 存放了当前运行的是那个事务以及和用了那个lock id
 
 requesting_trx_id:申请锁资源的事务id也就等待锁的事务
 
 requesting_lock_id:申请事务锁id
 
 blocking_trx_id:阻塞的事务id 也就是当前运行的事务
 
 blocking_lock_id:当前事务用的锁id
 
 如果事务量大的话，一般使用上面三张边做联合查询


# 锁的算法

**Record Lock**：记录锁 记录指的是索引记录，而不是具体行数据

锁住当前索引记录以及对应的聚集索引，如果sql语句没有用到索引则会对每一条聚集索引后加X锁，类似于全表锁。

**GAP Lock**： 间隙锁 锁定一个范围 不包括记录本身 一般针对非唯一索引而言的

gap lock的机制主要是解决可重复读模式下的幻读问题，例如select * from t where id > 7 for update 这个时候会在(7,+infinity)加上gap锁，例如 insert into t values(8,...)将会阻塞，如果没有gap锁则在同个事务多次执行select * from t where id > 7 for update 会出现多种结果


**Next-Key Lock**： 等价于GAP Lock + Record Lock 锁定一个范围包括记录本身

默认情况下，InnoDB工作在可重复读隔离级别下，并且会以Next-Key Lock的方式对数据行进行加锁，这样可以有效防止幻读(Phantom Problem)的发生。Next-Key Lock是行锁和间隙锁的组合，当InnoDB扫描索引记录的时候，会首先对索引记录加上行锁（Record Lock），再对索引记录两边的间隙加上间隙锁（Gap Lock）。加上间隙锁之后，其他事务就不能在这个间隙修改或者插入记录。

注：如果更新条件用的不是索引，则会进行对表加锁，如果为唯一索引则降级为记录锁，辅佐索引则会使用gap lock和next-key lock ，适用于REPEATABLE READ及以上隔离级别。

如果隔离级别为READ COMMITED 只会锁住已有的记录，不会加gap锁

如果隔离级别为 SEARIALIZABLE 对于普通的select 也会变成 select ... lock in share mode 也会获取 gap next-key 锁

例子：

![props](http://dymdmy2120.github.com//static/2018-04-images/gap-lock.jpg)

	myid为非唯一索引
	delete from test_gap_lock where myid=100

	从图中的数据组织顺序可以看出，myid=100的记录有两条
	如果加gap锁就会产生三个间隙，分别是
	gap1（98,100），gap2（100,100），gap3（100,105）
	在这三个开区间（如果我高中数学没记错的话）内的myid数值无法插入
	显然gap1还有(myid=99，id=3)(myid=99,id=4)等记录
	gap2无实际的间隙，gap3还有（myid=101，id=7）等记录。
	并且，在myid=100的两条记录上加了record lock
	也就是这两条数据业务无法被其他session进行当前读操作


# 锁问题

## 脏读

事务A中能读到未提交事务B修改的数据，这个数据称为脏数据，因为事务B可以会回滚。这个在READ UNCOMMITTED隔离级别下才会发生，像MySQL InnoD默认隔离级别为 REPEATABLE READ

## 不可重复读

在同个事务A多次执行相同sql语句时会得到不同的数据结果，select * from t where id > 7 ,如果其他事务Binsert id = 8那么事务A又会多了id=8的数据，和脏读不同的是读到的数据都是其他事务已经提交的，但是违反了事务的一致性。 如果隔离级别为REPEATABLE READ 使用NEXT-KEY LOCK 给索引加上gap lock保证了事务的可重复读， 例如会对 (7,+infinity)加上 gap lock 其中也对7记录本身进行锁定，其他事务不能对id在其范围内进行更新操作。

## 丢失更新

事务1查询id=3的数据到内存然后进行操作

事务2查询id=3的数据到内存也进行操作

其中就会出现数据的丢失问题，如果事务2发生最后，那么就会覆盖事务1的操作数据，此时可以数据库层面进行解决，select * from t where id = 3 for update 由于默认的select 查询为快照读，所以需要对读取的这一行数据加行级别的X锁 for update，其他事务的select ... for update 会阻塞等到上个事务commit 或rollback。

## 阻塞

对于不能获取对应的锁的事务只能进行等待，可以通过参数 innodb_lock_wait_timeout 控制等待超时时间，默认为50s innodb_rollback_on_timeout 这个就是在超时时是否需要回滚操作，此参数静态不能更改，默认为OFF。不进行回滚，所以需要在业务代码中，如果发生等待超时异常是否需要回滚。

## 二叉查找树和平衡二叉树

二叉树特点是：左孩子节点比父节点值要小，右孩子节点都比父节点要大。

当二叉查找树的所有节点都在同一边的情况下，查找的效率和顺序查找相当，所以引进了平衡二叉树，任何节点的两个子树的高度最大差为1

InnoDB中有两种存放行记录的格式：Compact压缩型的和Redundant冗余型的

# 死锁

当事务1获得A锁时同时还需要等待获取B锁， 此时事务2获得到了B锁同时还需要获得A锁，那么这个时候就会发生死锁。 对于上面提到了锁等待超时异常并不会回滚事务，但是死锁这种异常会选择一个代价小的(通过 innodb_tx表中记录的weight 代表事务进行操作的个数)事务回滚。

## 死锁的概率

发生死锁的概率和以下因素有关：

系统中事务量越大，发生的概率越大

每个事物操作的数量越多，发生的概率越大

操作数据(例如查询某几行数据)集合越小，发生概率越大

## 锁的实现

会为每个页分配锁空间，用位图方式实现，假如每个页用30字节来存储锁信息，一共240bit用来存放对应记录的锁信息。 这比对每条记录使用10个字节存放锁信息要节省好多空间。
