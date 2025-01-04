---
title: Java项目卡顿问题排查与解决方案
keywords: [Java性能优化, 系统监控, CPU分析, 线程堆栈, 问题排查, top命令, jstack]
date: 2024-03-15
tags: [性能优化, 系统监控, CPU分析, 线程堆栈, 问题排查]
summary: 介绍了Java项目出现卡顿时的排查方法，主要从系统负载、数据库执行和代码执行三个维度进行分析，包括使用top命令监控系统资源和线程堆栈信息查看等实用技巧。
author: 柳树下的程序员
---

>背景：你是否曾遇到过系统突然变慢、卡顿的情况?这不仅影响用户体验,还可能导致严重的业务损失。别担心!今天我们为你准备了一份"项目应急箱",帮你快速定位并解决系统卡顿问题

排查思路：
1. 系统负载
2. 数据库执行情况
3. 代码执行情况

>一、系统负载大揭秘

1. 首先,让我们来看看系统的整体情况:
```shell
top
```

使用top命令,你可以一目了然地看到当前系统的负载状况。如果负载异常高,那么很可能是某个贪婪的进程在疯狂消耗资源

> 揪出CPU大户

根据top命令,我们可以看到哪个进程占用了大量的CPU资源。接下来,我们需要深入挖掘这些进程,看看它们在做什么。

**mysql进程的 cpu占用高**

1. 查看数据库执行情况，是否存在慢sql
```
-- 执行 SHOW PROCESSLIST 命令
SHOW PROCESSLIST;

| Id | User | Host      | db   | Command | Time | State | Info 

-- 或者查询 information_schema.PROCESSLIST 表
SELECT * FROM information_schema.PROCESSLIST WHERE Info IS NOT NULL ORDER BY Time DESC;

```
2. 杀掉查询时间较长的sql 连接ID,切记不要干掉非查询语句的进程
```
KILL Id
```

**Java进程的CPU占比高**

![img_5.png](img_5.png) 图一
![img_7.png](img_7.png) 图二
![img_6.png](img_6.png) 图三

1. 查看java进程中某个线程的堆栈信息
```shell
//使用top命令,如图一 top
top
//用命令进入进程查看占用CPU的线程资源，如图二  top -H -p PID
top -H -p 32961
//将占用最高的线程id复制出来，转为16进制，如图三  printf '%x\n' nid
printf '%x\n' 32961  
//查看线程内部的堆栈信息  jstack PID |grep 'nid' -C5
jstack 32961 |grep '0x80c1' -C5
```
或者
```shell
//可以将进程的线程堆栈信息输出到 thread_dump.txt 文件中 ;命令 jstack PID | grep 'nid' -C5
jstack 32961 > thread_dump.txt
//过滤包含 nid 的行及其上下文 cat thread_dump.txt | grep 'nid' -C5
```

2. 通过堆栈信息，定位到代码
   通过关键信息，比如项目名称、方法名、类名等，结合项目代码，定位到问题所在。

> 如果觉得对你有点帮助，请点点最下面的关注、点赞、转发、在看。谢谢支持！公众号：`ddtongxiang`在线等你！

