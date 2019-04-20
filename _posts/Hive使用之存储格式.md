# Hive存储格式

官网有很详细的解释

https://cwiki.apache.org/confluence/display/Hive/FileFormats



Hive支持以下几种存储格式:

```text
Text File
SequenceFile
RCFile
Avro Files
ORC Files
Parquet
Custom INPUTFORMAT and OUTPUTFORMAT
```

Hive默认的存储格式为Text File 可以使用hive.default.fileformat来更改

![](https://s2.ax1x.com/2019/04/17/AzVnx0.png)

---


# 行式存储和列式存储


## 行式存储

传统的关系型数据库，如 Oracle、DB2、MySQL、SQL SERVER 等采用行式存储法(Row-based)，在基于行式存储的数据库中， 数据是按照行数据为基础逻辑存储单元进行存储的， 一行中的数据在存储介质中以连续存储形式存在。

![](https://s2.ax1x.com/2019/04/17/AzZqtH.png)

假设  一个数据副本有四列五行的数据对应于HDFS里，他们就是一个block。这个block呈红框标注的形式

**行式存储的特点**

- 每一行对应的列都在同一个block中
- 适合随机增改删查操作
- 在查询少量列的数据时,它会一行一行读取数据，如select A,B from xxx; 这里并不需要C、D两列，但是也会读取进来，这样会造成IO提升
- 不方便压缩，每一列的数据类型都可能不一样，那么在压缩的时候，每种数据类型的压缩比不同，无法压缩至最好的效果

## 列式存储

列式存储(Columnar or column-based)是相对于传统关系型数据库的行式存储(Row-based storage)来说的。简单来说两者的区别就是如何组织表：



![](https://s2.ax1x.com/2019/04/20/EC2MvT.png)

以上图为例,假设把AB两列存储在一个块里，C、D分别存储在一个块里

这样做的好处是

- 同一列的数据，数据格式一定相同，这样方便采用高比例的压缩
- 当执行查询以列为条件的SQL时会大大的减少IO消耗，如SELECT A,B FROM XXX;

这样做也有缺点:

- 选择完成时，被选择的列要重新组装(数据重组)
- 查询的数据量大时，可能读取更多的Block



# TextFile

TextFile 是Hive表的默认存储格式，可以使用<kbd>set hive.default.fileformat</kbd> 来更改



# SequenceFile

**创建SequenceFile表**

```sql
hive> create table page_views_seq(
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
) row format delimited fields terminated by '\t'
stored as sequencefile;
hive> load  data local inpath '/root/page_views.dat'  into table page_views_seq;
```

报错:

<font size=3 color=red>Failed with exception Wrong file format. Please check the file's format.</font>

<font size=3 color=red>FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.MoveTask</font>

原因:

创建的表是sequencefile,而原始的数据时一个文本格式的数据，直接从文本导到sequencefile是不行的

解决: 借助临时表导入

```sql
hive> create table page_views(
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
) row format delimited fields terminated by '\t';
hive> load data local inpath '/root/page_views.dat' into table page_views;
hive> insert into table page_views_seq select * from page_views;
```

***在HDFS中查看大小***

```shell
[root@hadoop001 ~]# hadoop fs -ls -h /user/hive/warehouse/page_views/
Found 1 items
-rwxr-xr-x   1 root supergroup     18.1 M 2019-04-17 00:58 /user/hive/warehouse/page_views/page_views.dat
[root@hadoop001 ~]# hadoop fs -ls -h /user/hive/warehouse/page_views_seq/
Found 1 items
-rwxr-xr-x   1 root supergroup     19.6 M 2019-04-17 01:01 /user/hive/warehouse/page_views_seq/000000_0
```

查看到sequencefile 表的大小比原来的数据表还要大。

下图是sequencefile的结构，可以看出它存储的东西明显比TextFile多了一些，所以生产环境不用Sequencefile

![](https://s2.ax1x.com/2019/04/20/ECRdFs.jpg)

# RcFile

RCFile（Record Columnar File）存储结构遵循的是“先水平划分，再垂直划分”的设计理念，这个想法来源于PAX。它结合了行存储和列存储的优点：首先，RCFile保证同一行的数据位于同一节点，因此元组重构的开销很低；其次，像列存储一样，RCFile能够利用列维度的数据压缩，并且能跳过不必要的列读取。

![RCFile](https://s2.ax1x.com/2019/04/20/ECIrlR.jpg)

上图是RCFile的结构，可以看出，它只是把一列数据转换成以Row Group形式的行数据，所以它并不是真正意义上的列式存储。RCFile的性能很低，下面是测试

```sql
hive> create table page_views_rc(
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
) row format delimited fields terminated by '\t'
stored as rcfile;
hive> insert overwrite table page_views_rc select * from page_views;
```
在HDFS中查看大小
```shell
[root@hadoop001 ~]# hadoop fs -du -s -h /user/hive/warehouse/page_views_rc/
17.9 M  17.9 M  /user/hive/warehouse/page_views_rc
[root@hadoop001 ~]# hadoop fs -du -s -h /user/hive/warehouse/page_views
18.1 M  18.1 M  /user/hive/warehouse/page_views
```

# ORC

[官网描述](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+ORC)

ORC是一种优化过后的RCFile。据官方文档介绍，这种文件格式可以提供一种高效的方法来存储Hive数据。它的设计目标是来克服Hive其他格式的缺陷。运用ORC File可以提高Hive的读、写以及处理数据的性能。

ORC File包含一组组的行数据，称为stripes，除此之外，ORC File的file footer还包含一些额外的辅助信息。在ORC File文件的最后，有一个被称为postscript的区，它主要是用来存储压缩参数及压缩页脚的大小。
在默认情况下，一个stripe的大小为250MB。大尺寸的stripes使得从HDFS读数据更高效。

在file footer里面包含了该ORC File文件中stripes的信息，每个stripe中有多少行，以及每列的数据类型。当然，它里面还包含了列级别的一些聚合的结果，比如：count, min, max, and sum。下图显示出可ORC File文件结构：

![](https://s2.ax1x.com/2019/04/20/ECbkrR.png)

从上图我们可以看出，每个Stripe都包含index data、row data以及stripe footer。Stripe footer包含流位置的目录；Row data在表扫描的时候会用到。

index data包含每列的最大和最小值以及每列所在的行。行索引里面提供了偏移量，它可以跳到正确的压缩块位置。具有相对频繁的行索引，使得在stripe中快速读取的过程中可以跳过很多行，尽管这个stripe的大小很大。在默认情况下，最大可以跳过10000行。拥有通过过滤谓词而跳过大量的行的能力，你可以在表的 secondary keys 进行排序，从而可以大幅减少执行时间。比如你的表的主分区是交易日期，那么你可以对次分区（state、zip code以及last name）进行排序。

**测试**

***使用压缩***

```sql
hive> create table page_views_orc(
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
) row format delimited fields terminated by '\t'
stored as orc;
hive> insert overwrite table page_views_orc select * from page_views;
```
查看大小
```shell
[root@hadoop001 ~]# hadoop fs -du -s -h /user/hive/warehouse/page_views
18.1 M  18.1 M  /user/hive/warehouse/page_views
[root@hadoop001 ~]# hadoop fs -du -s -h /user/hive/warehouse/page_views_orc
2.8 M  2.8 M  /user/hive/warehouse/page_views_orc
```

***不使用压缩***

```sql
hive> create table page_views_orc_none_compress(
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
) row format delimited fields terminated by '\t'
stored as orc
tblproperties("orc.compress"="NONE");

```
查看大小

```shell
[root@hadoop001 ~]# hadoop fs -du -s -h /user/hive/warehouse/page_views
18.1 M  18.1 M  /user/hive/warehouse/page_views
[root@hadoop001 ~]# hadoop fs -du -s -h /user/hive/warehouse/page_views_orc
2.8 M  2.8 M  /user/hive/warehouse/page_views_orc
[root@hadoop001 ~]# hadoop fs -du -s -h /user/hive/warehouse/page_views_orc_none_compress
7.7 M  7.7 M  /user/hive/warehouse/page_views_orc_none_compress
```

# Parquet

[官网描述](https://cwiki.apache.org/confluence/display/Hive/Parquet)

**测试**

***不使用压缩***

```sql
hive> create table page_views_parquet
stored as parquet
as select * from page_views;
```

查看大小

```shell
[root@hadoop001 ~]# hadoop fs -du -s -h /user/hive/warehouse/page_views
18.1 M  18.1 M  /user/hive/warehouse/page_views
[root@hadoop001 ~]# hadoop fs -du -s -h /user/hive/warehouse/page_views_parquet
13.1 M  13.1 M  /user/hive/warehouse/page_views_parquet
```



***使用压缩***

```sql
hive> set parquet.compression=GZIP;	
hive> create table page_views_parquet_gzip
stored as parquet
as select * from page_views;
```

查看大小

```shell
[root@hadoop001 ~]# hadoop fs -du -s -h /user/hive/warehouse/page_views
18.1 M  18.1 M  /user/hive/warehouse/page_views
[root@hadoop001 ~]# hadoop fs -du -s -h /user/hive/warehouse/page_views_parquet_gzip
3.9 M  3.9 M  /user/hive/warehouse/page_views_parquet_gzip
```

