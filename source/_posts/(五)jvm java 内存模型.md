---
title: (五)jvm java 内存模型
date: 2022-11-15 17:27:55
index_img: /img/java.png
categories:
  - jvm
tags:
  - jvm
---

### java 内存模型

java 内存模型主要关注的线程访问程序当中各种变量的规则,主要是在虚拟机主存中把变量读出放到内存当中,以及把内存当中的变量写入到虚拟机的主存里

java 内存模型规定所有的变量都存储于 `主存` 当中,每个工作线程还有自己私有的工作内存空间; 每个线程只能访问和操作自己工作内存空间的变量和数据,不允许直接操作主存当中的变量和数据;
不同的工作线程之间也不允许互相访问其他线程的工作内存

线程,线程工作内存和主存的关系如下

![img.png](https://tva1.sinaimg.cn/large/008vK57jgy1h86slkrt6mj30l408g3zw.jpg)

### volatile 关键字

`volatile` 关键字可以说是 jvm 提供的最轻量的同步机制,jvm 为其赋予了两项特征

1. 被 `volatile` 关键字修饰的变量对所有线程可见

这里的 `可见性` 指的是当一个线程改变了 `volatile` 变量之后,其他线程能够立即观察到这个变量的改变;
而普通变量则不行,因为普通变量的修改发生于线程的工作内存当中,而工作内存是线程私有的,只有当工作内存的改动刷回主内存后,其他线程才能观察到变量发生了改变

但是 `volatile` 并不能一定保证并发安全,考虑如下场景

有个 `volatile` 的 `int` 型变量 `a = 0` ,有 100 个线程并发对这个变量 `a` 执行自增操作,即 `a++;`

最终的结果 `a` 并不一定等于 100,而是一个小于 100 的值

这是因为 `volatile` 只保证变量的可见性,但是对于并发操作并不能保证一定是安全的,对于自增运算 `a++;`,其本质上是由多条指令构成的

```
get a
a = a + 1
put a
```

对于 `get a` 来说, `volatile` 能够保证所有线程都能读到 `a` 的值且是正确的

但是对于 `a = a + 1` 或者 `put a` 来说, `volatile` 无法保证这里的原子性,也就是说当一个线程写回 `a` 的时候,其他线程很有可能已经修改过了,此时就会把历史数据给写回; 导致最终 `a` 并不等于 100

那么 `volatile` 控制并发的最好场景,其实是类似于实现一个观察者模式,如下

```
volatile boolean flag = true;

public void shutdown() {
    flag = false;
}

public void run() {
    while (flag) {
        // do something
    }
}
```

当调用 `shutdown()` 方法后,其他线程的 `run()` 方法会立即观察到 `flag` 变量发生了改变,从而全部停止下来

2. `volatile` 会禁止指令重排序

对于普通变量的修改,jvm 会保证对其操作的最终结果是符合预期的,但是中间涉及到的计算过程 jvm 可能会对起进行指令重排序优化,也就是说代码的执行顺序可能并不与看到的保持一致

```
b = getb()
c = getc()
b = b + 2;
c = c + 3;
a = b + c
```

jvm 能够保证最终 `a` 的计算结果符合预期,但是对于中间过程,先获取 `b` 还是 `c` 以及获取之后先计算 `b` 还是 `c` 可能会发生重排序优化

而有些场景下,如果没有禁止指令的重排序可能会产生一些不符合预期的表现

```
// 这里 done 变量必须设置为 volatile
volatile boolean done = false

// load some config 
// init some variable

// check flag
while (!done) {
    // other threads will wait for done
    sleep();
}

// do something with config and variable
```

如果这里的 `done` 是一个普通变量,那么在指令重排序之后,很有可能 `check` 逻辑会被提前到 `load` 之前; 这样会导致其他线程在 `load 或者 init` 之前就开始使用这些没有初始化的变量了

还有个禁止指令重排序的运用场景就是单例模式的初始化时 `双重校验` 机制

```java
public class Singleton {
    private volatile static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    // 非原子操作,指令重排序发生非预期的表现
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }

    public static void main(String[] args) {
        Singleton.getInstance();
    }
}
```

这里的 `new` 操作并不是一个原子操作,很有可能在指令重排序之后,导致构造函数在整个对象初始化前就已经执行完了

`new` 操作的过程实际上分为三个步骤:
1. 给对象分配空间
2. 调用对象的构造函数进行初始化
3. 将对象的引用指向初始化后的空间(此时对象才不为 null)

而指令重排序之后,很有可能步骤变成了 132; 此时 `instance` 已经不为 `null` 了,但是其他线程拿到后还是无法使用,因为没有完成初始化只是一个半成品

即 `第一个进入同步代码块的线程,正在对 instance 进行初始化,但是因为指令重排序,先将对象应用指向了内存空间,还没来得及初始化的时候; 另一个并发线程进来发现此时的 instance 已经不为 null 而返回使用了`

所以此时需要使用 `volatile` 来禁止后续的指令被 jvm 重排序,保证双重校验机制的正常工作

`volatile` 通过对后面的代码添加 `内存屏障` ,这样 jvm 就不能把后面的指令放到 `内存屏障` 之前执行了

回到 java 的内存模型,当一个线程需要本地工作内存内的 `volatile` 变量的时,其语义要求必须从主存中刷新得到最新值,保证变量的改变能够被其他线程立即观察到,同时也要求对于变量的改动后必须立即刷回主存