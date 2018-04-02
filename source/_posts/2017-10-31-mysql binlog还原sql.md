---
layout: post
title:  "mysql binlog还原sql"
date:  2017-10-31 12:39:04
type: 灾备
categories: [灾备]
keywords: mysql,binlog,主备切换
---

## 场景

最近在做灾备演练，需要对mysql主备不同步的场景切换主备关系后修复数据。这个过程中就涉及到binlog的提取和恢复。
核心需求是将差异的binlog提取出来转换为可执行的sql语句。

这个需求涉及到两个点：
1）切换主从关系前先记录主从同步的位点。
2）根据位点提取binlog，并且将这部分binlog还原成sql语句。

解决这个问题需要对binlog有一定了解。当然前提是至少对mysql的主从同步有一个初步的认识。


## binlog概述

MySQL binlog以event的形式，记录了MySQL server从启用binlog以来所有的变更信息，能够帮助重现这之间的所有变化。MySQL引入binlog主要有两个目的：一是为了主从复制；二是某些备份还原操作后需要重新应用binlog。

有三种可选的binlog格式，各有优缺点：

- statement：基于SQL语句的模式，binlog数据量小，但是某些语句和函数在复制过程可能导致数据不一致甚至出错；
- row：基于行的模式，记录的是行的完整变化。很安全，但是binlog会比其他两种模式大很多；
- mixed：混合模式，根据语句来选用是statement还是row模式；
利用binlog闪回，需要将binlog格式设置为row。row模式下，一条使用innodb的insert会产生如下格式的binlog：

```
# at 1129
#161225 23:15:38 server id 3773306082  end_log_pos 1197         Query   thread_id=1903021       exec_time=0     error_code=0
SET TIMESTAMP=1482678938/*!*/;
BEGIN
/*!*/;
# at 1197
#161225 23:15:38 server id 3773306082  end_log_pos 1245         Table_map: `test`.`user` mapped to number 290
# at 1245
#161225 23:15:38 server id 3773306082  end_log_pos 1352         Write_rows: table id 290 flags: STMT_END_F

BINLOG '
muJfWBPiFOjgMAAAAN0EAAAAACIBAAAAAAEABHRlc3QABHVzZXIAAwMPEQMeAAAC
muJfWB7iFOjgawAAAEgFAAAAACIBAAAAAAEAAgAD//gBAAAABuWwj+i1tVhK1hH4AgAAAAblsI/p
krFYStYg+AMAAAAG5bCP5a2ZWE/onPgEAAAABuWwj+adjlhNeAD4BQAAAAJ0dFhRYJM=
'/*!*/;
# at 1352
#161225 23:15:38 server id 3773306082  end_log_pos 1379         Xid = 5327954
COMMIT/*!*/;
```

### binlog打开方式：
1)binlog的位置


```sql
show variables like 'log_%';
```

```
 +----------------------------------------+--------------------------------+
| Variable_name                          | Value                          |
+----------------------------------------+--------------------------------+
| log_bin                                | ON                             |
| log_bin_basename                       | /var/lib/mysql/mysql-bin       |
| log_bin_index                          | /var/lib/mysql/mysql-bin.index |
| log_bin_trust_function_creators        | OFF                            |
| log_bin_use_v1_row_events              | OFF                            |
| log_error                              | /var/log/mysql/error.log       |
| log_output                             | FILE                           |
| log_queries_not_using_indexes          | OFF                            |
| log_slave_updates                      | OFF                            |
| log_slow_admin_statements              | OFF                            |
| log_slow_slave_statements              | OFF                            |
| log_throttle_queries_not_using_indexes | 0                              |
| log_warnings                           | 1                              |
+----------------------------------------+--------------------------------+
```
binlog的位置在：/var/lib/mysql/mysql-bin
2)查看binlog的文件：

```sql
show binary logs;
```

3)查看binlog具体的事件：

```
show binlog events in 'mysql-bin.000006';
```

```
+------------------+------+-------------+-----------+-------------+------------------------------------------------------------------------------------------------------------------------------------+
| Log_name         | Pos  | Event_type  | Server_id | End_log_pos | Info                                                                                                                               |
+------------------+------+-------------+-----------+-------------+------------------------------------------------------------------------------------------------------------------------------------+
| mysql-bin.000006 |    4 | Format_desc |         1 |         120 | Server ver: 5.6.33-0ubuntu0.14.04.1-log, Binlog ver: 4                                                                             |
| mysql-bin.000006 |  120 | Query       |         1 |         237 | create database `demo`character set utf8mb4                                                                                        |
| mysql-bin.000006 |  237 | Query       |         1 |         429 | use `demo`; Create table `demo`.`test`(
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(50),
  primary key (`id`)
) |
| mysql-bin.000006 |  429 | Query       |         1 |         501 | BEGIN                                                                                                                              |

| mysql-bin.000006 | 2755 | Xid         |         1 |        2786 | COMMIT /* xid=1056 */                                                                                                              |
+------------------+------+-------------+-----------+-------------+------------------------------------------------------------------------------------------------------------------------------------+
```

4) binlog的另外另外一种查看方式

```
 /usr/bin/mysqlbinlog  --base64-output=decode-rows -v  mysql-bin.000006
```

5)查询主从同步的位点
```sql
show master status;
```

## 原理
>MySQL binlog以event的形式，记录了MySQL server从启用binlog以来所有的变更信息，能够帮助重现这之间的所有变化。

这个是binlog数据修复的理论依据。剩下的就是不同语言的实现。但是基本方式都是大同小异。比较通用的做法都是：
1）伪装成slave，拉取mysql binlog日志
2）解析mysql binlog日志，遍历binlog event
3）根据binlog event还原出sql

## 方案
知晓原理，花点时间实现一个初版的工具应该不太难。但是要想实现一个靠谱的工具还是需要有点经验的。这里有一个实现的比较靠谱易用的脚本，强烈安利：https://github.com/danfengcao/binlog2sql

>当然有兴趣的也可以自己试试，多动手对巩固学习还是非常有帮助的。
对于java系的同学直接依赖阿里的canal来实现也是比较容易的。
花了大概2个小时简单了实现了下想法（还有很多坑没填），代码地址：https://github.com/dempeZheng/binlog2sql
业余玩玩也是不错的，对巩固canal，了解binlog都是有一定帮助。
顺便也安利一下cannal，场景用的对，能解决复杂问题。是java解决数据库相关问题的一个比较好的case。https://github.com/alibaba/canal ，文档真心不错。



## 总结
binlog的常见运维还是比较值得去了解的，熟悉后对于误操作还可以人工恢复，不至于跪掉。

## 参考
https://github.com/danfengcao/binlog2sql/blob/master/example/mysql-flashback-priciple-and-practice.md


