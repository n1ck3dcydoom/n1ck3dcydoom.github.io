---
title: (十六)mysql 读写分离
date: 2022-11-08 09:09:29
index_img: /img/mysql.png
categories:
  - mysql
tags:
  - mysql
---

### 一主多从架构下的读写分离

先回顾下一主多从的结构图

![img.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7y196hmklj30j00cttan.jpg)

读写分离的主要目的就是为了 **分摊主库上的请求压力** ,上面的架构主要是在 `client` 客户端主动做负载均衡; 在这种模式下,会把数据库的配置信息放到客户端的连接层,由客户端主动选择处理写请求和读请求的数据库实例

还有一种就是在客户端和数据库之间新增一个代理网关(正向代理:发送方不知道实际处理方是谁,只知道一个处理方代理),这样客户端就不用区分读写请求,只需要把请求全部发送给代理即可; 具体的读写请求分发到主库还是从库全部由代理判断

![img_1.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7y19d7bvhj30hb0dvq4f.jpg)

无论是那种架构,都存在一个问题: **由于存在主从延迟,客户端刚刚写入一条数据后马上发起查询请求,此时处理读请求的从库很可能还没来得及完成主从同步,导致数据不一致**

读写分离解决数据不一致的方案主要有以下几种:

1. 强制查主库
2. `sleep` 等待
3. 判断是否存在主从延迟
4. `semi-sync`
5. 等待主库位点
6. 等待主库 `GTID`

#### 强制查主库

业务方将请求分为以下两类:

1. 有强一致性要求的请求,强制把这部分读请求打到主库上,保证数据一致性
2. 接受一定时间内数据不一致的请求,把这部分读请求打到从库上,由主从同步保证一定时间内的最终一致性即可

考虑一种极端场景,所有的请求都有强一致性要求; 这样一主多从的读写分离结构就退化为只有一个主库的单节点结构; 失去了读写分离的意义

#### sleep 等待

在主库完成写请求之后,下一个强一致性读请求到达业务方,在发送到从库之前先 `sleep(x)` 等待一段时间 `x`

假设当前的主从延迟在 `2s` 以内,这样 `sleep(2)` 之后就可以认为从库已经同步了刚刚最新的写请求,此时业务方再把读请求发送到从库上,有很大概率能拿到最新的数据

这个方案仍然是不可靠的,假如主从延迟并不稳定在固定值之内

1. 如果主从延迟只有 `0.5s` 了,此时等待 `2s` 显然时间过长
2. 如果主从延迟超过了 `2s`, 即使有等待的时间,还是会读取到历史脏数据,导致数据不一致的问题

#### 判断是否存在主从延迟

1. 通过主从延迟时间判断是否存在主从延迟,每次查询前,先获取 `seconds_behind_master` 值,判断主从延迟的时间是否等于 0; 如果等于 0 就发起读请求,否则就等待主从延迟的时间变为 0 后才执行查询
2. 通过位点判断是否存在主从延迟,查询主库上的最新位点和从库上的最新位点,如果两个位点相同,则认为没有主从延迟可以执行查请求;否则就等待直到位点相同后才执行查询
3. 通过 `GTID` 判断是否存在主从延迟,查询主库的 `GTID` 集合与从库的 `GTID` 集合是否相同; 若相同则执行查询请求;否则就等待直到相同后才执行查询

每次判断是否存在主从延迟,无论选用那种方式判断,也都还是会发生数据不一致的问题

考虑如下场景,主库 A 已经写入 2 个事务到 `binlog` ,并且发送了前两个事务到从库 B 里面,且从库 `B` 也已经同步完成; 此时主库 A 上一个新事务 3 刚刚写入完成,还没来得及发送

![img_2.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7y19iv59mj30bq0a6wfe.jpg)

此时无论是位点还是 `GTID` 都会认为没有发生主从延迟,但严格上来说此时的读请求,在从库上仍然是查询不到事务 3 的

#### semi-sync 半同步

`semi-sync` 半同步的设计如下:

1. 事务提交后,主库把 `binlog` 发送给从库
2. 从库收到 `binlog` 在消费 `relay log` 并完成从库 `binlog` 的写入后,会给主库发送一个 `ack` 表示完成当前事务的同步
3. 主库收到 `ack` 之后,才会返回给客户端表示当前事务已经完成

也就是说 `semi-sync` 半同步机制保证了所有返回给客户端的事务请求,都是在从库上完成了同步的

在一主多从结构下,`semi-sync` 也可能会犯错

如果一个从库收到同步 `binlog` 的请求后,完成同步并向主库回复了 `ack`; 此时主库认为这个事务的同步已经完成,则向客户端回复成功; 假如针对这个事务数据的读请求进来,被代理分配到一个还没有完成同步的从库上,此时又产生了数据不一致的问题

而且如果是基于位点或者 `GTID` 的半同步,如果主库压力很大其不停地在推进位点或者 `GTID` 而从库可能会一直在追赶主库的同步进度

由于主从延迟加上 `semi-sync` 半同步机制,对于一个事务来说,从库迟迟不会向主库回复 `ack` ,从而主库不会向客户端回复成功; 导致整个业务阻塞

对于一笔刚写入完成的事务,并不需要完全等待从库完成同步后,才回复 `ack` 可以看一下主从同步之间的中间状态

![img_3.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7y19r2wa6j30oe0ixdk3.jpg)

对于等待位点检测主从延迟的方案来说,上面的请求中,从库和主库的位点一直都不相同; 如果一个查询必须要等待同步完成才能响应的话,上面的步骤将会导致整个查询不可用

其实当一个事务已经同步写入从库的 `binlog` 当中并完成刷盘,此时就可以认为这个事务的同步已经完成了; 至于其他事务导致的主从延迟,其实和当前已同步完成的事务无关

总结,使用 `semi-sync` 且检测主从延迟的方案,存在两个问题

1. 一主多从,从某些还未来得及同步主库的从库查询时,可能查到没有更新的历史数据
2. 如果出从延迟持续存在,从库等延迟结束,主库等从库回复 `ack` ,这里可能导致长时间的等待

#### 等待主库位点的操作

从库上有一条指令,可以用于主动检测与主库之间位点的差异

```sql
select master_pos_wait(file, pos, timeout)
```

1. `file` 指定了检测主库的 `binlog` 文件, `pos` 指定了检测事务的位置
2. `timeout` 制定这个函数检测时的超时时间

这条命令的返回值如下:

1. 正常返回一个正整数 `M` 表示从命令开始执行,到同步 `file` 的 `pos` 位置时,执行了多少个事务
2. 如果主从同步发生异常,返回 NULL
3. 如果超时,返回 `-1`
4. 如果开始执行的时候,就已经超过 `pos` 位置,返回 0

对于上面 `semi-sync` 产生的问题,看看等待主库位点如何解决

1. 主库事务 `trx1` 执行完成之后,立马查询当前主库的 `binlog` 文件 `file` 和最新的位点 `pos`
2. 随机选择一个从库进行查询
3. 从库查询前先执行上述语句,检测主从同步之间的位点差异
4. 如果返回值 `>=0` 则表示当前从库对于主库刚刚执行的事务 `trx1` 已经执行过了,可以由从库返回读请求
5. 否则说明当前从库还未来得及同步刚刚主库的事务,此时读请求交给主库响应

* 假设检测主从延迟的等待时长为 `1s` ,如果这 `1s` 内,返回了大于 0 的整数 `M` ,则说明在这 `1s` 内,从库从某个之前的位置同步到 `pos` 时,应用了 `M` 个事务; 虽然等待了一些时间,但是好歹追上了主从延迟,所以从库可以相应对应的读请求

* 如果返回值等于 0,则表示从库当前同步主库的位点已经领先于主库执行事务 `trx1` 时的位点,此时从库肯定也是完成了事务 `trx1` 的同步,可以响应读请求

* 如果返回值等于 `-1` 或者 `NULL` 则说明在等待时间内从库都没能追上主从延迟,此时记录的肯定是历史数据,必须交给主库响应当前查询请求

图例如下:

![img_4.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7y19xa2l8j30mh0g5n2i.jpg)

退化到主库查询是一种通用的兜底方案,因为读请求不可能无限等待从库去追赶主从延迟;超时后就必须要由主库响应请求

#### 等待 GTID 的操作

同理,从库通过主库发送过来的事务 `GTID` ,判断自己当前的 `GTID` 集合是否包含对应事务的 `GTID` 

同样,超时后结束等待; 或者在等待时事务的 `GTID` 加入当前集合; 或者在检测时当前集合就已经包含了 `GTID` 