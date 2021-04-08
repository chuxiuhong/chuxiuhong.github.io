
## 起因

最近在实际的项目中需要把300W+的数据换到新数据库里（原因是SQL Server的存储过于废），与此同时，数据源还在源源不断地生成新数据。考虑到查询性能，除了加索引，第一反应就是该分表了。

既然决定分表，就开始考虑用什么方法来做。如果按照之前的传统方式，可能需要复杂的触发器或者利用一些Java中间件，不过这两者都会引入附带的复杂性。搜集资料过程中发现PostgreSQL的新版本是直接支持分表的，实际使用中发现PostgreSQL 13.2确实很好用，下面用文档中的例子来讲一下应用。

## 分表

### 定义表

```sql
CREATE TABLE measurement(
    city_id   int not null,
    logdate   date not null,
    peaktemp  int,
    unitsales int
) PARTITION BY RANGE(logdate)
```

这里是设计利用`logdate`的值来分表，要额外注意的是通常需要把分表依据的列当做主键的一部分。

### 设定分区规则

按前述建立表后数据库只知道要按照`logdate`来分表，但是不会知道该按照什么规则来分，每个表分多少。这部分需要写出定义

```sql
CREATE TABLE measurement_y2006m2 PARTITION OF measurement FOR VALUES FROM ('2006-02-01') TO ('2006-03-01');
CREATE TABLE measurement_y2006m3 PARTITION OF measurement FOR VALUES FROM ('2006-03-01') TO ('2006-04-01');
.....
```

这部分我在项目中是用Python写了一个生成脚本，按照月份分表，直接生成10年的语句（手写十有八九要写错）。这里要注意的是除了语句生成的正确性还要考虑分区开头之前和分区末尾以后的数据。比方说如果有1970年或者2150年的异常数据，按照这种方法就存不进去，因为没有存储他们的规则。所以尽量要增加一个`history`分区表、一个`future`分区表，用以存储这些奇葩的数据。

做完分区表后对比一下查询性能，比原来用SQLServer不加索引的情况大约提升10倍的查询速度，效果非常好。

### 善后设置

```sql
CREATE INDEX ON measurement (logdate);
```

如果前面把时间加到主键中的一部分的话，加索引这一步就不用做了

同时注意`postgresql.conf`文件中的`enable_partition_pruning`不能为off，否则不会用分区。

同时要注意生成出的分区表并不在你主表的schema里，如果不限定schema，会生成到public这个schema里。主表和分区表都可以查到数据，分区表只能查到当月（符合条件）的数据。



## 其他的小细节

1. `PostgreSQL`安装后的`pgAdmin`真的好用
2. 如果远程连不上，很可能是防火墙的问题。否则要找`pg_hba.conf`配置文件，在里面加上一行`host all all 0.0.0.0/0 md5`
3. 管理数据库工具强烈建议使用`Datagrip`，我对比`Navicat`和`Datagrip`，感觉后者无论功能还是速度，都比前者要强一些。而且在表格的数据操作上也是更加便捷