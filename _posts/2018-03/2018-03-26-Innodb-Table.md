---
layout: post
title: "mysql表的剖析"
date: 2018-03-26
categories: 数据库
excerpt: InnoDB存储数据的数据结构
---

* content
{:toc}

> 
>  下面主要分析InnoDB存储引擎中最核心的部分--表，表是存储数据的集合，分为逻辑结构和物理结构：
>  
>  **逻辑结构**：就是在逻辑上将表进行拆分成段、区、页，目的是为了更好的管理数据。
>  
>  **物理结构**：具体到数据是是如何存放的，以及页的数据结构。
>  
> 当然也会从数据完整性约束、视图和物化视图、分区表以及子分区进行全面认识表在InnoDB的作用。


# InnoDB表的逻辑存储结构

## 一、索引组织表

InnoDB存储引擎中都是根据主键的顺序组织存放数据的，如果创建表时没有定义主键时首先会看有没有非空唯一索引，若有则选择(如果有多个呢？其按照定义索引顺序选择第一个作为主键)，否则自动会生成一个6字节ROW_ID作为主键。



## 二、逻辑存储结构

表空间由多个段 Segment(叶子节点段、数据段、回滚段)组成，其中段又由多个区(Extent)组成，每个区中包含64个页，在MySQL中每个页(Page)可以存放16K数据。

以下为InnoDB逻辑结构：

![props](http://dymdmy2120.github.com//static/2018-03-images/innodb-logical-str.png)



## 三、表空间

数据都放在一个空间中，这个空间称为表空间(tablespace)，其实也就是一个存放数据的文件，其中包括共享表空间和独立表空间。可以通过一个参数配置innodb_file_per_table 设置ON就会将数据，索引和插入缓冲Bitmap数据页存放在 表名.ibd中，其他一些插入缓冲索引页，undo回滚信息，二次写入缓冲等信息还是会放在共享表空间的，当然如果其独立空间不断增大，共享空间也会增大。 否则所有数据都会放在其共享表空间中 ibdata1中。

> 对于上面使用了独立表空间时也会随着其增大，共享空间也会不断增大的情况举个例子：
> 
> 可以设置set autocommit=0进行自动提交，在commit之前不断进行update t set a = 0时，其共享表空间中的undo回滚数据不断增多，哪怕rollback了 其共享表空间大小仍然没有变小，其中在主线程中会每隔10s中进行full purge清除无效的undo页，这个清除只是标记可用以便下次重复利用。 当用户再执行一次update操作，其共享表空间不会增大。

## 四、段

表空间中包含多个段，数据段、索引段、回滚段，其中数据段为B+ Tree的叶子结点，索引段为非叶子结点

## 五、区

一个段中有多个区组成的，一个区默认会有64个Page 一个Page=16K 所以一个区可以存放1M的数据。InnoDB为了保证页的连续性，一次申请4~5个区。如果开启了innodb_file_per_table 一开始其.ibdata文件怎么只有96K? 应该只有1M，其实为了节省磁盘空间，一开始只分配6个页，后面使用完后再申请连续的64个页。对于undo这类段可以节省部空间。

## 六、页

在InnoDB有页的概念也可以称为块，默认一个页可以存放16KB数据在InnoDB1.2x支持对页进行修改 innodb_page_size，如果修改了那么当前数据库的表中的页都是以innodb_page_size进行分配的，不能对其进行修改，除非创建一个新的库。

页的类型有：

1、数据页(B+ Tree叶子结点)

2、索引页(B+ Tree非叶子结点)

3、undo页(undo log page)

4、系统页(System page)

5、事务数据页(Transaction system page)

6、插入缓冲位图页(Insert Buffer Bitmap page)

7、插入缓冲空闲列表页(Insert Buffer Free List)

8、未压缩的二进制大对象页(Uncompressed BLOB page)

9、压缩的二进制大对象页(Compressed BLOB Page)

## 七、行

其中每页中包含多行记录，InnoDB是面向行存储的，其中一个页规定了最多存放 16KB/2-200=7992行数据，200个字节为和页信息相关的头部信息数据，这里为什么16KB/2有待探索？

# InnoDB行记录格式

InnoDB中有两种存放行记录的格式：Compact压缩型的和Redundant冗余型的

## 一、Compact

Compact格式使得一个页中能存放更多的数据，这样就能减少磁盘I/O提高了查询效率。其中Compact格式的结构如下：

![props](http://dymdmy2120.github.com//static/2018-03-images/compact-line-record-format.png)

**其首部是一个存放可变长字段的实际长度，按照列顺序逆序放置。**

如果varchar(length) 定义的length < 255则使用一个字节存放变长列的实际长度，如果大于255 则使用2个字节存储。其中在MySQL数据库中varchar的长度定义和当前使用的编码集有关以及和现在每行的最大存放空间有关(因为一个中文字符在utf8是用3个字节表示的)

例如某个列为 varchar(100),但是实际存放数据为‘a'那么首部占用了一个字节，并且表示为1 00000001


**第二个表示列是否NULL的标志位 1个字节**

表示该行数据的列是否有NULL 有则用1表示，占用1个字节

**record header记录头信息 5个字节**

![props](http://dymdmy2120.github.com//static/2018-03-images/innodb-compact-record-header.png)


![props](http://dymdmy2120.github.com//static/2018-03-images/row_format.png)

info bits的第三位表示该行是否已被删除，如果是则标记1，没有被删除则标记0，第四位表示该记录是否是预先被定义为最小的记录，如果是则标记为1

n_owned该记录拥有的记录数，指的是该记录所在页中page diectory所属slot中拥有的记录数

record type表示记录的类型，数据行记录(叶子节点页中)0,非叶子节点页中行记录1，伪记录首记录infimum值为2，伪记录最后一个记录supremum的值为3

next record offset下一条记录的相对offset，通过这个next record offset 我们可以遍历一个页中的所有记录。记录与记录之间通过链表的形式组织

最后如果该列无论是varchar还是char类型的值为NULL时，都不会占用字节，除了存放自己定义的字段数据外，还有主键，事务id，回滚指针列

为了能更好的学习其内部数据的存储结构，可以将独立表空间文件以16进制形式导出 
hexdump -C -v test1.ibdata > test1.txt中进行查看结构如下

![props](http://dymdmy2120.github.com//static/2018-03-images/hexdump-file1.png)

其中左边为offset 代表逻辑地址

![props](http://dymdmy2120.github.com//static/2018-03-images/hexdump-file2.png)

![props](http://dymdmy2120.github.com//static/2018-03-images/hexdump-file3.png)

其中可以看的出来char类型的字段的其他部分用0x20字符也就是空格进行填充了。

注意Record Header中最后两个字节(0x002c)为指向下一条记录位置开始的指针，也就是当前的offset+0x002c=下条记录开始位置

## 二、Redundant

该行的格式是MySQL5.0之前的记录存储方式,存储结构为：

![props](http://dymdmy2120.github.com//static/2018-03-images/redundant-file.png)

![props](http://dymdmy2120.github.com//static/2018-03-images/redundant-h.png)

其中和Compact不同的是，varchar类型的字段，如果为NULL时不会占用字节，但是char类型如果NULL值会占用字节，其中也会和数据库的编码集有关，对于char(10) 列 若为 latin1字符集则占用10个字节，在utf8字符集占用30个字节。

## 三、行溢出数据

varchar(length) 其中length代表字符的长度，并不是字节数

如果定义varchar类型超出行记录允许存放最大字节数范围时就会转换到 二进制大数据Uncompressed BLOB页中存放，由于还有别的开销，所以允许定义最大varchar长度为65532个字节，如果字符集为latin1  则允许最大长度为65532 如果为utf8，由于中文字符用3个字节表示，所以最大长度为 65532/3。

其中如果定义成了varchar类型时在插入过程也可能将数据放入到BLOB页中，如果定义了BLOB类型，也不一定会放在BLOB页中，那什么时候放入BLOB页中呢？为了保证B+ Tree特性，保证页中至少需要两条记录(如果仅仅一条记录那树结构就演变成链表结构)。如果放入BLOB页，那么数据页只保存数据的前768个字节。

## 四、char类型行结构存储

在多字节字符集情况下，InnoDB将 把char类型的字段视为varchar变长类型，在存储上和varchar一样，在头部会存放char类型列实际需要存放字节数目。 例如定义 char(10) 在utf8字符集下，插入了 ‘我们’ 那么头部记录为 6个字节， 其列的值中其他4个字节用空格填充。

# InnoDB数据页结构

数据页由以下几个部分构成

1、File Header 文件头

2、Page Header 页头

3、Infimum和Supremum Records

4、User Records 这个就是上面介绍的行记录数据结构

5、Free Space(空闲空间)

6、Page Directory(页目录)

7、File Trailer(文件结尾信息)


File Header Page Header File Trailer大小固定的 38 56 8 字节，用来存放Checksum、previous page、next page、所在B+ Tree索引的层数等。

User Records Free Space Page Directory 这些空间都和实际存放的行记录有关，是动态改变的。

以下为页的数据结构：

![props](http://dymdmy2120.github.com//static/2018-03-images/page-str.png)

下面就从每个部分进行剖析：


## 一、File Header(38个字节)

![props](http://dymdmy2120.github.com//static/2018-03-images/page-file-header.png)

以下就是MySQL InnoDB使用到所有Page_Type，每一种页的类型存放不同的数据，这节讲解的页属于叶子节点，存放行记录的页。 还有非叶子节点(索引页)

![props](http://dymdmy2120.github.com//static/2018-03-images/page-type.png)

## 二、Page Header(56个字节)

![props](http://dymdmy2120.github.com//static/2018-03-images/page-header1.png)

![props](http://dymdmy2120.github.com//static/2018-03-images/page-header2.png)

## 三、Infimum和Supremum Record

Infimum Record是一行特殊的记录行，记录的值比该页的任何主键都要小，Supremum Record的记录比该页的任何主键都要大，在页创建时就已经定义了，并且永远都不会删除。

![props](http://dymdmy2120.github.com//static/2018-03-images/infimum-supremum.png)

## 四、User Record和 Free Space

User Record就是前面说的部门，在InnoDB存储引擎中的表是通过B+ Tree树索引组织的，也就是说在表的创键和插入大量数据时都是遵循B+ Tree这特性进行的，当在大量数据中能很快查找出符合条件的数据，也是通过B+ Tree一步一步定位到数据行的。

Free Space 指的是空闲列表，，也是一个链表数据结构，当删除一行数据时，会把该空间放入该空闲链表中。

## 五、Page Directory

由于在MySQL中查找某个值时，并不是可以直接定位到改值的位置，而是通过B+ Tree索引从非叶子节点一步一步到叶子节点页，定位到该值在某个叶子节点页中，然后将其加载到内存中，内部行记录通过链表组织的。一条一条记录进行匹配最终找到符合条件的值，然而InnoDB为了提高页内的行记录查找效率，就使用了Page Directory 二叉树数据结构。

Page Directory 中有多个Slots 槽位，一个槽位内包含多条行记录，记录时根据主键进行排序的，由于Page Directory时稀疏目录并没有将其全部行记录存放进来，而是通过n_owned表示该行记录前多少条记录属于这个slot管辖，然后通过next_record进行查找相关记录。其实实际上Page Dirctory缩小了查找范围，提高了查找效率。

## 六、File Trailer(8个字节)

文件结尾信息只占用8个字节，	前四个字节表示checksum，和File Header中的checksum一个意思，后四个字节表示LSN 记录的是最近同步到磁盘的最新的edo重做日志序号，其和File Header 中FILE_PAGE_LSN一个意思。 其中InnoDB为了保证页的完整性，防止在写数据时突然down机或磁盘损坏导致页信息丢失，所以每次读区页时都需要比较这两个值和File Header的两部门数据是否一致 (这里的checksum比较并不是简单的比较值而是使用checksum函数根据某种算法进行判断的)。

由于每次从磁盘读取页时都需要进行完整性检测，所以会有一定的开销，可以通过参数 innodb_checksum_algorithm进行控制 是否开启检测以及使用检测函数 checksun使用的算法。默认使用crc32 none表示不对其进行检测。

## 七、页数据结构

由上面介绍了页的几大组成部分，可以看出页与页之间使用双向链表进行连接的，结构如下：

![props](http://dymdmy2120.github.com//static/2018-03-images/page-link.png)

主要部分:

![props](http://dymdmy2120.github.com//static/2018-03-images/page-str-info.png)

![props](http://dymdmy2120.github.com//static/2018-03-images/page-full-info.png)

![props](http://dymdmy2120.github.com//static/2018-03-images/page-user-record1.png)

![props](http://dymdmy2120.github.com//static/2018-03-images/page-user-record2.png)

![props](http://dymdmy2120.github.com//static/2018-03-images/user-record-inner.png)

![props](http://dymdmy2120.github.com//static/2018-03-images/user-record-example.png)

连贯起来整体查找过程为

![props](http://dymdmy2120.github.com//static/2018-03-images/page-search.png)

简化的B+ Tree查找示意图：

![props](http://dymdmy2120.github.com//static/2018-03-images/page-simple.png)

# 约束

为了保证数据的完整性 InnoDB提供的以下约束：

Primary Key 主键约束

Unique Key 唯一键约束

Foreign Key 外键约束

DEFAULT 默认约束

NOT NULL 不为空约束

还可以通过触发器进行控制数据完整性

create trigger trigger_name [after|before] [inert|update|delete] on table_name for each row trigger statement


ENUM 和SET 约束

# 视图

视图是虚表，并没有实际存储文件数据的来源于其他物理表。视图主要是为了将关系到多个表的复杂查询sql的结果可以通过一个视图进行展示，并不不需要关心具体用到了哪几个表，起到了一个安全作用。可以通过视图进行更新数据，实际上是基本的数据，可以开启WITH  CHECK OPTION，每次更新时会检查是否满足查询的条件。

**物化视图**： 是实际存在的实表，为了提高各个表联合查询的sql语句，可以提前将结果保存到一个表中，保证下次查询会更快。MySQL数据库并不支持物化表，但是可以通过触发器来实现，每次提交一条数据时可以进行一些相关逻辑操作。

# 分区表

分区表一般使用在 OLAP(在线分析处理)中，很少使用在 OLTP(在线事务处理)， 分区的目的是将大量的数据根据某分区列，某种规则存放在不同的区文件中，这样如果直接查询某个区文件中数据会更快。

MySQL支持水平分区暂不支持垂直分区，局部分区索引，就是一个分区中既存放索引又有数据，而全局分区索引是所有数据索引存放在一个对象中。 

**水平分区：** 将不同行数据根据某个分区列某种规则存放到不同物理文件

**垂直分区：**将不同列数据根据某个分区列某种规则存放到不同物理文件


分区类型：

RANGE 现在支持 RANGE COLUMNS

LIST 现在支持 LIST COLUMNS
 
HASH 根据用户定义的表达式的返回值进行分区

KEY 根据数据库自己提供hash函数来分区 

COLUMNS 由于分区的条件是数据必须是整数，而使用RANGE LIST时都需要将非整数类型列转换成整数，而COLUMNS会自动进行转换，然后直接使用列就行。例如 RANGE COLUMNS

不管哪种分区类型，如果存在唯一键则其分区列必须是唯一键的组成部分

**子分区：**

就是在原来分区中在根据 HASH或KEY的方式进行再划分一次


**分区使用场合：**对于OLAP 仅仅对大数据表进行某个时间段扫描时，可以考虑使用分区根据时间进行将数据分到不同数据文件中。 到时对于OLTP 需要考虑清楚，分区后的查询方式有哪些?会不会因为分区后会导致更慢的查询。 举个例子：


如果对1000w的数据根据id hash到10个文件中，由于使用B+ Tree组织的数据，假如分区后执行 select * from a where PK=@pk 需要执行 2次磁盘I/O，分区前要3次磁盘I/O 那么查询效率是提高了，但是如果执行 select * from a where Key=@key 此查询根据非主键索引，最终会获取到主键id 再进行一次搜索，如果查询的主键id分别在10个不同的文件搜索总共会有 10*2=20次磁盘I/O 原来单表只需要2~3次就可以了。

