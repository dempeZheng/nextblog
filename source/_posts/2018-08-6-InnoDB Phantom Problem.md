---
layout: post
title:  "InnoDB Phantom Problem(幻读)"
date:  2018-08-06 22:39:04
type: mysql
categories: [mysql]
keywords: mysql,InnoDB,mvcc,phantom problem,幻读
---


## 关于幻读的困惑
最近在看《Mysql 技术内幕 InnoDB存储引擎》，看到幻读的部分很困惑。

>在默认的事务隔离级别下，即REPEATABLE READ下，InnoDB存储引擎采用Next-Key Locking机制来避免Phantom Problem；--《Mysql 技术内幕 InnoDB存储引擎》

InnoDB通过MVCC机制来实现一致性非锁定读，在REPREATABLE READ事务隔离级别下，只能看到比当前事务号小的版本，什么场景下会出现幻读呢？
关于MVCC详见[InnoDB MVCC多版本并发读](http://zhizus.com/2018-08-1-MVCC%E5%A4%9A%E7%89%88%E6%9C%AC%E5%B9%B6%E5%8F%91%E8%AF%BB.html)

百思不得其解。

网上查了很多资料，参差不齐。也不知道应该信任哪一个版本。

于是，顺手翻了《高性能的Mysql》，读到下面一段关于幻读的解释，

>REPEATABLE READ解决了脏读的问题。该级别保证了在同一事务中多次读取的同样记录是一致的。但是理论上，可重复读隔离级别还是无法解决幻读（`Phantom Problem`）的问题。
所谓的幻读，指的是当某个事务再读取某个范围内的记录是，回产生换行（Phantom Row）。**InnoDB存储引擎通过多版本并发控制（`MVCC，Multiversion Concurrency Control`）解决了幻读问题。**--《高性能的Mysql]》

比较有意思了，两本书上面说法不太一致。
>To prevent phantoms, InnoDB uses an algorithm called next-key locking that combines index-row locking with gap locking. --Mysql官网

官网上面找到这句，这个至少说明《Mysql 技术内幕 InnoDB存储引擎》这本书是上面的没错。

>The so-called phantom problem occurs within a transaction when the same query produces different sets of rows at different times. For example, if a SELECT is executed twice, but returns a row the second time that was not returned the first time, the row is a “phantom” row.

官网给出关于幻读的定义：在同一事务下，连续两次执行相同的查询sql，可能返回不同的结果。第二次可能返回不存在的行。

在这个语意的描述下，`REPEATABLE READ`正常场景下就不会出现幻读了。
MVCC机制能提供一致性的非锁定读，在`REPEATABLE READ`隔离级别下保证读取的版本在事务开始之前。
那么在`REPEATABLE READ`隔离级下，什么时候会出现两次读的结果不一样呢？
我们先来看一个InnoDB在`REPEATABLE READ`下的"诡异现象"!

##  "诡异现象"
1）建一个简单的表

```
show create table t5;
+-------+---------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                            |
+-------+---------------------------------------------------------------------------------------------------------+
| t5    | CREATE TABLE `t5` (
  `id` int(11) DEFAULT NULL,
  KEY `id` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+-------+---------------------------------------------------------------------------------------------------------+
```
2）初始化数据如下：
```
mysql> select * from t5;         
+------+
| id   |
+------+
|    1 |
|    4 |
|    7 |
+------+
```

3）诡异现象
<table class="wrapped fixed-table confluenceTable tablesorter tablesorter-default stickyTableHeaders" role="grid" style="padding: 0px;"><colgroup><col style="width: 49.0px;"><col style="width: 274.0px;"><col style="width: 252.0px;"><col style="width: 355.0px;"></colgroup><thead class="tableFloatingHeaderOriginal"><tr role="row" class="tablesorter-headerRow"><th class="confluenceTh tablesorter-header sortableHeader tablesorter-headerUnSorted" data-column="0" tabindex="0" scope="col" role="columnheader" aria-disabled="false" unselectable="on" aria-sort="none" aria-label="时间: No sort applied, activate to apply an ascending sort" style="user-select: none;"><div class="tablesorter-header-inner">时间</div></th><th class="confluenceTh tablesorter-header sortableHeader tablesorter-headerUnSorted" data-column="1" tabindex="0" scope="col" role="columnheader" aria-disabled="false" unselectable="on" aria-sort="none" aria-label="Session A: No sort applied, activate to apply an ascending sort" style="user-select: none;"><div class="tablesorter-header-inner">Session A</div></th><th class="confluenceTh tablesorter-header sortableHeader tablesorter-headerUnSorted" data-column="2" tabindex="0" scope="col" role="columnheader" aria-disabled="false" unselectable="on" aria-sort="none" aria-label="Session B: No sort applied, activate to apply an ascending sort" style="user-select: none;"><div class="tablesorter-header-inner">Session B</div></th><th colspan="1" class="confluenceTh tablesorter-header sortableHeader tablesorter-headerUnSorted" data-column="3" tabindex="0" scope="col" role="columnheader" aria-disabled="false" unselectable="on" aria-sort="none" aria-label="备注: No sort applied, activate to apply an ascending sort" style="user-select: none;"><div class="tablesorter-header-inner">备注</div></th></tr></thead><thead class="tableFloatingHeader" style="display: none;"><tr role="row" class="tablesorter-headerRow"><th class="confluenceTh tablesorter-header sortableHeader tablesorter-headerUnSorted" data-column="0" tabindex="0" scope="col" role="columnheader" aria-disabled="false" unselectable="on" aria-sort="none" aria-label="时间: No sort applied, activate to apply an ascending sort" style="user-select: none;"><div class="tablesorter-header-inner">时间</div></th><th class="confluenceTh tablesorter-header sortableHeader tablesorter-headerUnSorted" data-column="1" tabindex="0" scope="col" role="columnheader" aria-disabled="false" unselectable="on" aria-sort="none" aria-label="Session A: No sort applied, activate to apply an ascending sort" style="user-select: none;"><div class="tablesorter-header-inner">Session A</div></th><th class="confluenceTh tablesorter-header sortableHeader tablesorter-headerUnSorted" data-column="2" tabindex="0" scope="col" role="columnheader" aria-disabled="false" unselectable="on" aria-sort="none" aria-label="Session B: No sort applied, activate to apply an ascending sort" style="user-select: none;"><div class="tablesorter-header-inner">Session B</div></th><th colspan="1" class="confluenceTh tablesorter-header sortableHeader tablesorter-headerUnSorted" data-column="3" tabindex="0" scope="col" role="columnheader" aria-disabled="false" unselectable="on" aria-sort="none" aria-label="备注: No sort applied, activate to apply an ascending sort" style="user-select: none;"><div class="tablesorter-header-inner">备注</div></th></tr></thead><tbody aria-live="polite" aria-relevant="all"><tr role="row"><td class="confluenceTd">1</td><td class="confluenceTd"><p>mysql&gt; begin;<br>Query OK, 0 rows affected (0.04 sec)</p><p>mysql&gt; select * from t5 where id&gt;4;<br>+------+<br>| id |<br>+------+<br>| 7 |<br>+------+<br>1 row in set (0.04 sec)</p><p><br></p><p><br></p></td><td class="confluenceTd"><br></td><td colspan="1" class="confluenceTd"><br></td></tr><tr role="row"><td class="confluenceTd">2</td><td class="confluenceTd"><br></td><td class="confluenceTd"><p>begin;<br>Query OK, 0 rows affected (0.04 sec)</p><p>mysql&gt; insert into t5 value(8);<br>Query OK, 1 row affected (0.03 sec)</p></td><td colspan="1" class="confluenceTd"><br></td></tr><tr role="row"><td class="confluenceTd">3</td><td class="confluenceTd"><p>mysql&gt; select * from t5 where id&gt;4;<br>+------+<br>| id |<br>+------+<br>| 7 |<br>+------+<br>1 row in set (0.04 sec)</p></td><td class="confluenceTd"><br></td><td colspan="1" class="confluenceTd">这里 Session B事务没有提交，不管是READ COMMITED还是REPEATABLE READ都是不可见的</td></tr><tr role="row"><td class="confluenceTd">4</td><td class="confluenceTd"><br></td><td class="confluenceTd">commit；</td><td colspan="1" class="confluenceTd"><br></td></tr><tr role="row"><td class="confluenceTd">5</td><td class="confluenceTd"><p>mysql&gt; select * from t5 where id&gt;4;<br>+------+<br>| id |<br>+------+<br>| 7 |<br>+------+<br>1 row in set (0.04 sec)</p></td><td class="confluenceTd"><br></td><td colspan="1" class="confluenceTd">由于当前的事务隔离级别是<span>REPEATABLE READ，InnoDB通过MVCC机制提供一致性非锁定读，故目前仍然不可见Session B事务</span></td></tr><tr role="row"><td class="highlight-red confluenceTd" colspan="1" data-highlight-colour="red">6</td><td class="highlight-red confluenceTd" colspan="1" data-highlight-colour="red"><p>mysql&gt; update t5 set id=10 where id=8;<br>Query OK, 1 row affected (0.04 sec)<br>Rows matched: 1 Changed: 1 Warnings: 0</p></td><td class="highlight-red confluenceTd" colspan="1" data-highlight-colour="red"><br></td><td class="highlight-red confluenceTd" colspan="1" data-highlight-colour="red">诡异现象出现了，这里update的语句竟然可以看到Session B的提交</td></tr><tr role="row"><td colspan="1" class="confluenceTd">7</td><td colspan="1" class="confluenceTd"><p>mysql&gt; select * from t5 where id&gt;4; <br>+------+<br>| id |<br>+------+<br>| 7 |<br>| 10 |<br>+------+<br>2 rows in set (0.03 sec)</p></td><td colspan="1" class="confluenceTd"><br></td><td colspan="1" class="confluenceTd">Session A再次查询的时候就可以查看新的update的值，当然这里是合理的，因为在同一个事务中，应该能看到本事务的修改。</td></tr><tr role="row"><td colspan="1" class="confluenceTd">8</td><td colspan="1" class="confluenceTd">commit</td><td colspan="1" class="confluenceTd"><br></td><td colspan="1" class="confluenceTd"><br></td></tr></tbody></table>

## 『诡异现象』背后的原因

[关于InnoDB事务的一个“诡异”现象](https://yq.aliyun.com/articles/8868) 这篇文章从源代码的角度解释了这个诡异现象背后的原因。
简单的说可以是：update语句少了一个版本比较的判断。查询语句会判断查询语句的事务号是否大于当前的事务号。但是update语句没走这个判断分支。

对于这个『诡异现象』，InnoDB怎么解决的呢？

## 如何避免『诡异现象』
>To prevent phantoms, InnoDB uses an algorithm called next-key locking that combines index-row locking with gap locking. --Mysql官网

InnoDB存储引擎采用Next-Key Locking机制来避免`Phantom Problem`（幻读问题）

我们对查询sql加上一个X锁，那么其他事务在插入或者修改该范围的时候就会阻塞。这样就避免了幻读。
```sql
select * from t5 where id>4 for update;
```

## 总结
>（幻读（Phantom））：事务T1读取满足一些<搜索条件>的一组数据项。事务T2然后创建满足T1的<搜索条件>并提交的数据项。如果T1然后使用相同的<搜索条件>重复读取，它将获得与第一次读取不同的组数据项  --《ANSI SQL隔离级别》

如果按照这个幻读的定义，Mysql InnoDB存储引擎通过MVCC机制确实解决了幻读的问题。至少目前在`REPEATABLE READ`的隔离级别下，范围查询语句，没有发现存在幻读的案例，但是在更新操作存在幻读的场景。（见上文中的『诡异现象』）
那InnoDB 存储引擎在`REPEATABLE READ`事务隔离级别下的『写』幻读到底算不算幻读，这个要看幻读的定义标准了。

## 参考资料
[InnoDB官方文档--Phantom Row](https://dev.mysql.com/doc/refman/8.0/en/innodb-next-key-locking.html)

[关于InnoDB事务的一个“诡异”现象](https://yq.aliyun.com/articles/8868)

[InnoDB MVCC多版本并发读](http://zhizus.com/2018-08-1-MVCC%E5%A4%9A%E7%89%88%E6%9C%AC%E5%B9%B6%E5%8F%91%E8%AF%BB.html)