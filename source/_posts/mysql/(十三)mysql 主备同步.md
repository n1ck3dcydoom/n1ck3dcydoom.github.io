---
title: (十三)mysql 主备同步
date: 2022-11-06 17:34:17
index_img: /img/mysql.png
categories:
  - mysql
tags:
  - mysql
---

### mysql 主备同步的过程

![img.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7vlun7uysj30rm0h5dlk.jpg)

对于主节点 A 来说,记录一个事务的过程依然符合两阶段提交,只不过多了一个 `dump_thread` 线程将 `binlog` 定时同步给从库 B

从库 B 通过 `io_thread` 与主库维持一个长连接接收从主库发过来的 `binlog`, 然后有多个 `sql_thread` 线程同步 `binlog` 里面的内容到数据库当中 

### binlog 的存储格式

1. statement
2. row
3. mixed

#### statement

`statement` 模式会如实的记录每一条执行的 sql,例如

![img_1.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7vlutt4h1j314q05vjxi.jpg)

注意一个问题,使用 `statement` 格式的时候,如果 `delte` 语句带有 `limit` 的话,很可能会导致主备不一致

因为 `delete` 后面如果有多个 `where` 条件,在主库和从库上分别执行这条 sql 语句是,很有可能两个库选择的索引不一致

这是因为 `statement` 记录的是原始 sql,当主库执行时,走索引 `a`,当从库执行时走索引 `t_modified`, 这是完全可能发生的情况

#### row

`row` 模式不会记录原始 sql 语句,而是记录每条 sql 语句实际操作的行

**这里因为我的 Mac 是 M1 arm64 架构的,而 docker 支持 arm64 架构的 image 都不带 mysqlbinlog 工具,唯一支持 mysqlbinlog 工具的镜像是 mysql-debian,然而 mysql-debian 仅支持 amd64 架构**

![img_2.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7vluz1purj30sg09sn5a.jpg)

实际上可以看到最终执行的语句里面其实是删除 `id=4` 的记录

### 循环复制

如果说主从之间互为主备关系,即 `binlog` 文件会互相发给对方用于同步

对于同一行更新语句生成的 `binlog` 记录,如何解决循环复制的问题

1. 规定两个库的 `server id` 必须不同
2. 从库收到日志后,重放过程记录 `binlog` 时生成与原 `server id` 相同的日志
3. 收到从库发过来的日志后,判断日志里面记录的 `server id` 是否是自己的; 如果是自己生成的则丢弃日志;否则进行同步