---
date: 2024-12-30
tags: 
summary: 详细介绍了MySQL死锁(DeadlockLoserDataAccessException)的排查和解决方案。包括获取死锁日志、分析事务信息、还原死锁场景等步骤，并提供了常见原因分析和解决方案，如减小事务范围、使用连续性主键ID等。
coverImage: https://mmbiz.qpic.cn/sz_mmbiz_png/aAia1M9Vb9bicypPFaFMqhicicUAtWvRw62nKcgRjgQGicibK3Cq7cnTsqNdFvWDFm0YrdAjbS4O0Gibr0ILPFtzf6ssQ/640?wx_fmt=png&amp;from=appmsg
author: 柳树下的程序员
---
死锁 DeadlockLoserDataAccessException 解决方案
`经验总结：`
1. 先获取死锁日志，这个bug，需要在第一时间得到死锁日志。
2. 分析事务信息
3.  还原死锁场景
4. 定位根本原因


```shell
org.springframework.dao.DeadlockLoserDataAccessException: 
Deadlock found when trying to get lock; try restarting transaction
### SQL: update t_order set status = ? where order_id = ?
```

### 第一步：执行死锁检查命令
```shell
SHOW ENGINE INNODB STATUS;
```

### 第二步：定位关键信息
在输出结果中，重点关注以下几个部分：
```shell
------------------------
LATEST DETECTED DEADLOCK
------------------------
*** (1) TRANSACTION:
TRANSACTION 8472, ACTIVE 2 sec
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 28, query id 131 localhost updating
update t_order set status = 2 where order_id = 10001

*** (2) TRANSACTION:
TRANSACTION 8471, ACTIVE 4 sec
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1136, 3 row lock(s)
MySQL thread id 27, query id 130 localhost updating
update t_order set status = 3 where order_id = 10002

*** WE ROLL BACK TRANSACTION (1)
```

### 第三步：定位项目代码位置，分析原因
一般情况有：
1. 大范围事务串行化，导致死锁，解决方案：尽量使用手动事务控制，减小事务范围
2. 如果主键id 生成方式是非连续性的id，这种死锁的情况极有可能发生，具体原因是mysql 间隙锁。解决方案：推荐使用雪花算法作为主键。当然自增id也行
3. 更新操作应该尽量避免连表查询更新，避免锁表

### 如果觉得对你有点帮助，请点点最下面的关注、点赞、转发、在看。谢谢支持！公众号：`ddtongxiang`在线等你！