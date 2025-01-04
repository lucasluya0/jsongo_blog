---
date: 2024-03-15
title: Windows和Linux系统常用命令指南
keywords: [命令行工具, Linux命令, Windows命令, 端口管理, 文件处理, 日志分析, less命令, 系统管理]
tags: [命令行工具, Linux命令, Windows命令, 系统管理, 日志分析]
summary: 整理了Windows和Linux系统下的常用命令，包括Windows下的端口占用排查、大文件分割处理，以及Linux下的文件压缩、日志查找分析等实用操作。特别介绍了less命令的常用操作技巧。
author: 柳树下的程序员
---

>Windows开发环境常用命令

1. 端口占用问题排查

   当遇到端口被占用的情况，可以使用以下命令进行排查：

```shell
# 查找指定端口的进程
netstat -aon|findstr "8080"

# 终止进程
taskkill /PID 进程号 /F

# 查看所有端口占用情况
netstat -ano

```

2. 大文件处理

   安装git，使用Git Bash提供的split命令进行文件分割：

```shell
# 按大小分割（推荐）
split 源文件 -b 500m    # 将文件分割成500MB大小的多个文件

# 按行数分割
split 源文件 -l 10000   # 将文件分割成每个10000行的多个文件

# 分割时指定输出文件前缀
split 源文件 -b 500m 前缀名_    # 生成的文件将以"前缀名_"开头
```

>Linux常用命令

1. 文件压缩

```shell
# 压缩文件
tar -zcvf 目标文件.tar.gz 源文件/目录

# 常用参数说明：
# -z：使用gzip压缩
# -c：创建新归档
# -v：显示详细信息
# -f：指定归档文件名

# 解压文件
tar -zxvf 文件名.tar.gz
```

2. 日志查找和分析

```shell
# 查找并显示上下文
cat web_start.log | grep -50 'getItemListMoneyByOrderIds' | less

# 其他常用日志分析命令
grep -n "关键字" 文件名     # 显示行号
tail -f 文件名             # 实时查看日志
```

**`less`命令常用操作技巧：**

1. 搜索：
    - 按/进行向下搜索
    - 按?进行向上搜索
    - 输入要搜索的文本后按Enter
2. 导航：
    - n：跳转到下一个匹配项
    - N：跳转到上一个匹配项
    - 空格：向下翻页
    - b：向上翻页
    - g：跳转到文件开头
    - G：跳转到文件末尾
3. 退出：
    - q：退出less查看器

#### PS:针对 `redis.conf` 搜索 ``/var/run`` 的内容，并显示上下文50行。

第一步：
![img_16.png](img_16.png)
第二步：
![img_18.png](img_18.png)
第三步：再搜索的基础上直接输入导航的命令


> 如果觉得对你有点帮助，请点点最下面的关注、点赞、转发、在看。谢谢支持！公众号：`ddtongxiang`在线等你！