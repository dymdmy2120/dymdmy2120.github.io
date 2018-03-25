---
layout: post
title: "mysql之文件解析"
date: 2018-03-18
categories: 数据库
excerpt: MySQL内的文件和Innodb文件
---

* content
{:toc}

# 参数文件

参数有参数名和对应参数值，在MySQL启动时会加载一个配置文件 mysql.cnf一般在mysql安装目录下的conf中，里面配置了参数，如果没有指定则使用MySQL默认的配置

分静态和动态的，静态的只能读不能修改的
可以通过 show variables like '%innodb%' 查询参数
可以设置全局的 set @@global.param=value(重新打开一个终端时的会话的参数值会改变，但是当前会话值还是旧的)可以通过 select @@global.innod\_buffer\_pool_size 设置会话 set @@session.param=value (只是作用于当前会话)，这两种设置的值，在MySQL重启后,设定的参数不会保存，除非在my.cnf文件中配置

#MySQL内的日志文件

##错误日志

可以通过 show variables like '%error%'查看错误日志路径在哪里，可以通过错误日志信息得知启动失败原因，以根据一些警告日志找到优化点。


##慢查询日志

记录一些查询时间>10s的sql语句 这个10s可以通过 long\_query\_time配置阈值，其中可以使用long\_query\_io配置执行某SQL语句进行逻辑IO读取的次数(访问缓冲池和磁盘)>100时记录到慢查询日志中。
 
 通过 slow\_query\_type设置类型 1表示只记录超过一定时间的SQL 2表示只记录逻辑访问次数大于一个阈值的SQL， 3表示两种都有。
 
 可以通过 mysqldumpslow slow.log 进行查看,如果查看执行时间最长的10条SQL语句：mysqldumpslow -s al -n 10 slow.log 
  其中为了方便查询和直观，可以将慢查询日志放入mysql数据库中的slow\_log表中，存储引擎使用的MyISAM。其中通过log_output参数进行设置为File表示使用文件存储，TABLE是使用表来存储。


##查询日志

记录一些未能正确执行的SQL语句，其中将日志信息放入到数据库mysql中的general_log中


##binlog

每次对数据库更改操作都会记录到binlog中，并不是每次的更改都会马上写入到日志文件中，而是等到该事务提交时再刷到磁盘上(为了保证数据一致性，有可能事务回滚)，每个会话(每个线程)开启了一个事务就会在内存中开辟空间默认是32K可以通过binlog_cache_log进行配置大小，此值不能设置太大，因为每个线程都需要分配，但是不能太小，如果太小了会将数据写入到临时文件中。

可以通过show global status likie '%binlog_cach%' 其中binlog_cahce_use：使用缓冲写二进制日志的次数，binlog_cache_disk_use：使用临时文件写入binlog文件的次数 从而得知当前binlog_cache_size设置的是否合理。

**binlog_format格式**：默认使用statement 新版本的Innodb使用的是ROW 日志记录的是具体到行的信息但是会消耗大量的磁盘空间

**作用：**可以用来在主库可从库同步数据的工具，还可以用来恢复(point-in-time指定恢复的某个时间段)

**binlog日志查看：**mysqlbinlog -vv --start-position=106 test.0008

 **思考：**默认是二进制文件不是每次写入到磁盘上，而是放入到缓冲中，但是当岩机时会有一部分数据没有写入到binlog中，这给恢复和复制带来问题 sync_binlog=1表示不使用操作系统的缓冲，而是直接写入到二进制日志中,但是也有个问题就是如果在提交事务之前刚好岩机，那么事务没有提交，重启时会回滚事务，但是这些操作都已经记录在binlog中，因此为了保证binlog和InnoDB存储引擎数据文件的同步，需要进行设置 innodb_support_xa=1。如果当前数据库担任slave角色，则不会从master取得并执行的二进制文件写入自己的二进制日志文件中去，如果需要写入则设置 log_slave_updates，若要搭建master->slave->slave必须设置该参数。


#套接字文件

就是本地连接数据库的一种unix域套接字方式，需要一个socket文件，放在/temp/mysql.soc

#PID文件

每次启动MySQL进程时，会将pid写入到一个pid文件中
show variables like '%pid_file'\G; 可以查看pid文件在哪里，一般在／mysql/data下以主机名命名.pid结尾文件

#表结构空间

无论采用何种存储引擎，每个表都会有一个 表名.frm文件，存放表的结构信息。frm也可以存放视图的定义。

#InnoDB文件
##表空间文件
innodb存储数据的文件，分为共享表空间(存放所有表的数据) 又ibdata1和ibdata2两个文件组成的，默认为10M，然后再追加。用户可以不用将所有的数据放在共享表空间中，可以通过 innodb_file_per_table ON 设置成每张表的数据放在独立表空间，存放在ibdata文件中。其中文件命名是  表名.ibdata 此文件中仅仅存放该表的数据、索引、插入BITMAP等信息，其余的信息还是放在共享表空间中。

![props](http://dymdmy2120.github.com//static/2018-03-images/innodb-table-file.png)

##redo 重做日志文件

innodb之所以支持事务，原理实现就是利用redo文件，每次在执行更改数据之前先记录到redo文件中，该文件不能太大否则恢复时间很长，当然也不能太小否则会导致 async checkpoint(redo_lsn_checkpoint > 75*redo_log_size)将脏页中数据频繁的刷新到磁盘上，性能的抖动，此时阻塞用户线程。

**redo日志和binlog区别：**

1、redo日志是存放innodb存储引擎本身事务日志，而binlog是与MySQL数据库相关的日志记录包括MyISAM、InnoDB等其他存储引擎的日志。 

2、内容不一样 无论是binlog的格式是 statement、row、mixed 存放的是一个事务的具体操作内容。而redo重做日志记录的每个页Page的更改情况。

3、写入的时间也不一样，binlog是在事务提交前提交，即只写入一次磁盘 不论事务多大。 在事务进行中不断有redo重做日志写入到重做日志文件中。


日志用户组中可以有多个redo日志文件，为了保证可用性可以使用镜像日志组(将日志存放在多台服务器磁盘中)。一般1个日志用户组，组中有2个日志文件，不使用镜像日志组，redo日志文件是循环使用的，例如有ib_logfile0 ib_logfile1 先存放在ib_logfile0->ib_logfile1->ib_logfile0这样循环使用。

![props](http://dymdmy2120.github.com//static/2018-03-images/log-group.png)

**redo重做日志文件结构**

![props](http://dymdmy2120.github.com//static/2018-03-images/redo-stru.png)


除了在主线程中每秒刷新redo缓冲日志到文件中，还可以通过 innodb_flush_log_at_trx_commit进行控制提交事务时刷新一次。 0提交事务不刷新，1提交事务 fsync同步到磁盘中 2表示将数据异步刷新到磁盘中，即写入操作系统缓存中，但不一定保证commit时一定写入磁盘中。因此为了保证ACID 中持久性需要设置成1，也就是当事务提交时就必须确保事务已经写入重做日志文件中，当数据库岩机时设置0或2 都有可能发生恢复时丢失数据，不同的是如果设置2 操作系统和机器没有岩机的话，由于未写入磁盘中的事务日志保存在文件系统缓存，恢复时同样保证数据不丢失。