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

## 

# 行式存储和列式存储

## 行式存储

### 定义

传统的关系型数据库，如 Oracle、DB2、MySQL、SQL SERVER 等采用行式存储法(Row-based)，在基于行式存储的数据库中， 数据是按照行数据为基础逻辑存储单元进行存储的， 一行中的数据在存储介质中以连续存储形式存在。

![](https://s2.ax1x.com/2019/04/17/AzZqtH.png)

假设  一个数据副本有四列五行的数据对应于HDFS里，他们就是一个block。这个block呈红框标注的形式

**行式存储的特点**

- 每一行对应的列都在同一个block中
- 适合随机增改删查操作
- 在查询少量列的数据时,它会一行一行读取数据，如select A,B from xxx; 这里并不需要C、D两列，但是也会读取进来，这样会造成IO提升
- 不方便压缩，每一列的数据类型都可能不一样，那么在压缩的时候，每种数据类型的压缩比不同，无法压缩至最好的效果

## 列式存储

