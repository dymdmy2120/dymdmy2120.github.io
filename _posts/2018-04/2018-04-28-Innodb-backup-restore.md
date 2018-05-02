---
layout: post
title: "mysql备份和恢复"
date: 2018-04-28
categories: 数据库
excerpt: 数据库的备份和恢复
---

* content
{:toc}




# 备份



## 1、冷备（备份共享文件数据）

就是将 frm 共享表空间、*.ibd、重做日志文件备份起来，下次恢复时直接将恢复路径指定到改文件目录下就可以，恢复速度快无需重建索引、不需要执行SQL语句，由于存放像redo undo log数据导致备份文件比逻辑文件大。而且不能轻易的跨平台，和跨数据库。

## 2、逻辑备份（SQL复制）

dump
mysqldump --all-database > dump.sql 

mysqldump --database db1 db2 db3 > dump.sql

mysaldump 参数

--single-transaction 备份开始使用START TRANSACTION命令，只适用于InnoDB存储引擎，保证在备份数据时没有其他DDL语句执行，保证数据的一致性

-lock-tables(-l)  这个适用于MyISAM存储引擎，备份数据时会对依次锁住每个架构下所有表，其中只能读取数据，当该架构中既有InnoDB存储引擎的表又有MyISAM存储引擎的表，这是只需要-lock-tables就可以了，由于--single-transaction是互斥的。 -l只能保证每个架构下表的数据是一致性的，但是不能保证所有架构下表备份数据一致性。

--lock-all-tables(-x) 可以解决-l带来的问，当备份数据时,对所有数据库中的所有表加锁。

--add-drop-database 就是在导出的sql语句中会加上 CREATE DATABASE IF NOT EXISTS

select ... into outfile from t  

只是对某张表的数据进行拷贝到另外一个文件中，字段与字段之间默认使用\t TAB制表符，同时可以自己设置 select * into outfile 'a.txt' fields terminated by ',' from a


# 恢复

## 1、逻辑备份的恢复

mysql -uroot -proot < backup.sql 进行恢复

或 使用 source backup.sql 


LOAD DATA INFILE

对上面使用 select * into outfile 'a.txt' from a 导出到文件的数据或使用mysqldump-tab 进行恢复，这个仅仅对单表 

load data  infile 'a.txt'  into table a ignore 1 lines

mysqlimport

其实和load data infile语法类似，但是可以并发导入多个表文件

mysqlimport --user-thread=2 test(db_name) a.txt b.txt


SHOW FULL PROCESSLIST

查看数据库中当前执行的线程


## 2、二进制日志备份与恢复

主从数据库的同步是通过 binlog 完成的

InnoDB存储引擎服务器的配置

[mysqld]

log-bin=mysql-bin
sync_binlog = 1 //当事务提交之前会将日志持久到磁盘中，避免丢失
innodb_support_xa = 1 //保证二进制日志和innodb存储引擎中文件数据保持一致，有可能将数据持久到binlog文件时，刚好岩机了事务并没有提交，重启时回滚了已经操作的数据，而使用 inndo_support_xa保证binlog和undo log redo log数据的一致性

恢复多个binlog

mysqlbinlog binlog.[0-10]* | mysql -uroot -proot test(db_name)

恢复时可以指定 开始位置和结束位置  mysqlbinlog --start-position binlog.100 | mysql -uroot -proot test(db_name) 
--start-datetime  --stop-datetime 可以指定二进制文件的某个时间点进行恢复


## 3、热备

在数据库运行过程中，不阻塞其他DDL操作完成数据库的数据备份操作

ibackup是InnoDB存储引擎官方提供的热备工具，可以同时备份MyISAM和InnoDB存储引擎表

对于InnoDB热备的工作原理：

1) 备份开始时记录重做日志检查点 checkpoint的LSN1

2) 复制共享表空间和独立表空间

3) 复制完表空间后，再一次记录重做日志文件检查点的LSN2

4) 复制在备份过程中产生的重做日志(读取LSN2-LSN1之间的重做日志数据)


优点：不阻塞任何SQL语句，跨平台、备份性能好(备份的实质是复制数据库表空间文件和重做日志)，支持压缩备份

恢复步骤：

恢复表空间

应用重做日志文件(重新再执行在备份期间产生的重做日志数据)


XtraBackup

由于ibackup需要收费，而XtraBackup是免费，可以实现了ibackup所有的功能

XtraBackup实现增量备份

1) 首先完成一个全备，并记录此时的LSN

2) 在进行增量备份时，比较表空间中的每个页的LSN是否大于上次增量备份时的LSN，如果是则备份该页，同时记录当前检查点LSN


## 4、快照备份

文件系统支持的快照功能对数据进行备份

## 5、复制

![props](http://dymdmy2120.github.com//static/2018-04-images/replication1.png)

为了保证 主从数据的同步，可以通过binlog复制数据到从服务器中

1) 主服务(master) 把数据更改记录到二进制日志中，并且推送到其他从库中

2) slave从服务接受到bin log复制到自己的中继日志中(relay log) 

3) 读取服务器的中继日志，将更改应用到自己的数据库中，保证数据的一致性

异步实时的，如果主库负载大的话，主从的同步就会被延时，想要查看延迟的情况可以通过

show  slave status 和 show master status进行查看


主库中的字段 Position - 从库中的字段 Read_Master_Log_Pos 若差值越大表明延时越大

**快照+复制**

架构图：

![props](http://dymdmy2120.github.com//static/2018-04-images/replication2.png)

如果在主库中执行 DROP DATABASE或DROP TABLE 然后从库通过binlog跟着执行，导致从库中的数据也丢失了，因此用户不能进行恢复数据，所以为了避免这个风险，需要在从库中所在的分区做快照，直接从服务上的快照进行恢复。所以有上面的架构图。

此外可以在从库中 设置
[mysqld]
read-only 如果操作从服务器的用户没有SUPER权限，则对从服务器进行任何修改会抛出一个错误。