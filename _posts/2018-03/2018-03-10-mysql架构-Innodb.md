---
layout: post
title: "mysql之架构详解"
date: 2018-03-10
categories: 数据库
excerpt: MySQL的架构和Innodb特性
---

* content
{:toc}

# MySQL体系结构和存储引擎

## 一、数据库和实例的区别

数据库是一种物理操作系统文件(数据文件、索引文件、事务用到的日志文件)或其他形式文件类型的集合，如果使用NDB存储引擎时，数据库的文件就不是操作系统上的文件而是存放在内存中的文件，但定义保持不变。而实例是程序，是位于用户和操作系统之间的一层数据管理软件。运行起来就是一个进程，用户可以通过实例来进行DML DDL等操作(实际上是对文件的操作)

mac环境启动mysql

/usr/local/mysql/support-files/mysql.server start

/usr/local/mysql/support-files/mysql.server restart

/usr/local/mysql/support-files/mysql.server stop

## 二、体系架构

![props](http://dymdmy2120.github.com//static/2018-03-images/mysql-architect.png)

1、连接池组件

2、管理服务和工具组件

3、SQL接口组件

4、查询分析器组件

5、优化器组件

6、缓冲组件

7、可插拔式存储引擎

8、物理文件

存储引擎是基于表的，可以给数据库中的每个表指定不同的存储引擎，每个存储引擎会根据SQL Interface定义的接口进行相应的操作底层的各种文件的实现。

## 三、存储引擎的比较

1、InnoDB

支持事务(MVCC 提高并发，默认隔离级别为REPEATABLE READ 采用next-key locking策略实现避免幻读)，行级锁，外键，B+ tree索引或自适应哈希，聚集存储方式，每张表存储都是按照主键顺序存放的，若没指定则会自动生成6字节ROWID

2、MyISAM

支持地理空间存储，不支持事务，表锁，全文索引 缓存的是索引文件，数据文件不会索引(会使用操作系统的缓存) .MYI 索引文件 .MYD数据文件。可以使用myisampack进行对表的数据进行压缩，变成只读数据，如果要修改就需要解压后进行更改。

3、NDB

行级锁，索引和数据都放在内存中，使用B+ tree和哈希索引

3、Memory

表级锁，不支持事务，B+ tree和哈希索引，数据放在内存，重启后表中数据丢失，一般存放临时表，不能使用TEXT BLOB列类型，例如存储varchar变长字段时会按照定长char方式进行，因此会浪费内存

## 四、连接

1、TCP/IP

在连接之前会验证连接的client的ip和用户名、密码是否在mysql数据库中的表中出现过。其实可以使用root用户进行赋予权限 grant all privilege (select、insert等) dbName.* 某个数据库用户@登录端ip identified by password 
mysql -h127.0.0.1 -uroot -proot一般使用TCP/IP方式进行远程的连接，在连接端会启动一个客户端进程进行连接，连接成功后,就可以进行数据传输了。如果使用 -hlocalhost 或者不输入host则直接走的是本地的unix域套接字/tmp/mysql.sock socket连接

2、命名管道和共享内存

客户端和服务端都在同个机器上，可以使用named-piple或共享内存方式进而两个进程可以通讯

3、unix域套接字

同样也是需要连接端和服务端在同台机器上，此连接不走网络，通过本地的socket进行连接的。当MySQL服务启动时会在 /tmp目录下生成一个mysql.sock的socket文件，进而可以通过 mysql -h127.0.0.1 -uroot -proot -S /tmp/mysql.sock进行连接，



show engine innodb status \G;

srv_master_thread loops:2188 1_second,1537 sleeps, 218 10_seconds, 2 background, 2 flush

主线程的主循环运行类2188次，此数是根据定时器1s运行一次，一共过了2188s理论上运行了2188次主循环，但是由于循环体内需要处理事情，导致时间上那时刻只有运行了1537次，所以通过此差值可以看数据库的负载情况。


#  InnoDB存储引擎架构

![props](http://dymdmy2120.github.com//static/2018-03-images/innodb-inner.png)
 

##  线程

**主线程：**主要将缓冲池中的数据定时刷新到磁盘中，保证内存中数据是最新的，同时保证在数据库发送异常时能恢复到正确状态

**IO线程：**read thread 、 write thread 、log IO thread(在更新缓冲数据页前先放入到redo缓冲数据区，然后定时更新到磁盘) 、insert buffer thread(合并插入缓冲) 其中读写线程可以使用innodb_read/write_io_threads参数进行控制，默认为4

**Purge Thread：**对于已经提交的事务的undolog页进行回收，InnoDB1.1之前是在Master Thread为了提高CPU使用率，单独出来用一个线程做这个事。

 **Page Cleaner Thread：**缓冲池中脏页数据同步到磁盘中，一开始在主线程中，后独立一个线程避免查询时候阻塞，提高数据库的性能

##  内存

 数据库中页的结构：
 首先page_no是由一个int 4个字节32位表示的，一个页存放16KB数据，页头+内容+内尾部

 其中页头又由pageNumer(唯一标识该页)+Previous Page指针+Next Page指针
 User Record 存放的就是每条记录
 格式有四种：
 主索引的非叶子节点 (最小的key,指向符合条件的page_no Page编号)

 主索引的叶子节点：主键值 和 除非主键其他列的集合

 辅助索引非叶子节点 (最小的key,指向符合条件的page_no Page编号)

 辅助索引叶子节点 辅助索引键值 和 主键值

 详见：[InnDB页的数据结构](https://segmentfault.com/a/1190000008545713#articleHeader5)

 为了解决慢的磁盘速度给数据库带来的影响，所以使用缓冲池(一块内存空间)来提高数据库的性能，当使用(查询或修改)某页数据时首先判断缓冲池存不存，若存在则直接使用，否则从磁盘中加载到内存中。对内存的操作比操作磁盘速度更快，更高效，其中在内存修改的页数据什么时候同步到磁盘中，有个**Chekpoint**机制进行控制的。

**1、内存架构**
![props](http://dymdmy2120.github.com//static/2018-03-images/buffer-architect.png)

 索引页、数据页、insert buffer(插入缓冲)、自适应哈希索引、lock info锁信息、数据字典
 redo缓冲(重做日志缓冲)
 额外内存
 
**2、LRU list Free list Flush list**

  LRU list 使用LRU数据结构来管理缓冲池页，其中缓存池中存储最小单位是页，默认为16KB，当缓冲池空间不够时，会删除最底部的页，最近访问的页放在顶端。

>   要点1：
> 
>   但是在传统的LRU list基础上做了些改动，有个midpoint概念，默认每次从磁盘查询到的页会放在LRU列表的3/8的这个位置，其上面为new列表(存放的是热数据)，其下面为old列表。为什么不直接放在顶部呢？因为对于某些扫描全表的查询会访问全部页时，会把原来热点的数据页移除掉实际上这些数据下次可能都用不上。
>   因此每次都会放在3/8的位置，如果空间不够的话仅仅只会3/8位置之下做移动，不会影响热点数据。可以通过innodb_old_block_pct进行配置midpoint位置，默认为37 也就是37%大概3/8的位置
> 
>   如果在非热位置的页再次访问时就会从old移动到new列表中 这称为page made young，其中可以通过参数 innodb_old_block_time单位毫秒 进行设定多久(默认为1000ms)之后再次访问就放入到热数据区,如果因为在设置的时间内访问了该页而后面没有访问过导致没有移动到new列表
>   称为 page not made young，也就是要想将数据从old列表移到new列表 至少要在innodb_old_block_time时间后。

>   要点2：
> 
>   通过 show engine innodb status\G; 查看innodb运行的转态时，有个 Buffer pool hit rate 此值代表缓存池命中率，就是加载到内存的页下次被重新访问的比率，如果该值小于95%，看看是不是因为全部扫描导致LRU列表被污染了。
> 
>   Free list 存放的空闲页，每次从磁盘中加载页到内存中，首先去Free list申请页，申请成功后从list中删除放入到LRU中，否则会淘汰LRU列表末尾的页，将该内存空间分配给新的页
> 
>   Flush list 脏页既存放在LRU list中又在Flush list，表示列表中的页都是脏页(内存中的页和磁盘的页不一致)，经过CHECKPOINT机制刷新会磁盘中

  **3、重做日志缓冲 redo log buffer**

  其中在主线程中会每隔1s刷新一次到日志文件中，所以保证1s的事务量小于缓冲大小 默认为 innodb_log_buffer_size:8388608 8M
  下面三种情况下会刷新到磁盘文件中：
  主线程中每秒将重做缓冲刷新到日志文件中
  每次提交事务时也会将重做缓冲刷新到日志文件中
  当重做日志缓冲空间小于1/2时也会将重做缓冲刷新到日志文件中

 **4、额外内存**
 例如一些Lock等信息会从该内存申请，如果该内存不够时会从缓冲池中申请，所以当缓冲池内存增大时，同样该额外内存也要增大。
 
##  Checkpoint技术

 该机制实际上就是将缓冲池中修改的页数据同步到磁盘中(当缓冲池不够用时，或重做日志不可用时将脏页刷新到磁盘中)，其中通过一个怎样的频率，从哪里，一次取出多少页,什么时间触发(记录最近一次同步的位置，只对Checkpoint后的重做日志进行恢复，这大大缩短了恢复时间)

**Master Thread Checkpoint**

 **FLUSH_LRU_List Checkpoint**  为了保证LRU列表中有100左右空闲页，在查询的时候如果没有则将LRU尾端的页移除，如果此页时脏页则就需要进行Checkpoint

**Async/Sync Flush Checkpoint**  指的是重做日志不可用情况,将脏页刷新到磁盘中，此脏页从Flush list选取 是已经写入到重做日志的LSN记为redo_lsn，已经刷新到磁盘最新页的LSN记为checkpoint_lsn checkpoint_age = redo_lsn-checkpoint_lsn

其中checkpoint_age就是表示重做日志中还有多少页的数据没有同步过去
 async_water_mark = 75%*total_redo_log_file_size
 
 sync_water_mark = 90%*total_redo_log_file_size
 
 checkpoint_age<async_water_mark不做任何动作
 async_water_mark < checkpoint_age < sync_water_mark 异步从Flush list选取一定量的页刷新到磁盘中 直到 checkpoint_age < async_water_mark
 checkpoint_age > sync_water_mark Sync Flush Checkpoint 同步进行刷新

 **Async Flush Checkpoint**异步仅仅会阻塞发现问题(重做日志不可用)的查询线程，而Sync Flush Checkpoint会阻塞所有查询线程，在InnoDB 1.2x(MySQL5.6后)会单独放在Pager Cleaner Thread故不会阻塞用户线程

 **Dirty Page too much Checkpoint** 当缓冲池中脏页太多是就会强制进行Checkpoint 可以通过innodb_max_dirty_pages_pct进行设置 默认75 表示如果脏页占据量为整个LRU总的75%则触发Checkpoint

###  Master Thread工作过程

  该线程由各个循环loop组成， 有主循环，background后台循环， flush循环，suspend挂起循环，并在这几个循环之间进行切换
  主要做的事情是：同步脏页(在InnoDB1.2x会单独成一个线程 Page Cleaner Thread减轻了主线程的工作，提高了并发性)、重做日志缓冲到磁盘中、合并插入缓冲、删除无用的undo日志页，其中在各个循环里会判断当前系统I/O情况 刷新不同页数到磁盘中。
  
  show engine innodb status \G;

srv_master_thread loops:2188 1_second,1537 sleeps, 218 10_seconds, 2 background, 2 flush

主线程的主循环运行类2188次，此数是根据定时器1s运行一次，一共过了2188s理论上运行了2188次主循环，但是由于循环体内需要处理事情，导致时间上那时刻只有运行了1537次，所以通过此差值可以看出数据库的负载情况。1537/2188 比值越小说明负载越高，为1时负载越小。

# InnoDB关键特性

## 插入缓冲

### 1、Insert Buffer 插入缓冲(缓存对象是 不唯一的辅助索引)

  使用插入缓冲条件： 辅助索引并且索引不唯一

 在InnoDB存储引擎中要有主键，最好是自动增长，这样在插入的时候就避免了随机访问离散的页，找到该主键值在索引中合适的位置，直接在其后追加，数据页中的记录也是按照主键排序的。但是当主键为UUID时，插入的值就是随机，这个时候就会每次都会从B+ Tree根结点开始找到合适的位置插入到叶子节点中，这个时候每经过一个非叶子节点就会访问一次页(可能就会从磁盘中加载一次页)。 同理，辅助索引的键值也是随机的，因此为了提高插入性能，所以使用Insert Buffer 存放插入辅助索引key值， 插入时首先判断是否在缓冲中，若存在则直接插入到指定位置，否则放入到Insert Buffer中，后面使用线程对插入缓存进行合并(同个页面的操作放在一起)，这样大大较少了对离散页的请求。

 show innodb engine status\G

7888 inserts 3000 merge recs 1888 merges. 其中 merges代表合并的次数实际读取页的次数，merge recs代表合并插入记录数量

### 2、Change Buffer

Change Buffer是对Insert Buffer一次升级 支持DML操作 Insert Delete  Update 对应于 Insert Buffer Delete Buffer Purge Buffer
对于更新有两个过程 1、标记为已删除 delete mark 对应于Delete Buffer 2、真正删除 其Purge Buffer对应于Update第二个过程

### 3、内部实现

 从MySQL5.5后innodb使用一个 Insert Buffer 之前是每个表对应于一个Insert Buffer。

 Insert Buffer内部结构是个B+ Tree 

 非叶子节点结构： space id + marker + page_no，由于所有表共用一个Insert Buffer 所以使用 space id唯一标识一个表，其中 page_no对应于插入辅助索引记录实际对辅助索引文件中的页号,结构如图： 其中offset是 页的偏移量实际上就是页的唯一标识
 ![props](http://dymdmy2120.github.com//static/2018-03-images/insert-buffer-searchkey.png)


 叶子节点结构： space id + marker + page_no + metadata + index record 叶子节点存放的就是插入的具体辅助索引记录，其中该叶子节点存放的都是 space id +page_no相同的(也就是同一个页)辅助索引key的值。结构如图：
 
 ![props](http://dymdmy2120.github.com//static/2018-03-images/insert-buffer-leaf.png)

 所以对同个页面的操作都放在同个叶子节点中(相当于页)，当辅助索引从磁盘读取到缓存池中，或者通过Insert Buffer Bit页追踪到该Insert Buffer中辅助索引无可用空间时，再就是Master Thread都会进行 Merge Insert Buffer

**Ibuff Bitmap page**

 ![props](http://dymdmy2120.github.com//static/2018-03-images/insert-buffer-bitmap.png)
 ![props](http://dymdmy2120.github.com//static/2018-03-images/Ibuf-bitmap.png)


## Double write

>   为什么需要两次写？

>为了保证数据页的可靠性。如果在刷新16K脏页到磁盘的过程中，这个时候服务器岩机了，只写入了4K可能会想到redo重做日志，如果刚好页因为同步重做日志缓冲到日志文件时出现故障，则此时日志文件也是不完整的。


>解决方案：

>因此在缓冲中会有个doublewrite buffer 大小为2M会使用 memcpy将脏页拷贝到此缓冲中，此页是连续的(目的是为了保证刷新到磁盘中是连续的)，在磁盘中对应也有个2M的共享表空间。 其中doublewrite buffer分成两个1M，调用fsync 写入到共享共享空间后，最后将其buffer中页写入各个表空间文件中，此时是离散的。

恢复的时候，从共享空间找到该页的一个副本，将其复制到表空间文件，然后再执行重做日志。

## 自适应哈希索引

  例如对多次使用 where a=1这种查询建立一个哈希索引，哈希查询的复杂度为O(1) ，但是对于范围查询不能建立哈希索引


## 异步IO

AIO相对的是Sync IO ，当一个SQL查询语句可能需要扫描多个索引的时候，如果是AIO则会同时发起多个IO请求，然后等待所有IO完成，而Sync IO首先发起一个IO请求然后等待完成后又发起一个IO请求，当然AIO效率更高。

IO Merge，可以将多个IO合并成一个IO 例如需要访问的 （space,page_no) space为表id page_no为页id (8,6) (8,7) (8,8) AIO会判断请求的页是连续的，此时会将其三个IO合并一个IO 从（8，6）页开始读取48KB的页

## 刷新邻接页

当刷新脏页时，会判断邻近区的页是否有脏页，若有则一起刷新

# 启动、关闭与恢复

在关闭时会根据 innodb_fast_shutdown的不同的参数值，做出不同的行为(一些多脏页或insert buffer数据同步)
例如0 表示在关闭之前 Innodb完成full purge 和 merge insert buffer 并且把所有的脏页刷新回磁盘
2表示不进行 full purge 等操作只需要将重做缓冲刷新到日志文件中，但是当下次启动时会根据redo日志进行恢复

根据 innodb_force_recovery不同的设置值，恢复的行为也会不同。

