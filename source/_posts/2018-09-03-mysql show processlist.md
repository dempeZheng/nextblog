---
layout: post
title:  "Mysql show processlist"
date:  2018-09-03 22:39:04
type: java
categories: [mysql]
keywords: mysql,processlist
---


## show processlist

![Alt text](./images/1535941357441.png)

show processlist显示的sql会被截取，如果需要显示完成的sql可以用`show full processlist`；
如果需要根据processlist过滤，可以通过导出文本利用shell的`grep`过滤。
另外：
>Newer versions of SQL support the process list in information_schema:\

```sql
SELECT * FROM INFORMATION_SCHEMA.PROCESSLIST
```

>You can ORDER BY in any way you like.
The INFORMATION_SCHEMA.PROCESSLIST table was added in MySQL 5.1.7. You can find out which version you're using with:

```sql
SELECT VERSION()
```

 
## 示例

### 1.找出慢查，并杀死慢查进程

```sql
SELECT Id FROM information_schema.processlist where time>180;
```
杀死慢查sql 进程


### 2.找出死锁并杀死

```sql
SELECT Id,State FROM information_schema.processlist where state like '%lock%';
```


### 3.批量kill
查询超过5min钟的慢sql，并组装kil sql语句

```sql
mysql> select concat('kill ', id, ';') from information_schema.processlist where Command != 'Sleep' and Time > 300 order by Time desc;
+--------------------------+
| concat('kill ', id, ';') |
+--------------------------+
| kill 89969399;           |
+--------------------------+
1 row in set (0.02 sec)
```

## 参考文档
http://www.ywnds.com/?p=9337
