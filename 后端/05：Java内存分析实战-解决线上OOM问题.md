---
date: 2024-03-15
title: Java内存分析实战-解决线上OOM问题
keywords: [Java内存分析, OOM问题分析, MAT工具使用, 堆内存分析, 内存泄漏排查, JVM调优, 性能优化]
tags: [内存分析, OOM问题, MAT工具, 堆转储, JVM调优]
summary: 详细介绍了如何使用Eclipse Memory Analyzer(MAT)工具来分析和解决Java应用中的OutOfMemoryError问题，包括内存问题的症状识别、堆快照获取和分析方法。
author: 柳树下的程序员
---
>在企业级Java应用中,内存问题是我们经常遇到的挑战之一。本文将介绍如何使用MemoryAnalyzer工具来分析和解决线上出现的`OutOfMemoryError(OOM)`问题。

`推荐使用：`

1. 系统无响应,用户无法登录
2. CPU使用率异常高
3. 频繁的Full GC,且每次GC后内存占用依然居高不下

这些症状都指向可能存在的内存问题。接下来,我们将使用专业工具来进行深入分析。

> 分析工具准备

我们选择使用`Eclipse Memory Analyzer (MAT)`作为主要分析工具。它是一个强大的Java堆内存分析工具,可以帮助我们找出内存泄漏和内存占用过高的原因。

下载地址: `https://eclipse.dev/mat/previousReleases.php`

请根据您的操作系统和JDK版本选择合适的版本。
JDK1.8 可以用 `Memory Analyzer 1.8.1 Release`
![img_15.png](img_15.png)

> 获取堆快照文件

要分析Java应用的内存使用情况,我们首先需要获取一个堆快照(Heap Dump)文件。以下是获取步骤:

1. 使用jmap命令生成堆快照文件:
   ```shell
   jmap -dump:format=b,file=heap.hprof {pid}
   ```
2. 压缩文件以便传输:
   ```shell
   tar -zcvf heap.gz heap.hprof
   ```
3. 下载压缩文件到本地进行分析

> 使用MAT分析堆转储

1. 打开MAT,选择File -> Open Heap Dump
2. 加载heap.hprof文件(可能需要几分钟时间)
3. 如果提示内存不足,可以修改MAT的配置文件(MemoryAnalyzer.ini),增加可用内存

> 分析重点

在MAT中,我们主要关注两个方面:

1. Leak Suspects(疑似内存泄漏)
2. Top Components(大对象)
   ![img_9.png](img_9.png)
   通过分析这两部分,我们可以找出导致内存问题的根本原因。

**方案一：**
![img_10.png](img_10.png)
图一
![img_11.png](img_11.png)
图二
1. 进入Leak Suspects后 如图1。
2. 进入 `See stacktrace.` 可以看到图二所示，堆栈信息中可以找到导致内存泄露的代码位置。
3. 如果堆栈信息很多，可以搜项目关键字，避开项目引入三方大对象的杂乱信息。
4. 然后根据代码位置优化代码

**方案二：**
![img_12.png](img_12.png)
图一
![img_13.png](img_13.png)
图二
![img_14.png](img_14.png)
1. 点击 `Thread Overview`，查看占用内存最大的线程。
2. 点击 线程后，进入 `Thread Details` 查看堆栈信息。
3. 根据项目信息找到泄露内存的代码位置,优化代码

> 解决方案

1. 优化数据库查询,减少一次性加载的数据量
2. 实现分页加载机制,避免一次性加载大量数据到内存
3. 调整JVM参数,增加堆内存大小并优化GC策略


> 如果觉得对你有点帮助，请点点最下面的关注、点赞、转发、在看。谢谢支持！公众号：`ddtongxiang`在线等你！

