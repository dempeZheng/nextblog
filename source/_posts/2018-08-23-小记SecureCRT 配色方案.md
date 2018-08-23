---
layout: post
title:  "小记SecureCRT 配色方案"
date:  2018-08-23 22:39:04
type: 札记
categories: [札记]
keywords: secureCRT,配色
---


## 最终效果
SecureCRT 的配色真的是很难看，一直忍受了几年，最近突然被说难看，索性花点时间，搜了写文章，对照修改一下，最终效果如下：
![Alt text](./images/1534991196253.png)
整体感觉还行，至少比之前的要好看很多。
顺便记下来，方便以后调整。

## 步骤
### 1. 设置背景颜色和字体颜色： 
选项（`Options`）==》会话选项（`Sessions options`）==》终端（`Terminal`）==》仿真（`Emulation`） 

![Alt text](./images/1534991693552.png)

### 2.选项（Options）==》全局选项（Global options）==》一般（General）==》默认会话（defualt session）==》点击 Edit Defualt Setting 

![Alt text](./images/1534991765619.png)

### 3.进去第一步先将use global ANSI color settings的勾去掉，否则无法编辑这些颜色，第二遍点击第一个颜色块配置背景颜色，我用的颜色数据如第三步128、240、25，也可以配置自己喜欢的颜色 
![Alt text](./images/1534991790841.png)

接下来配置字体颜色，点击最后一块颜色块步骤同样，如图： 
![Alt text](./images/1534991808218.png)
![Alt text](./images/1534991825924.png)
![Alt text](./images/1534991834618.png)
![Alt text](./images/1534991842702.png)
![Alt text](./images/1534991924431.png)


## 参考文章
https://blog.csdn.net/zq710727244/article/details/53909801