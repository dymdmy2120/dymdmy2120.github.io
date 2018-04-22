---
layout: post
title: "mysql事务"
date: 2018-04-15
categories: 数据库
excerpt: InnoDB事务
---

* content
{:toc}




# 认识事务

## 概述

事务是数据库区别于文件系统的重要特性之一，如果修改两个文件，在修改第二个文件时操作系统突然崩溃了，这回导致第一个和第二个文件数据不一致了，如果引入了事务，可以保证要么这两个文件都会修改，要不都不会修改，保证了原子性操作。 其实这两个文件可以看作是两个表。

InnoDB存储引擎的事务会满足ACID特性

Atomicity 原子性

Consistency 一致性

Isolation 隔离型

Durability 持久性


InnoDB默认使用的REPEATABLE READ完全遵循和满足ACID特性，隔离级别为 READ COMMITTED 不能满足隔离性，在事务A中如果多次执行 select * from t (每次获取最新的快照数据)会查询到 提交的事务B 插入的数据。

**Atomicity** : 

由于事务中会包含多个操作表(文件)的SQL，原子性是指这些sql语句要么都执行成功，要么都失败，任意一个sql执行失败都会回滚之前修改的数据。

**Consistency** :

这里的一致性是指：事务执行的前后依然不会改变数据库中完整性约束。


**Isolation** :


隔离性保证了同时运行的事务A 事务B 事务B操作的数据提交前对事务A是不可见的，这里就需要用到Record Lock、GAP Lock、Next-Key Lock 来实现。

**Durability**：

当事务已提交，那么数据就永远的持久化了，即使发生岩机，数据库也能将数据恢复(reado log)，但是如果一些外部原因，例如RAID损坏了，导致redo log写入失败，所有的数据可能会丢失。因此保证高可靠性，并不能保证高可用性，需要一些系统配合使用达到高可用。


## 分类

**扁平化事务：**

BEGIN WORK 

原子操作(要么操作成功、要么都执行失败)

ROLLBACK或 COMMIT


一种常用的一种事务，但是如果想回退事务的一部分就需要使用下面的保存点扁平化事务。 例如事务A中包括了 操作A1 操作A2 但是在操作A3时失败了，如果是扁平化事务会回滚事务中所有操作，这样代价会很大，所以有下面的保存点扁平化事务。


**保存点扁平化事务：**

可以根据业务逻辑，在事务执行过程中使用SAVE A 设置保存点，下次可以指定回滚到某个保存点 ROLLBACK A

**链式事务：**

和保存点扁平事务不同，就是只能回退到最近一个保存点，而保存点扁平事务可以回退到任意一个保存点。

**嵌套式事务：**

从顶级事务中创建子事务，其中子事务可以是嵌套事务也可以是扁平事务，把嵌套事务看作是树，子事务并不会立刻提交而是在顶级事务COMMIT后子事务操作才生效。


![props](http://dymdmy2120.github.com//static/2018-04-images/nest-transaction.png)


其实保存的事务可以模拟嵌套事务，父事务可以选择性的将锁传递给子事务


**分布式事务：**

事务的组成的SQL执行在不同的数据库服务器上，例如从工商银行转账到招商银行， 首先从工商银行数据服务器减少该账户的金额，然后在招商银行中的数据服务器增加该用户对应的金额，其中这两部任意一步失败了都需要回滚。

# 事务的实现

上一节讲解的锁就是用来实现事务的隔离性，redo log重做日志用来实现事务的原子性和持久性，undo log用来实现一致性和MVCC(多版本并发控制)

> redo和undo 不同点：
> 
> redo log记录的是具体修改了那些页，物理日志
> undo log 记录的是不同版本的行记录，以便于事务回滚使用，在每次修改行记录之前需要将其存放到undolog中

## redo log

**1、基本概念**

redo log是用来实现持久性的，使用来具体记录修改页的信息，和redo log buffer重做日志缓冲结合使用的，其中 redo log buffer容易丢失 redo log 是持久性的。

每次修改缓存池中数据之前需要将日志记录到redo log buffer中，由于修改的数据是在缓冲池中(内存中)并没有持久化到磁盘上，如果岩机了会导致数据的丢失，而使用redo log保证了每次修改的数据已经在磁盘中，下次重启时可以根据redo log进行恢复。

redo log一般都是顺序写入的，在数据库运行过程中不会读取，而undo log需要随机的读写(快照读)

在事务完成提及之前是否需要将redo log buffer中数据持久倒 redo log文件中，可以通过 innodb_flush_log_at_trx_commit参数进行设定，不同的参数值直接会影响到事物的长短和数据库性能。

由于每次将数据持久到磁盘中，首先会写入操作系统缓存中，需通过fsync写入磁盘中。

innodb_flush_log_at_trx_commit值 默认为1

0: 事务提交时不直接写入到磁盘中，而是等待master thread 每秒将redo log buffer数据 fsync到磁盘中。

1: 事务提交时马上调用 fsync持久到磁盘中

2：事务提交时会将日志写入文件中，但是只是放入操作系统缓存中，不进行fsync，如果数据库岩机了，那么这部分事务数据不会丢失，若是操作系统岩机了则会丢失。


可以做个测试，多50万条数据，每条数据插入都是一个事务,其实为了提高性能可以放在一个事务中。

![props](http://dymdmy2120.github.com//static/2018-04-images/innodb-commit.png)


**redo和binlog区别**

用途： binlog是用来主从数据库之间完成数据的同步

形式： redo是在InnoDB存储引擎层存在的，而binlog存在与数据库服务层，可以针对于任意一个存储引擎，记录了对数据库的更改

时间：一个事务中会多次记录日志，而binlog是在事务提交成功后记录日志

**2、log block**

![props](http://dymdmy2120.github.com//static/2018-04-images/log-block1.png)
![props](http://dymdmy2120.github.com//static/2018-04-images/log-block2.png)


**3、log group**

会由多个相同大小的的redo do file组成，当第一个写满了则进行写第二个，循环写入

**4、重做日志格式**

InnoDB存储引擎的管理是基于页的，那么重做日志具体记录到操作页的信息，那么页是基于页的。

数据结构为：

![props](http://dymdmy2120.github.com//static/2018-04-images/redo-format1.png)

![props](http://dymdmy2120.github.com//static/2018-04-images/redo-format2.png)


**5、LSN**

Log Sequence Number 表示的是日志序列号，首先事务提交时都会刷新到重做日志文件中，这时LSN会递增，这时修改的是缓冲池中的页，然后Master Thread会每1s将缓冲池中的页同步到表空间中(以表命名文件中)，这个时候每个页的头部会使用FILE_PAGE_LSN记录最新一次同步的LSN。

**LSN表示的含义：**

重做日志写入的总量(每次事务修改时都会进行递增单位字节)

checkpoint的位置(在Master Thread中定时任务最新一次同步到磁盘中的LSN)

页的版本(记录该页最近一次同步到磁盘中的LSN,以方便恢复)

例如P1的LSN=10000 代表字节 在数据库启动时重做日志的LSN=13000，那么数据库需要进行恢复操作，将重做日志应用到P1页中，对于重做日志LSN<10000，不需要重做，因为P1页中LSN表示已经刷新到该位置。

可以通过 show engine innodb status 查看当前页同步情况

Log sequence number 表示当前的LSN

Log flushed up to 表示刷新到重做日志文件中的LSN

Log checkpoint at 表示缓冲池中的页刷新到磁盘中的LSN


![props](http://dymdmy2120.github.com//static/2018-04-images/lsn.png)


**6、恢复**


![props](http://dymdmy2120.github.com//static/2018-04-images/restore.png)


## undo log

undo log用来记录修改之前行记录，以便下次回滚和MVCC 快照读，可以进行对undo log页进行重用


## purge

用专门的purge线程进行对已经提交成功的事务相关的undo log page进行清除，由于undo log会被其他事务进行引用，当使用快照读的时候就会使用undo log，所以清除这些undo log page保证没有其他事务引用。

## group commit

虽然现在有固态硬盘，在写入磁盘数据性能也得到提高，但是fsync性能也有限，group commit就是一次 fsync可以确保多个事务日志被写入到文件中。

# 事务控制语句

BEGIN|START TRANSACTION 开启一个事务

COMMIT|ROLLBACK 结束一个事务

SET TRANSACTION 设置事务隔离级别(READ UNCOMMITED、READ COMMITTED、REPEATABLE READ、SERIABLIZABLE)

SAVEPOINT A 可以自定义设定在那几条sql语句后面加保存点

RELEASE SAVE A 删除一个保存点

>注：
>ROLLBACK SAVEPOINT 这个只是将当前事务恢复到该保存点，这个事务并没有结束，必须通过COMMIT 或ROLLBACK来结束事务。

# 隐式提交的SQL语句

DML语句：

ALTER DATABASE ,ALTER TABLE ,TRUNCATE TABLE ........

TRUNCATE TABLE 和 DELETE不同的是不能回滚，DELETE可以回滚。


# 事务的隔离级别

>在READ COMMITED隔离级别下的binlog format一定为 ROW，否则会导致数据不一致性问题，如果是STATEMENT 格式的话可能记录SQL语句的顺序会改变。 其实为了保证master-slave数据一致性将事务隔离级别设定为REPEATABLE READ

# 分布式事务

当数据库资源存在于不同的机器上，为了又能保证ACID，事务的特性所以就有了分布式事务。

**MySQL分布式事务(外部事务)**

XA事务由一个或多个资源管理器、一个事务管理器、一个应用程序组成


![props](http://dymdmy2120.github.com//static/2018-04-images/xa.png)

两阶段：第一阶段是所有全局事务节点都开始准备，告诉事务管理器都已经准备好了，当事务管理器收到了所有节点同意信息后，来判断通知全局节点是COMMIT还是ROLLBACK。

JTA(Java Transaction API)


**内部分布式事务**


这个是指binlog和InnoDB存储引擎的redo log一致性，例如事务提交后写入到binlog，这时也已经同步到了其他slave节点中了，但是在写入redo log时岩机了，导致主节点和从节点数据不一致。



![props](http://dymdmy2120.github.com//static/2018-04-images/inner-xa.png)


# 使用事务的不好的习惯

## 不要在循环中提交事务

这样每次插入一条数据都需要记录一次redo log导致写入到磁盘性能直接影响到数据库的性能，而如果将这些操作放在一个事务中，写入redo log只写入一次大大提高了性能。

## 不要使用自动提及

可以在MySQL进行设置 set autocommit=0或 使用 BEGIN|START TRANSACTION。对于不同程序的API，自动提交是不同的，像MySQL Python会自动将 AUTOCOMMIT设置为0禁止自动提交，MySQL C是自动提及。

# 长事务

长事务，执行时间非常长意味着如果做到一半时出现错误时，回滚重做的代价非常大，所以需将事务划分成多个短事务，哪怕回滚代价也不会很大。
