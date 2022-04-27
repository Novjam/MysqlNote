# mysql的优化

## 概念

在应用的的开发过程中，由于初期数据量小，开发人员写 SQL 语句时更重视功能上的实现，但是当应用系统正式上线后，随着生产数据量的急剧增长，很多 SQL 语句开始逐渐显露出性能问题，对生产的影响也越来越大，此时这些有问题的 SQL 语句就成为整个系统性能的瓶颈，因此我们必须要对它们进行优化.

MySQL的优化方式有很多，大致我们可以从以下几点来优化MySQL:

* 从设计上优化
* 从查询上优化
* 从索引上优化
* 从存储上优化

## 查看sql执行频率

MySQL 客户端连接成功后，通过 show \[session|global] status 命令可以查看服务器状态信息。通过查看状态信息可以查看对当前数据库的主要操作类型。

```sql
--下面的命令显示了当前 session 中所有统计参数的值
show session status like 'Com_______';  -- 查看当前会话统计结果
show global  status  like 'Com_______';  -- 查看自数据库上次启动至今统计结果
show status like 'Innodb_rows_%’;       -- 查看针对Innodb引擎的统计结果
```

![参数含义](<.gitbook/assets/image (1) (1).png>)

## 定位低效率执行SQL

可以通过以下两种方式定位执行效率较低的 SQL 语句。

1. 慢查询日志 : 通过慢查询日志定位那些执行效率较低的 SQL 语句。

```sql
-- 查看慢日志配置信息 
show variables like '%slow_query_log%’; 

-- 开启慢日志查询 
set global slow_query_log=1; 

-- 查看慢日志记录SQL的最低阈值时间 
show variables like 'long_query_time%’; 

-- 修改慢日志记录SQL的最低阈值时间 
set global long_query_time=4;

```

1. show processlist：该命令查看当前MySQL在进行的线程，包括线程的状态、是否锁表等，可以**实时**地查看 SQL 的执行情况，同时对一些锁表操作进行优化。

```
show processlist;
```

1. id列，用户登录mysql时，系统分配的"connection\_id"，可以使用函数connection\_id()查看
2. user列，显示当前用户。如果不是root，这个命令就只显示用户权限范围的sql语句
3. host列，显示这个语句是从哪个ip的哪个端口上发的，可以用来跟踪出现问题语句的用户
4. db列，显示这个进程目前连接的是哪个数据库
5. command列，显示当前连接的执行的命令，一般取值为休眠（sleep），查询（query），连接（connect）等
6. time列，显示这个状态持续的时间，单位是秒
7. state列，显示使用当前连接的sql语句的状态，很重要的列。state描述的是语句执行中的某一个状态。一个sql语句，以查询为例，可能需要经过copying to tmp table、sorting result、sending data等状态才可以完成
8. info列，显示这个sql语句，是判断问题语句的一个重要依据

## explain分析执行计划

通过以上步骤查询到效率低的 SQL 语句后，可以通过 EXPLAIN命令获取 MySQL如何执行 SELECT 语句的信息，包括在 SELECT 语句执行过程中表如何连接和连接的顺序。

示例：

```
explain select * from user where uid = 1;
```

![](<.gitbook/assets/image (5).png>)

**Explain分析执行计划-Explain 之 id**

id 字段是 select查询的序列号，是一组数字，表示的是查询中执行select子句或者是操作表的顺序。id 情况有三种:

1、id 相同表示加载表的顺序是从上到下；

2、id 不同id值越大，优先级越高，越先被执行；

3、id 有相同，也有不同，同时存在。id相同的可以认为是一组，从上往下顺序执行；在所有的组中，id的值越大，优先级越高，越先执行。

**Explain分析执行计划-Explain 之 select\_type**

表示 SELECT 的类型，常见的取值，如下表所示：

![](<.gitbook/assets/image (4).png>)

**Explain分析执行计划-Explain 之 type**

type 显示的是访问类型，是较为重要的一个指标，可取值为：

![](.gitbook/assets/image.png)

<mark style="color:red;">结果值从最好到最坏以此是：system > const > eq\_ref > ref > range > index > ALL</mark>

**Explain分析执行计划-其他指标字段**

**Explain 之 table**

显示这一步所访问数据库中表名称有时不是真实的表名字，可能是简称

**E**_**xplain 之 rows**_

扫描行的数量。

**Explain 之 key**

_possible\_keys :_ 显示可能应用在这张表的索引， 一个或多个。

_key_ ： 实际使用的索引， 如果为NULL， 则没有使用索引。

_key\_len_ : 表示索引中使用的字节数， 该值为索引字段最大可能长度，并非实际使用长度，在不损失精确性的前提下， 长度越短越好 。

![](<.gitbook/assets/image (1).png>)

**Explain之 extra**

![](<.gitbook/assets/image (2).png>)

## show profile分析SQL

Mysql从5.0.37版本开始增加了对 show profiles 和 show profile 语句的支持。show profiles 能够在做SQL优化时帮助我们了解时间都耗费到哪里去了。

通过 have\_profiling 参数，能够看到当前MySQL是否支持profile：

```plsql
select @@have_profiling; 
set profiling=1; -- 开启profiling 开关； s
```

执行完一些sql命令之后，再执行show profiles 指令， 来查看SQL语句执行的耗时：

```plsql
show profiles;
```

通过show profile for query query\_id 语句可以查看到该SQL执行过程中每个线程的状态和消耗的时间：

```plsql
show profile for query 8;
```

在获取到最消耗时间的线程状态后，MySQL支持进一步选择all、cpu、block io 、context switch、page faults等明细类型类查看MySQL在使用什么资源上耗费了过高的时间。例如，选择查看CPU的耗费时间 ：

```plsql
show profile cpu for query 133; 
```

## trace分析优化器执行计划

MySQL5.6提供了对SQL的跟踪trace, 通过trace文件能够进一步了解为什么优化器选择A计划, 而不是选择B计划

![](<.gitbook/assets/image (3).png>)

打开trace ， 设置格式为 JSON，并设置trace最大能够使用的内存大小，避免解析过程中因为默认内存过小而不能够完整展示。

```plsql
SET optimizer_trace="enabled=on",end_markers_in_json=on; 
set optimizer_trace_max_mem_size=1000000;
```

执行自定义SQL语句，最后， 检查information\_schema.optimizer\_trace就可以知道MySQL是如何执行SQL的 ：

```plsql
select * from information_schema.optimizer_trace\G; -- 不能再navicat运行，需要在终端上运行
```

## 使用索引优化

索引是数据库优化最常用也是最重要的手段之一, 通过索引通常可以帮助用户解决大多数的MySQL的性能优化问题。

