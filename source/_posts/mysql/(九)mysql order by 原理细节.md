---
title: (九)mysql order by 原理细节
date: 2022-11-05 16:29:47
index_img: /img/mysql.png
categories:
  - mysql
tags:
  - mysql
---

## order by

假设有一张城市表,记录了城市里面每个人的名字,你年龄的信息

现在需要对城市是 `杭州` 的所有人,按照年龄排序后返回前 1000 人

```sql
select city,name,age from t where city='杭州' order by name limit 999;
```

### 全字段排序

为了避免扫描全表,很自然的在 `where` 语句查询的字段 `city` 上加上索引

![img.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7ugsbrl2wj30th0fbjuw.jpg)

使用 `explain` 指令查看执行情况

![img_1.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7ugsiczttj31bh041jxx.jpg)

在 `extra` 一列里面可以看到 `using filesort` 表示进行了排序,实际 mysql 会为每个需要排序的线程分配一块内存区域专门用来加速排序,称为 `sort_buffer`

一个完整的查询流程如下:
1. 初始化 `sort_buffer` ,里面有 3 个字段分别是 `city,name,age` 
2. 从 `city` 索引开始查找,找到第一个满足条件的主键 id ,即 `id_x`
3. 回表得到 `id_x` 对应的完整数据,取出 `name,age` 和 `city` 一起放入 `sort_buffer`
4. 从 `city` 索引继续取出下一个满足条件的主键 id,重复 34 过程,直到 `city` 不再满足条件
5. 对 `sort_buffer` 里面的所有数据做快速排序
6. 将排序结果前 1000 条返回

![img_2.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7ugsnhzo9j30m60gbtd4.jpg)

需要注意的是,由于每次分配的 `sourt_buffer` 大小是固定的,其不一定能够完全放下所有需要排序的记录; 所以 mysql 会根据实际情况,在内存中完成排序,或者在磁盘上完成排序; 如果在磁盘上进行排序,就不得不依赖磁盘创建临时文件辅助排序

打开 msyql 的 `optimizer_trace` 查看优化器的输出

![img_3.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7ugt1fzvoj30mt05twgy.jpg)

可以看到在磁盘排序时,使用了 12 个临时文件,这是因为在磁盘上排序一般使用 **归并排序** ,将整个数据集分为若干个子文件,对子文件进行排序后再合并为一个有序的大文件

### rowid 排序

上面的全字段排序会将所有需要排序的字段取出放到 `sort_buffer` 里面,如果字段多,数据行多的话,就会导致 `sort_buffer` 里面要保存的数据太多,由于内存是有限制的,所以很快 `sort_buffer` 就会被占满,这样就需要把数据分散到多个临时文件里面做归并排序,其性能会差很多

可以通过修改 mysql 控制排序数据行长度的参数,让 myslq 选择另一种排序方式 `rowid` 排序

```sql
set max_length_for_sort_date=16
```

如果单行总长度超过这个值,就该用 `rowid` 排序

`rowid` 排序不把单行所有字段放入 `sort_buffer` 里面,仅仅把主键 id 和需要排序的列放入 `sort_buffer`, 由于 `sort_buffer` 里面缺少了单行的所有信息,所以不能够在完成排序后直接返回,需要多添加一步回表查询的过程

当 `sort_buffer` 完成排序后,遍历前 1000 条记录,通过主键 id 再次回表查询得到所有的数据行返回

![img_4.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7ugst18g8j30n10hhtf6.jpg)

### 全字段排序和 rowid 排序

如果 mysql 检测到内存足够,就会使用全字段排序,因为这样能够节省一次回表的查询开销

如果内存不足,则会改用 `rowid` 排序,这样可以提高一次性排序的行数,但是对应的代价就是多了一次回表查询

总的来说,mysql 体现了一个原则就是: 内存够用就要尽可能利用内存,减少磁盘的访问

### 使用覆盖索引来加速排序过程

如果查询的列天然有序,则可以进一步提高 `order by` 语句的效率

对于仍然是查询 `city`, `name`, `age` 的 sql,如果在 `city` 和 `name` 上建立联合索引

这样 `where` 语句使用 `city` 索引检索的时候, `name` 也是自然有序的,这样可以节省排序的步骤,只需要一次回表查询即可返回结果集

![img_5.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7ugt8teb7j30lq08xacf.jpg)

如果在 `city`, `name`, `age` 三个字段上建立联合索引,这样检索的时候,索引树里面就已经全部包含了所有需要查询的数据,而且自然有序,这样的覆盖索引还能再减少一次回表查询即可返回结果集

![img_6.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7ugtf3quvj30he09cmyt.jpg)


### 随机排序

```sql
order by rand()
```

当添加上 `rand()` 随机函数之后,如果再使用 `explain` 查看执行过程的话,会发现 `extra` 列上多了个 `using temporary` 表示当前查询会使用 **临时表**

![img_7.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7uhsdqh5nj307w01yq3k.jpg)

对于带有临时内存表的排序,mysql 应当选择何种排序算法:
1. 如果是 `InnoDB` 为了减少磁盘访问,优先选择全字段排序
2. 如果是带有临时内存表的排序,回表相当于访问内存,其性能几乎不会受到影响,此时会选择 `rowid` 排序

```sql
select word from t order by rand() limit3
```
其排序的查询过程如下:
1. 创建一张临时内存表,表里面保存两个字段 浮点数 `R` 和 字符串 `W`
2. 从主表中按照主键顺序取出 `word`,并且调用 `rand()` 函数生成一个随机浮点数,将这个随机浮点数和 `word` 放入内存表的 `R` 和 `W` 字段
3. 在内存表上排序,初始化 `sort_buffer`, 其包含两个字段 浮点数和字符串
4. 从内存表中一行一行地取出 `R` 和 `位置信息` 将其放入 `sort_buffer`,然后在 `sort_buffer` 里面完成排序
5. 排序完成后,取前 3 个数据返回

![img_8.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7uhskxbg6j30o30hrjvf.jpg)

其中内存表的 `内存信息` 就是 mysql 为我们隐式生成的 `rowid`, 其实如果一张表没有显式指出主键的话,mysql 就会隐式的创建 `rowid` 当做主键 id 使用

如果单行数据超过了 mysql 配置的最大单行数据长度,则全字段排序会被转换为 `rowid` 排序,会导致在磁盘上通过临时文件进行归并排序

对于 **随机排序** 且还带有 `limit` 关键字的情况,归并排序完成后所有的数据都是有序的,此时只用取前 `limit` 个,但是实际上除了前 `limit` 个以外的数据,并不关系其是否有序,所以 mysql 在 5.6 版本引入了 `堆排序` 

构造最大堆或者最小堆,初始化容量为 `limit` 个元素,将后续的元素依次添加到堆里,完成遍历后,堆里就保留了 **最大** 或者 **最小** 的前 `limit` 个元素,而后面的则无需关心是否有序

![img_9.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7uhsquhy8j30o00dqagy.jpg)

无论如何,使用 `order by rand()` 都会导致复杂的排序计算

优雅的随机排序应当使用如下操作

1. 统计表的总行数 `C`
2. 计算随机数 `R = floor(C*rand())` 其中 `floor()` 函数负责向下取整
3. `select xxxx limit Y,1` 表示从 `Y` 开始取出 `offer = ` 即 `Y+1` 行

若要得到 `X` 个随机记录,则将上述操作重复 `X` 次,这样可以避免主键空洞导致的伪随机