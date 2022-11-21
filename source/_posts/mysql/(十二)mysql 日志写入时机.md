---
title: (十二)mysql 日志写入时机
date: 2022-11-06 16:41:39
index_img: /img/mysql.png
categories:
  - mysql
tags:
  - mysql
---

### binlog

mysql 操作日志的步骤当中,几乎都有为日志文件添加对应的缓存,由此可见 mysql 最大程度上在利用缓存来平衡内存和磁盘间写入速率的差距

同样的 `binlog` 也有自己的 `binlog buffer`, mysql 为每个事务都分配了对应的 `binlog buffer` 当事务执行时,都是先写入 `binlog buffer`, 当事务提交的时候才把 `binlog buffer` 写入 `binlog`

根据以前的知识,这里写入的 `binlog file` 其实也还没有真正写入磁盘

![img.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7qp5h2r8mj30ca0a4tau.jpg)

与这张图类似,都是将用户态的 `buffer` 写入 `os buffer` 然后调用 `fsync()` 刷入磁盘

同样的,有三种写入机制
1. 延时写,事务提交后,只写入 `os buffer`,不调用 `fsync()` 数据可靠性最低
2. 实时写,实时刷,事务提交后,写入 `os buffer`,立即调用 `fsync()` 数据可靠性最高
3. 实时写,延时刷,事务提交后,写入 `os buffer` ,积攒 `N` 个事务后一次性刷盘, 可能会丢失 `N` 个事务的记录

### redo log

同样的, `redo log` 也由两个部分组成 `redo log buffer` 和 `redo log file`,与 `binlog` 差不多,每次都是先写入 `buffer` 然后写入 `os buffer` 再调用 `fsync()` 刷盘

`redo log` 也有三种写入机制:
1. 延时写,只写入 `redo log buffer`,由后台进程每秒写入 `os buffer` 然后再刷盘,可能丢失 1s 以内的数据
2. 实时写,实时刷,每次写入 `redo log buffer` 后立即写入 `os buffer` 并且刷盘,数据可靠性最高
3. 实时写,延时刷,每次写入 `redo log buffer` 后立即写入 `os buffer` 但是并不立即刷盘,由后台进程每 1s 进行刷盘操作,可能丢失 1s 以内的数据

根据 **两阶段提交** 的原则,实际上在事务提交的过程中,需要先将 `redo log` 写为 `prepare` 状态,再写 `binlog` 最后将 `redo log` 写为 `commit` 状态

所以在实际的写盘时,在 `prepare` 阶段就要进行一次持久化操作

mysql 数据最可靠的配置,就是 `binlog` 每次提交事物前刷盘, `redo log` 的 `prepare` 也刷盘

### 组提交

由于 mysql 最可靠的刷盘配置会在一次事务过程当中进行两次刷盘,为了尽可能提高写磁盘的性能,会将多次写盘操作合并为一次进行组提交

`LSN` 日志逻辑序号,例如 3 个并发事务都提交了 `prepare` 的日志,需要完成刷盘操作,根据先后顺序分别是 `LSN=50` 的事务 1, `LSN=120` 的事务 2, `LSN=160` 的事务 3

当 InnoDB 刷新 `redo log` 的后台进程开始刷盘时,发现需要刷入的日志尾序号是 `LSN=160`, 头部是 `LSN=50` 那么这一次刷盘就会把三个事务的请求合并一起刷入磁盘,尽可能的减少了刷盘的次数

![img.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7vjetd1mdj30gq0bfdho.jpg)

mysql 为了尽可能的提高一次刷盘的组大小,采用最简单的办法 `拖时间`, 只要尽可能的晚调用 `fsync()` 那么一次刷盘的组成员就会越大,所以在事务的两阶段提交过程,也采取了对应的优化措施

两阶段提交: 存储引擎更新 `buffer pool` 里面的数据,写入 `redo log` 为 `prepare` 状态, 返回 `Server` 将记录写入 `binlog` 后提交事务, 存储引擎写入 `redo log` 为 `commit`

为了尽可能的利用组提交,mysql 实际上将 `binlog` 写入 `os buffer` 这一步提前,放到 `redo log` 刷入 `prepare` 之后

![img_1.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7vjf05oe6j30g50i9wiv.jpg)