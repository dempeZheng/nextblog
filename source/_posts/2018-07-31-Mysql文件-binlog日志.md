---
layout: post
title:  "MySql 文件-binlog"
date:  2018-07-31 22:39:04
type: mysql
categories: [mysql]
keywords: mysql,InnoDB,binlog
---
## MySql 文件-binlog
二进制日志（binary log）记录了对MySQL数据库执行的更改的所有操作，但是不包括SELECT和SHOW这类操作.  
指定要记录binlog的数据库和binlog_ignore_db指定不记录binlog的数据库。对运行中的mysql要启用binlog可以通过命令SET `SQL_LOG_BIN`=1来设置

- 物理日志（physical loggin）：物理日志记录每一行改变的内容。
- 逻辑日志（logical loggin）：逻辑日志记录不是改变的行，而是引起行内容改变的SQL语句（insert ，update ，delete）

按照上面的定义binlog文件就应该属于逻辑日志，redo log属于物理日志。

## 作用

- 恢复（recovery）：某些数据的恢复需要二进制日志
- 复制（replication）：通过复制和执行二进制日志使一台远程的MySQL数据库（slave或standby）与一台MysQL数据库（Master）实时同步
- 审计（audit）：用户可以通过二进制日志中的信息来进行审计，判断是否有对数据库进行注入攻击

## Binlog的日志格式
binlog有三种格式：Statement, Row和Mixed.

- 基于SQL语句的复制（statement-based replication, SBR）
- 基于行的复制（row-based replication, RBR）
- 混合模式复制（mixed-based replication, MBR）
### Statement

每一条会修改数据的sql都会记录在binlog中。

**优点**：不需要记录每一行的变化，减少了binlog日志量，节约了IO, 提高了性能。

**缺点**：由于记录的只是执行语句，为了这些语句能在slave上正确运行，因此还必须记录每条语句在执行的时候的一些相关信息，以保证所有语句能在slave得到和在master端执行的时候相同的结果。另外mysql的复制，像一些特定函数的功能，slave可与master上要保持一致会有很多相关问题。

相比row能节约多少性能与日志量，这个取决于应用的SQL情况，正常同一条记录修改或者插入row格式所产生的日志量还小鱼statement产生的日志量，但是考虑到如果带条件的update操作，以及整表删除，alter表等操作，row格式会产生大量日志，因此在考虑是否使用row格式日志时应该根据应用的实际情况，其所产生的日志量会增加多少，以及带来的IO性能问题。

### Row

5.1.5版本的MySQL才开始支持row level的复制,它不记录sql语句上下文相关信息，仅保存哪条记录被修改。

**优点**： binlog中可以不记录执行的sql语句的上下文相关的信息，仅需要记录那一条记录被修改成什么了。所以row的日志内容会非常清楚的记录下每一行数据修改的细节。而且不会出现某些特定情况下的存储过程，或function，以及trigger的调用和触发无法被正确复制的问题.

**缺点**:所有的执行的语句当记录到日志中的时候，都将以每行记录的修改来记录，这样可能会产生大量的日志内容。

>新版本的MySQL中对row level模式也被做了优化，并不是所有的修改都会以row level来记录，像遇到表结构变更的时候就会以statement模式来记录，如果sql语句确实就是update或者delete等修改数据的语句，那么还是会记录所有行的变更。

### Mixed

从5.1.8版本开始，MySQL提供了Mixed格式，实际上就是Statement与Row的结合。 
在Mixed模式下，一般的语句修改使用statment格式保存binlog，如一些函数，statement无法完成主从复制的操作，则采用row格式保存binlog，MySQL会根据执行的每一条具体的sql语句来区分对待记录的日志形式，也就是在Statement和Row之间选择一种。

## 相关参数

## sync_binlog

`sync_binlog=1 or N`

>This makes MySQL synchronize the binary log’s contents to disk each time it commits a transaction 

- `sync_binlog=0`:基于操作系统的fysnc()方法同步，性能好，但是容易丢数据；
- `sync_binlog=1`:每一个transcation commit都会调用一次fysnc()，能保证数据最安全，但是不能保证完全不丢数据。另外对性能影响较大。
- `sync_binlog=N`:保证最多丢N条数据     

 >默认情况下，并不是每次写入时都将binlog与硬盘同步。因此如果操作系统或机器(不仅仅是MySQL服务器)崩溃，有可能binlog中最后的语句丢 失了。要想防止这种情况，你可以使用sync_binlog全局变量(1是最安全的值，但也是最慢的)，使binlog在每N次binlog写入后与硬盘 同步。**即使sync_binlog设置为1,出现崩溃时，也有可能表内容和binlog内容之间存在不一致性。如果使用InnoDB表，MySQL服务器 处理COMMIT语句，它将整个事务写入binlog并将事务提交到InnoDB中。如果在两次操作之间出现崩溃，重启时，事务被InnoDB回滚，但仍 然存在binlog中**。可以用`--innodb-safe-binlog`选项来增加InnoDB表内容和binlog之间的一致性。(注释：在MySQL 5.1中不需要--innodb-safe-binlog；由于引入了XA事务支持，该选项作废了），该选项可以提供更大程度的安全，使每个事务的 binlog(sync_binlog =1)和(默认情况为真)InnoDB日志与硬盘同步，该选项的效果是崩溃后重启时，在滚回事务后，MySQL服务器从binlog剪切回滚的 InnoDB事务。这样可以确保binlog反馈InnoDB表的确切数据等，并使从服务器保持与主服务器保持同步(不接收 回滚的语句)。
 
 ### 示例：
1)正常场景  
```sql
   fsync写binlog  
  commit  
```

 
 2)`sync_binlog=1`数据丢失的场景

```sql
 fsync写binlog  
  宕机  
  commit  失败  
  重启  
  回滚  
```

 
 `这个时候事务回滚，但是binlog写入成功`
 
 
 >
 
 ## binlog的解析
 binlog事件类型一共有三个版本：
- v1: Used in MySQL 3.23
- v3: Used in MySQL 4.0.2 though 4.1
- v4: Used in MySQL 5.0 and up
>v2出现了很短的时间，并且已经不被支持

现在所使用的MySQL一般都是5.5起了，所以下面陈述的都是v4版的binlog事件类型:

### event的类型
binlog的事件类型一共有以下几种：

```cpp
enum Log_event_type { 
  UNKNOWN_EVENT= 0, 
  START_EVENT_V3= 1, 
  QUERY_EVENT= 2, 
  STOP_EVENT= 3, 
  ROTATE_EVENT= 4, 
  INTVAR_EVENT= 5, 
  LOAD_EVENT= 6, 
  SLAVE_EVENT= 7, 
  CREATE_FILE_EVENT= 8, 
  APPEND_BLOCK_EVENT= 9, 
  EXEC_LOAD_EVENT= 10, 
  DELETE_FILE_EVENT= 11, 
  NEW_LOAD_EVENT= 12, 
  RAND_EVENT= 13, 
  USER_VAR_EVENT= 14, 
  FORMAT_DESCRIPTION_EVENT= 15, 
  XID_EVENT= 16, 
  BEGIN_LOAD_QUERY_EVENT= 17, 
  EXECUTE_LOAD_QUERY_EVENT= 18, 
  TABLE_MAP_EVENT = 19, 
  PRE_GA_WRITE_ROWS_EVENT = 20, 
  PRE_GA_UPDATE_ROWS_EVENT = 21, 
  PRE_GA_DELETE_ROWS_EVENT = 22, 
  WRITE_ROWS_EVENT = 23, 
  UPDATE_ROWS_EVENT = 24, 
  DELETE_ROWS_EVENT = 25, 
  INCIDENT_EVENT= 26, 
  HEARTBEAT_LOG_EVENT= 27, 
  IGNORABLE_LOG_EVENT= 28,
  ROWS_QUERY_LOG_EVENT= 29,
  WRITE_ROWS_EVENT_V2 = 30,
  UPDATE_ROWS_EVENT_V2 = 31,
  DELETE_ROWS_EVENT_V2 = 32,
  GTID_LOG_EVENT= 33,
  ANONYMOUS_GTID_LOG_EVENT= 34,
  PREVIOUS_GTIDS_LOG_EVENT= 35, 
  ENUM_END_EVENT 
  /* end marker */ 
};
```
### 事件类型分析

```
+=====================================+
| event     | timestamp         0 : 4    |
| header  +----------------------------+
|              | type_code         4 : 1    |
|             +----------------------------+
|              | server_id         5 : 4    |
|             +----------------------------+
|              | event_length      9 : 4    |
|             +----------------------------+
|              | next_position    13 : 4    |
|             +----------------------------+
|              | flags            17 : 2    |
|             +----------------------------+
|              | extra_headers    19 : x-19 |
+=====================================+
| event     | fixed part        x : y    |
| data      +----------------------------+
|              | variable part              |
+=====================================+
```

## binlog的应用

### canal
cannl的本质也是通过协议伪装成slave，然后解析binlog来工作的。

### 数据恢复脚本

## 补充
relay log
