---
title: gin 源码解读 (一)
date: 2022-10-28 02:14:31
index_img: /img/gin.png
categories:
  - gin
tags:
  - gin
---


## 走进 Gin 的大门

### 启动 gin 服务

启动 gin 服务很简单,代码里面只需要两行

```go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")
}
```

跟进 `r.Run()` 方法一探究竟

### Run() 方法

![img01.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7kefvpgnuj30m709ljx3.jpg)

可以看到 `Run()` 方法大概做了 3 件事

1. 解析配置的受信任 CIDRs 地址
2. 如果有在启动时指定监听的地址,就解析对应 `addr` 地址
3. 传入解析后的 `addr` 地址,并监听这个地址上的 http 事件

通过上面的注释可知,其本质是 `http.ListenAndServe(addr, router)` 的一个实现

且 `Run()` 方法是一个阻塞方法,将会阻塞调用协程,除非异常发生

了解过 `Socket` 编程的应该都知道, `socket.Accept()` 就是一个阻塞方法,它会一直阻塞监听套接字绑定的 ip 和端口,直到收到请求数据才返回

可见 `http.ListenAndServe()` 也是一直阻塞并监听端口事件发生的

### http.ListenAndServe

这是 gin 的核心函数,其依赖于 go 语言内置的 `net` 库实现

![img02.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7kefvnvo2j30ir06ladw.jpg)

注意,外面传入的是 `gin.engine` 到这里变成了 go 语言 `net` 库的 `Handler` 接口对象

可以看到的是 `gin.engine` 是实现了 `Hanler` 接口的

并且后续所有的 `request` 请求和 `response` 响应也都是通过这个 `Handler` 接口实现的

### gin 如何通过 Handler 接口实现接收和响应请求的

跟进 `Handler` 接口,找到对应的 gin 实现 engine 对象

![img03.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7kefvra9xj30jf06w42g.jpg)

`engin.pool.Get()` 获取一个 gin 自己封装的 `Context` 对象,注意这里不是 go 原生的 `Context` 对象

可以看到 gin 使用了池化技术,做到内存复用,避免频繁的创建上下文

初始化 `Context` 对象时,把接收到的请求 `req` 也放到上下文里

进一步跟进 `handleHTTPRequest()` 函数看看 gin 究竟是如何处理请求的

```go
package gin

func (engine *Engine) handleHTTPRequest(c *Context) {
	httpMethod := c.Request.Method // 解析 method
	rPath := c.Request.URL.Path    // 解析 url path
	unescape := false
	// {一些对 url path 的额外处理}

	// gin 维护了所有 method 的路由树集合
	t := engine.trees
	for i, tl := 0, len(t); i < tl; i++ {
		// 遍历路由树集合找到当前 method 对应的的路由树
		if t[i].method != httpMethod {
			continue
		}
		root := t[i].root
		// 从 root 节点开始查找路由树,直到查找到 path 对应的路由
		value := root.getValue(rPath, c.params, unescape)
		if value.params != nil {
			c.Params = *value.params
		}
		// 如果路由有配置 handler 函数
		if value.handlers != nil {
			c.handlers = value.handlers
			c.fullPath = value.fullPath
			// Next() 函数会依次执行路由配置的所有 handler 函数
			c.Next()
			c.writermem.WriteHeaderNow()
			return
		}
		// {后续处理}
	}
	// {后续处理}
}
```

这里有两个关键点 `engine.trees` 路由树 和 `Next()` 函数

### 先来看 Next() 函数

![img04.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7kefvsw2xj30io05y0vq.jpg)

还记得之前的 `gin.engine` 对象吗? `engine` 实现了 `ServeHTTP` 接口,在里面封装了 gin 关于 http 请求的处理

其中 `c.reset()` 初始化 `Context` 上下文对象的时候,有设置 index 的值为 -1

![img05.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7kefvmytfj309203fgm8.jpg)

这说明还没有 `handler` 函数处理的时候,`Context` 上下文里面保存的处理索引是从 `-1` 开始计算

`Next()` 函数里,一来就让 index 自增 `c.index++` 后 index = 0,说明从第 0 个 `handler` 函数开始依次执行

而 `handleHTTPRequest()` 函数在从路由树当中找到请求对应的路由节点后,如果发现这个路由有配置过 `handler` 函数,则会把路由配置的 `handler` 函数全部赋值给 `Context` 上下文 `c` 里

这样就相当于通过上下文 `c` 依次执行路由配置的 `handler` 函数处理请求,而每次执行 `handler` 之后,都会使 `index` 自增

```go
package gin

func (c *Context) Next() {
	// 关注核心代码
	c.handlers[c.index](c)
	// c.index 表示当前待执行的索引
	// c.handlers[] 数组
	// c.handlers[c.index] 相当于取出处于 index 的 handler 函数
	// 而 handler 函数的签名是 func (c *Context)
	// 可以理解为 c.handlers[c.index] = func (c *Context)
	// 所以 c.handlers[c.index](c) 相当于把 c 又当做参数传给了 handler 函数
}
```

除非设置了 `AbortIndex` 的值终止 `handlers` 的遍历,否则会执行完所有配置的 `handlers` 函数才会结束请求的处理

### engine.trees 路由树

接下来在看看路由树的是怎么实现的

一路跟进去看源码

![img06.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7kefvmodrj308903yq3f.jpg)

可以看到 gin 维护了一个 `[]methodTree` 数组,数组里面每个元素都是一个路由树节点对象,分别保存了当前路由的方法 `method` 和当前路由的根节点 `root`

继续跟进树节点 `node` 看看怎么实现的

![img07.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7kefvxqjkj30m5062q5u.jpg)

其实路由树就是一棵 **紧凑前缀树**

前缀树利用字符串的公共前缀来减少查询时间,而路由则天生符合前缀的特性

例如有以下几个路由:

1. /
2. /order
3. /order/:name
4. /order/:id

可以看到这些路由都有根节点 `/` ,同时下面三个路由都有重复节点 `/order`

普通的前缀树保存每个字符,而 gin 使用 **紧凑前缀树** 保存每个路由节点,减少了内存的使用,提高查询效率

此时生成的路由前缀树如下:

![img08.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7kefvvkzyj307007z74p.jpg)

通过路由树可以快速的检索到对应的的路由节点,至于路由如何注册在这里暂且不表

### 回到 handleHTTPRequest() 方法

在经历了路由检索, `handler` 函数的处理,此时一笔请求就已经全部执行完毕了

通过 `c.WriteHeaderNow` 将 http 状态写回 header 后,当前请求的处理协程结束

主协程仍然阻塞在 `accept()` 监听方法上,通过 channel 来启动协程继续处理后续的请求

至此,一笔请求就被完整的处理完并返回给调用方

剩余的细节:
1. go 内置的 net 网络通信库,如何建立连接,监听请求,前置处理
2. gin 注册路由如何生成路由树

后面再说吧