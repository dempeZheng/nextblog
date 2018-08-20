---
layout: post
title:  "Mysq seconds_behind_master的含义"
date:  2018-08-13 22:39:04
type: mysql
categories: [mysql]
keywords: mysql,InnoDB,seconds_behind_master
---
## 背景
`seconds_behind_master`这个指标公司的db监控平台上面有，因为单词语义比较清晰，一致似懂非懂的认为是从库延时的时间。
>很多人说不上这个指标的含义，有的人认为是落后的事件数，有的人则认为是延迟的时间。

## case
1） 经常业务侧监控到从库延时，但是seconds_behind_master这个指标正常

官方指标解释：

>Seconds_Behind_Master: The number of seconds that the slave SQL thread is behind processing the master binary log


是SQL thread在执行IO thread dump下来的relay log的时间差。大家都知道relay log中event记录的时间戳是主库上的时间戳，而SQL thread的时间戳是从库上的，也就是说，如果主库和从库的时间是一致的，那么这个SBM代表的确实是从库延后主库的一个时间差。但是如果主库和从库的时间不是一致的，那么这个SBM的意义就基本不存在了

## 结论
`seconds_behind_master`，简单的来说，就是Slave SQL线程执行relay log(主库的binlog)的时候，通过relay log中event的时间戳（主库commit事务的时间）和服务器当前时间对比，得出的时间差。 
 
1）如果binlog本身传输延时，`seconds_behind_master`监控不到；   
2）`seconds_behind_master`得出的延时时间是很可能会滞后；例如：`insert into aaa select 1 from aaa where x = 1 or sleep(1000);`，从库在执行这条sql   `seconds_behind_master`指标不会有变化，但是执行完了`seconds_behind_master`就突然变成了1000；  
3）mysql 升级到5.7版本，支持并行复制，`seconds_behind_master`这个指标延时的概率非大大减小；  
4）从库重放relay log的sql线程或者接收binlog的IO线程假死造成的从库延时，`seconds_behind_master`这个指标也监控不到。  



## 参考文档

https://www.cnblogs.com/billyxp/p/3470376.html


