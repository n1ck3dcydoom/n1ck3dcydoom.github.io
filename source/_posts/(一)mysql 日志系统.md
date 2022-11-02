---
title: (一)mysql 日志系统
date: 2022-11-02 22:25:39
index_img: /img/mysql.png
categories:
- mysql
tags:
- 数据库
---

## 日志模块 redo log 和 binlog

### redo log

首先明确一点: mysql 是 **先写日志,再写磁盘**  这个和 redis 的 **先写磁盘,再写日志** 正好相反

mysql 写 **前日志** ,redis 写 **后日志**

在存储引擎内,任何更新操作都会先记录 redo log 后,并更新内存数据,然后再适当的时候将 redo log 的数据回写到磁盘里

由此可知 redo log 是存储引擎持有的

### redo log 组成

实际上 redo log 也是由两个部分组成

1. redo log buffer
2. redo log file

这也是为了解决内存和磁盘读写速率不一致的问题

所以 mysql 也需要提供一定的机制,保证将 redo log buffer 缓冲里面的数据,刷新到 redo log file 磁盘里持久化

mysql 运行在用户态,要想真正写入磁盘,必须通过 os 提供的 os buffer 进入内核态调用 `fsync()` 才能写入磁盘

![img.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7qp5h2r8mj30ca0a4tau.jpg)

mysql 提供 3 种写入 redo log 的配置

1. 延时写:不会在事务提交时写入 redo log,而是每秒写入 os buffer 后在调用 fsync()刷盘,发生宕机时会丢失大概 1s 的数据
2. 实时写,实时刷:事务提交后,写入 os buffer 立即刷新到磁盘,性能低,但是数据可靠性最高,即使宕机也不会丢数据
3. 实时写,延时刷:事务提交后,写入 os buffer 每隔 1s 中左右刷新磁盘,发生宕机会丢失大概 1s 的数据

整个 redo log 的数据结构类似以个环形数组,通过两个指针 `write_pos` 和 `check_point` 来记录读写进度

`write_pos` 记录当前已经写入的位置,随着新数据到来, `write_pos` 指针会不停地往后移动

`check_point` 记录当前正要落盘的位置,每次将数据落盘之后,`check_point` 也会往后移动

如果新数据的写入速度,超过了落盘的速度,导致 `write_pos` 和 `check_point` 相遇了

此时就需要暂停新数据的写入,现将一部分数据落盘后,空出来的 redo log 空间才允许新数据写入

![img_1.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7qp577ek3j30ll0cx764.jpg)


这样,当事务提交后,由于 mysql 写前日志的特性,即使还没来得及完成刷盘操作,也能通过 redo log 的 check_point 之后保存记录来恢复事务

### binlog

在存储引擎有 redo log 的存在,在 Server 层自然也有对应的 binlog 来记录日志

对比下 redo log 和 bin log 的不同点

1. redo log 写物理日志,bin log 写逻辑日志
   物理日志就是记录磁盘页的操作:在第 `x` 页磁盘,偏移量为 `y` 的位置,写入 `z` 个字节

binlog 写逻辑日志,也就是说 binlog 记录的是每一条实际操作的 sql 语句
由于 redo log 写的是物理日志,其记录了磁盘操作的本质,在数据恢复上就比逻辑日志的 binlog 快很多

2. redo log 是类似环形数组的循环写入,有固定的大小

而 binlog 则是顺序追加写入文件,这样记录了所有操作过的 sql 语句

redo log 由于其大小固定,不可能无限制地写入日志,所有 `check_point` 保证了在其之前的数据都是已经刷入磁盘的

不会保存历史记录,相对于 binlog 而言,节省了很多空间

同时由于 binlog 一只追加写,并没有什么标志位能够得知哪条记录之前的 sql 是已经刷入磁盘的,在做数据恢复的时候,难以定位哪些还未刷盘的日志

而 redo log 通过 `check point` 可以很方便的找到未刷盘的日志位置,从而进行数据恢复

3. redo log 是存储引擎 innoDB 独有的,binlog 是 mysql Server 层持有的,跟存储引擎无关

### 两阶段提交

以 `slq = 为 id = 2 的记录 c 值加 1` 为例,看看 mysql 具体的执行过程

![img_2.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7qp5x282nj30cs0hkdik.jpg)

1. 经过链接器,抛开查询缓存不谈,完成语法分析和分析,经过优化器来到执行器后

2. 首先执行器调用存储引擎的接口检索 `id=2` 的记录,检查记录是否存在于 buffer pool 当中,若命中则直接执行器,否则将会从磁盘里面读出数据所在的 **页** 放入内存当中

3. 执行器拿到 `id=2` 的记录后,对其 `c` 值完成加 1 操作后,在调用存储引擎的接口将其写入内存

4. 存储引擎将更新后的数据写入内存后,立即写入 redo log buffer 当中,此时 redo log 处于 `prepare` 状态,然后返回到执行器

5. 执行器接收到存储引擎完成数据写入后的响应,则记录当前 sql 到 binlog 里面,然后再调用存储引擎提交事务

6. 存储引擎接收到执行器提交事务后的请求后,将刚刚 redo log 当中的 `prepare` 状态更新为 `commit`

可以看到整个过程当中,redo log 分别在不同的时期处于两种不同的状态,这个特性被称作 mysql 的 `两阶段提交`

为何一个事务在写 redo log 的时候有两次操作,这个需要结合 binlog 解释下当事务提交时发生宕机后,如果通过 bin log 和 redo log 完成数据恢复

#### mysql 数据恢复时的规则

1. 若 redo log 里面有 commit 记录,则直接提交当前事务
2. 若 redo log 有 prepare 记录,则检查 binlog 是否有事务 commit 记录. 若 binlog 也有,则提交当前事务;若 binlog 没有,则回滚当前事务

#### 如果先写 redo log 再写 binlog

若数据更新后,redo log 已经完成写入,此时再写入 binlog 发生宕机

在恢复数据的时候,检查到 redo log 里面已经有事务的 commit 记录,此时应该提交当前事务到数据库

此时主库完成事务提交, `c` 值已经被更新为 2

但是由于 binlog 没有记录,导致通过 binlog 同步到从库时,从库缺少了当前事务,从而导致主备的数据不一致

#### 如果先写 binlog 再写 redo log

若数据更新后先写入 binlog,此时再写入 redo log 时发生宕机

在数据恢复的时候,检查 redo log 没有事务的 commit 的记录,从而主库缺少了当前事务的记录

而从库因为从 binlog 当中同步到了更新的事务,这也导致了主备的数据不一致

#### 两阶段提交如何避免发生宕机后数据恢复时主备数据不一致的问题

1. redo log prepare 之后,写 binlog 之前宕机

恢复数据时,检查到 redo log 有 prepare 的事务,再去检查 binlong 的情况,发现 binlog 里面没有记录事务

此时需要回滚 redo log 里面的事务,即主库回滚未提交的事务

这样主备之间通过 binlog 同步时,都没有宕机前未提交的事务,保证主备之间的数据一致性

2. binlog 完成,写 redo log commit 时宕机

数据恢复时,检查到 redo log 有 prepare 记录,再去检查 binlog 的情况,发现 binlog 里面也有事务的记录

此时需要将 redo log prepare 修改为 commit,将 binlog 里面记录的事务提交到主库

这样从库因为已经从主库同步了 binlog 执行了最新的事务,但是主库由于宕机导致事务没有提交,所以这时候检查后需要将 binlog 里面的时候重新在主库提交一遍,这样保证主备之间的数据一致性

