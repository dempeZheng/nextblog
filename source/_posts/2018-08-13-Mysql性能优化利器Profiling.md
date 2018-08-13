---
layout: post
title:  "Mysql语句性能分析利器Profiling"
date:  2018-08-13 22:39:04
type: mysql
categories: [mysql]
keywords: mysql,InnoDB,profiling
---
## profiling

>The SHOW PROFILE and SHOW PROFILES statements display profiling information that indicates resource usage for statements executed during the course of the current session.

`mysql profiling 对定位一条语句的I/O消耗和CPU消耗非常重要`

## 语法

```sql
SHOW PROFILE [type [, type] ... ]
    [FOR QUERY n]
    [LIMIT row_count [OFFSET offset]]

type:
    ALL
  | BLOCK IO
  | CONTEXT SWITCHES
  | CPU
  | IPC
  | MEMORY
  | PAGE FAULTS
  | SOURCE
  | SWAPS
```

SHOW PROFILES列出了最近发送到服务端的sql语句。列表长度由变量profiling_history_size控制，默认值为15，最大值为100。如果设为0相当于关闭profiling功能。

除SHOW PROFILE和SHOW PROFILES之外，所有sql语句的性能信息都会被记录，甚至包括有错误的语句。

SHOW PROFILE可以列出单条语句的详细信息。当不指定FOR QUERY n子句时，将输出最近执行的sql语句性能信息 。如果使用了FOR QUERY n，SHOW PROFILE会列出第n条sql的性能信息。n指的是SHOW PROFILES中列出的Query_ID值。

LIMIT row_count子句用于限制输出行数。

默认情况下，SHOW PROFILE显示Status和Duration两列信息。Status值的含义可以参见这里

使用type选项可以显示额外信息，可选的值如下： 
- ALL：展现所有信息 
- BLOCK IO：阻塞的输入输出次数 
- CONTEXT SWITCHES：上下文切换次数 
- CPU：用户及系统CPU耗时 
- IPC：收发消息次数（个人理解是特指进程间通信） 
- MEMORY：目前无用 
- SOURCE：列出相应操作对应的函数名及其在源码中的调用位置(行数) 
- SWAPS:swap次数

## 示例
profiling是会话级的，当会话结束，与之相关的profiling信息也会随之消失。

1）查询当前的profiling状态`SELECT @@profiling`

```sql
mysql> SELECT @@profiling;
+-------------+
| @@profiling |
+-------------+
|           0 |
+-------------+
```
2）设置profiling状态

```sql
mysql> SET profiling = 1;
```


```
mysql> SELECT @@profiling;
+-------------+
| @@profiling |
+-------------+
|           0 |
+-------------+
1 row in set (0.00 sec)

mysql> SET profiling = 1;
Query OK, 0 rows affected (0.00 sec)

mysql> DROP TABLE IF EXISTS t1;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> CREATE TABLE T1 (id INT);
Query OK, 0 rows affected (0.01 sec)

mysql> SHOW PROFILES;
+----------+----------+--------------------------+
| Query_ID | Duration | Query                    |
+----------+----------+--------------------------+
|        0 | 0.000088 | SET PROFILING = 1        |
|        1 | 0.000136 | DROP TABLE IF EXISTS t1  |
|        2 | 0.011947 | CREATE TABLE t1 (id INT) |
+----------+----------+--------------------------+
3 rows in set (0.00 sec)

mysql> SHOW PROFILE;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| checking permissions | 0.000040 |
| creating table       | 0.000056 |
| After create         | 0.011363 |
| query end            | 0.000375 |
| freeing items        | 0.000089 |
| logging slow query   | 0.000019 |
| cleaning up          | 0.000005 |
+----------------------+----------+
7 rows in set (0.00 sec)

mysql> SHOW PROFILE FOR QUERY 1;
+--------------------+----------+
| Status             | Duration |
+--------------------+----------+
| query end          | 0.000107 |
| freeing items      | 0.000008 |
| logging slow query | 0.000015 |
| cleaning up        | 0.000006 |
+--------------------+----------+
4 rows in set (0.00 sec)

mysql> SHOW PROFILE CPU FOR QUERY 2;
+----------------------+----------+----------+------------+
| Status               | Duration | CPU_user | CPU_system |
+----------------------+----------+----------+------------+
| checking permissions | 0.000040 | 0.000038 |   0.000002 |
| creating table       | 0.000056 | 0.000028 |   0.000028 |
| After create         | 0.011363 | 0.000217 |   0.001571 |
| query end            | 0.000375 | 0.000013 |   0.000028 |
| freeing items        | 0.000089 | 0.000010 |   0.000014 |
| logging slow query   | 0.000019 | 0.000009 |   0.000010 |
| cleaning up          | 0.000005 | 0.000003 |   0.000002 |
+----------------------+----------+----------+------------+
7 rows in set (0.00 sec)
```

>注意
在某些系统上，profiling只有部分功能生效，比如windows系统。此外，profiling是进程级的而非线和级别。这意味着其它线>>程的一些动作影响你在profiling中看到的执行时间信息。