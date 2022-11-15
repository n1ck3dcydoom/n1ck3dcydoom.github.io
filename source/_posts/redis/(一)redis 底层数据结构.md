---
title: (一)redis 底层数据结构
date: 2022-11-09 09:25:39
index_img: /img/redis.png
categories:
  - redis
tags:
  - redis
---

### redis 对外提供的上层数据结构

1. String
2. List
3. Hash
4. Set
5. Sorted Set

这五大基本类型是 redis 对外提供的数据结构,在底层 redis 分别有自己实现的底层数据结构去对应这些基本类型,分别是:

1. 简单动态字符串
2. 压缩列表
3. 双向链表
4. 哈希表
5. 跳表
6. 整数数组

### redis 所有的 kv 通过什么形式组织

redis 内部维护了一个巨大的 `hash` 表,这个 `hash` 表记录了实例里面的所有键值对

这个巨大的 `hash` 表其实就是一个数组,而数组每个元素都是一个 `hash` 桶,每个 `hash` 桶里面才保存了键值对数据

每个 `hash` 桶内其实也不是保存的真正的键值对,而是维护的指向键值对的指针,即 `*key` 和 `*value`

![img.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7zqfxowfbj30j609dgnk.jpg)

全局哈希表可以在 `O(1)` 的时间内快速查找到对应的键值对,只需要由 `hash` 函数计算出 `key` 的位置就能找到 `hash` 桶的位置,然后访问 `hash` 桶内的 `*value` 指针即可找到对应的值

#### 简单动态字符串 (SDS)

`SDS` 并不是一个简单的以 `\0` 结尾的字符数组,而是一个有着相对复杂结构的抽象类型,并且支持动态扩展

一个 `SDS` 的结构如下:

```c
struct sdshdr {
    // 记录buf数组中已使用字节的数量，即SDS所保存字符串的长度
    unsigned int len;
    // 记录buf数据中未使用的字节数量
    unsigned int free;
    // 字节数组，用于保存字符串
    char buf[];
};
```

![img_6.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7zqevfpm1j30jv0dvgp1.jpg)

`raw` 类型的 `redisObj` 和 `SDS` 内存不连续,需要两次内存分配

`EMBSTR` 的 `redisObj` 和 `SDS` 内存连续,只需要一次内存分配

但是 `EMBSTR` 用来保存短字符的优化,如果字符串长度增加需要重新分配时,整个 `redisObj` 和 `SDS` 都需要重新分配

1. 获取字符串长度的时候,不用遍历底层的字符数组,只需要取出 `len` 属性即可
2. 杜绝缓冲区溢出,在做字符串拼接的时候,首先会检查 `len` 和 `free` 判断内存是否满足需要;如果内存不足,则会对 `SDS` 做扩展后在才做
3. 减少字符串的重新分配次数; 字符串扩展时,实际的内存分配会比对应的长度更多,减少连续扩展的内存分配次数; 字符串缩短时,已经分配的内存空间不会立即回收,以免下次需要用的时候再次分配内存
4. 保证二进制文件安全,不通过 `\0` 判断是否达到末位,而是通过 `len` 和 `free` 属性计算末尾

在实际做字符串扩展的时候,如果新字符串长度小于 `1M` 时,扩展会分配一倍大小的空间,如果超过了 `1M` 则每次多分配 `1M` 的空间

#### 压缩列表 zlist

一个 `zlist` 的结构如下:

![img_1.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7zqf2b902j30f102nabh.jpg)

* zlbytes: 记录了整个压缩列表占用的字节数
* zltail: 记录了压缩列表最后一个元素的偏移量,用于快速定位列表尾部
* zllen: 记录了压缩列表实际保存的元素 `entry` 个数 (类似于 `SDS` 的 `len` 属性)
* entry...: 压缩列表实际存储每个元素
* zlend: 压缩列表尾部的终止符 (类似于 `\0` 的作用),标记一个压缩列表结束

`entry` 的结构如下:

![img_2.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7zqf7rasoj30fe03sta9.jpg)

1. 一般情况下
  * prevlen: 上一个 `entry` 的大小
  * encoding: 当前 `entry` 的编码格式
  * entry-data: 当前 `entry` 实际存储的内容
2. 如果保存的是整数类型
   * `encoding` 和 `entry-data` 会合并在 `encoding` 里面,通过前几位判断整型长度,用于节约空间
   * 且会尝试把 `int` 转化为 `string` 类型

压缩列表的设计就是为了节省内存,顺序数组由于每个元素的类型和大小都是固定的,会导致内存使用率低

而压缩列表的每个节点 `entry` 通过保存编码让节点刚好分配够用的长度,来达到提高内存使用率的效果;同时由于每个 `entry` 的长度不一样,就必须记录上一个 `entry` 的偏移量,这样才能在链表里面完成寻址遍历

压缩列表的缺点:
1. 在做缩容的时候,会直接回收对应的内存空间,而不是类似 `SDS` 会保留; 这样每次新添加元素的时候都会产生一次内存分配
2. 如果上一个节点的长度超过了 254 个字节,将会导致扩容,而扩容后的 `entry` 长度由于超过了 254 字节,下一个节点的 `prevlen` 属性无法保存 254 字节的信息,就只能跟着扩容; 这样容易引起一整条链路上的所有节点都对 `prevlen` 属性进行扩容操作

![img_7.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7zqfdaf2tj30u70gbq89.jpg)

#### 双向链表

在压缩列表 `zlist` 的基础上,每个节点都有维护指向上一个节点 `*prev` 和下一个节点 `*next` 的指针,构成一个双向链表

![img_3.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7zqfhppbkj30m20b7mzk.jpg)

可以看到,在双向链表的每个节点 `quicklistEntry` 上,其实保存的实质性内容就是一个 `zlist` 压缩列表

#### 字典

字典的结构如下:

```c
typedef struct dictht{
    //哈希表数组
    dictEntry **table;
    //哈希表大小
    unsigned long size;
    //哈希表大小掩码，用于计算索引值
    //总是等于 size-1
    unsigned long sizemask;
    //该哈希表已有节点的数量
    unsigned long used;
 
}dictht
```

可以看到字典其实是由 `dictEntry` 组成的数组构成,而 `dictEntry` 的结构如下

```c
typedef struct dictEntry{
     //键
     void *key;
     //值
     union{
          void *val;
          uint64_tu64;
          int64_ts64;
     }v;
 
     //指向下一个哈希表节点，形成链表
     struct dictEntry *next;
}dictEntry
```

每个 `dictEntry` 都包含一个 `kv` 对,还包含一个指向下一个节点的 `*next` 指针,这说明 redis 在解决哈希冲突的问题上,采取的是 `拉链法` 

当 `dictEntry` 节点个数在不停地增加时,哈希碰撞的概率也会加大,最严重的时候会导致字典退化为一个单链表,所以 redis 通过 `rehash` 操作来扩展字典的大小,将原来集中的 `kv` 再次分散到不同的 `dictEntry` 上,减小哈希碰撞的发生

redis 内部维护了两个全局的 `hash table` 起作用类似于 `AB` 表,当发生 `rehash` 时,就对新 `table` 做扩容操作,然后把旧 `table` 的数据全部拷贝过去,再释放旧 `table` 的空间

对于少量数据,这个操作可以瞬间完成; 对于大量数据的全量拷贝,势必造成 redis 的阻塞; 所以 redis 引入了 `渐进式 rehash`

1. 对新 `table` 进行扩容操作,但是并不立即拷贝旧的数据
2. 对旧 `table` 做读取操作时,每次读出的 `hash` 桶就会拷贝到新 `table` 上尽行 `rehash`

这样每次写都是写旧的 `table`  而读可能新旧 `table` 都能提供服务; 减少了一次性全量数据拷贝阻塞 redis 的风险

![img_8.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7zqfn2s4oj30x10jkqa6.jpg)

#### 整数数组

数组的结构很简单:

```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

一个记录长度的变量 `len` 一个记录编码的变量 `encoding` 一个实际保存变量的整型数组 `contents` 

#### 跳表

跳表可以说是 redis 最复杂的数据结构了,要好好理解下

考虑压缩列表或者双端列表的场景,由于这两者都不支持随机访问,如果要查找某个元素的话,只能从头到尾依次遍历

而作为有序集合,其性质本身保证了有序,对于有序的查找,使用二分查找能加快查找进度

跳表在原来一层链表的基础上,引入了多层索引结构,每次从最高层最左端开始查找,其查找过程类似于二分查找

![img_5.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7zqfs56nnj30k60cqq53.jpg)

在 redis 的实际运用当中,跳表 `zskiplist` 由指向头部的指针 `*header`, 指向尾部的指针 `*tail`, 当前跳表内最高的节点层数(表头节点不算) `level`, 当前跳表的长度(即节点个数,表头节点不算) `length`

跳表的结构定义如下:

```c
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;  // 跳表节点保存的数据,一个 SDS 简单动态字符串
    double score;
    struct zskiplistNode *backward;  // 当前节点的前驱节点
    struct zskiplistLevel {  // 当前节点所属的层级
        struct zskiplistNode *forward;
        unsigned int span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

`zskiplist` 跳表定义了两个指针,分别指向跳表的头部 `*header` 和尾部 `*tail` ,类型是跳表节点 `zskiplistNode`

`level[]` 表示当前节点在哪些层里面出现,例如上图的节点 `o1` 就出现在 `L1,L2,L3,L4` 当中,而节点 `o2` 自出现在 `L1,L2` 当中

`obj` 跳表节点存储的实际数据

`*backward` 后退指针,当检索跳表节点时,若发现当前节点大于检索值,则需要后退并且进入下一层检索

`score` 跳表的节点按照分值从小到大有序排列

对于层 `level`:

`*forward` 前进指针用于检索下一个节点

`span` 跨度,表示当前节点在当前层的下一个节点与当前节点之间有多少步长,例如对于节点 `o1` 在 `L4` 的节点来说,其跨度就是 `2` (走两步到当前层的下一个节点); 所有指向 `NULL` 的节点跨度为 `0`

跳表插入节点时,如果数据比较几种,可能会退化成一条单链表; 所以必须在插入时更新节点的层数,redis 通过一个随机函数来决定插入节点在下一层是否应该继续插入,每次进入下一层的概率大概为 `1/4`

为何 redis 选用跳表而不是红黑树

因为红黑树在做插入和删除时,涉及到树的旋转,其效率并不如跳表操作指针高; 而且在查询效率上分布均匀的跳表和红黑树几乎没有区别