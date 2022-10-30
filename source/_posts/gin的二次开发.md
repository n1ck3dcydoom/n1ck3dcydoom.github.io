---
title: gin 的二次开发
date: 2022-10-31 00:02:21
index_img: /img/gin.png
categories:
- gin
tags:
- 源码
---

### 结合业务对 gin 做二次开发

考虑下 gin 或者 go 原生的 http 包究竟为我们提供了哪些扩展点 **根据开闭原则,对修改关闭,对扩展开放**

1. go http Server

2. go http 包为我们提供了 Handler 接口

其实 go 的 http 处理最终也是交给 `Handler` 去处理,这里可以通过实现这个接口来扩展自定义 http 框架(因为 go 也是这么做的)

3. gin 的 `HandlerFunc` 中间函数,这个地方可以扩展的东西太多了,我们可以把各种和业务挂钩的 hook 函数全部添加到 `handlerChain` 里

有点 `AOP` 的感觉嗷

### 结合 fx 框架实现自定义启动和结束行为

`fx` 是 uber 开源的一款依赖注入框架,可以通过 `fx` 启动一个 `app` 服务,利用 `fx` 的 `hook` 函数,可以轻松地实现类似 `Spring` 的 `AOP` 能力

为 `fx` 容器注册启动和停止时的 `hook` 函数

首先根据前面总结的内容,启动 gin 服务,无非就是创建了一个 `http.Server` 容器而已

只不过 gin 通过实现 `Handler` 接口,实际上启动的是 gin 自定义的 `Engine` 对象而已

所以为了对 gin 做满足业务的二次开发,我们也通过同样的接口,定义我们自己的 `Server` 对象,并且让它配合 `fx` 框架完成启动和停止的自定义操作

```go
package main

// 首先定义我们的自定义 `Server` 对象
type GinxServer struct {
	server *http.Server
	proxy  *ProxyGinEngine
}

type ProxyGinEngine struct {
	engine *gin.Engine
}

func (s *GinxServer) OnStart() {
    // 注意
    // 注意
    // 注意
	go func() {
		fmt.Println("ginx server starting....")
        // 这里才是启动 gin 容器的关键代码
		err := s.server.ListenAndServe()
		if err != nil {
			fmt.Printf("ginx listen and serve error: %s", err.Error())
		}
	}()
}

func (s *GinxServer) OnStop() {
	fmt.Println("ginx server stoping....")
}

func NewServer(addr string) *GinxServer {
	// 使用我们自定义的 Server
	s := &GinxServer{
		server: &http.Server{
			Addr: addr,
		},
		// 底层仍然是 gin 的引擎,不过对它做了一些扩展
		proxy: &ProxyGinEngine{engine: gin.New()},
	}

    // 注册我们的路由函数
	s.Register("GET", "/ping", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"msg": "pong",
		})
	})
	s.Register("GET", "/hello", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"msg": "hello world",
		})
	})
	return s
}

func main() {
    ctx := context.Background()
	app := fx.New(
		fx.Provide(ginx.NewServer),
		fx.Invoke(func(lc fx.Lifecycle, server *ginx.GinxServer) {
			lc.Append(fx.Hook{
				OnStart: func(ctx context.Context) error {
					server.OnStart()
					return nil
				},
				OnStop: func(ctx context.Context) error {
					server.OnStop()
					return nil
				},
			})
		}),
	)
    // 注意通过 ide 启动 main 函数后,再通过 ide 结束是无法出发 OnStop 事件的
    // 因为 ide 的结束是直接杀死进程,而不是通过信号量的方式结束进程
    // 调用 fx.Stop() 时,会发送信号量来结束进程,这样 OnStop() 函数才能触发
	go func() {
		time.Sleep(2 * time.Second)
		app.Stop(ctx)
	}()
	app.Run()
}
```

注意这里有个大坑,在 `OnStart()` 里一定要用协程去启动这个 gin 的监听函数

对于 fx 框架来说,所有 `hook` 函数都必须正常启动,fx 应用才算最终启动

如果这里没有用协程启动,那么 `OnStart()` 函数将会一直阻塞下去(因为 `ListenAndServe()` 阻塞),相当于这个 `OnStart()` 函数就一直没有返回

而 fx 框架在执行 hook 函数时,发现某个 hook 超时没有正确返回,就会认为启动 fx.App 失败,从而发出停止的信号量

![img.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7nn3ybz8gj30t803rjvk.jpg)

### 添加自定义 HandlerFunc() 函数

假如我们要对每个 http 请求统计其请求耗时,以及添加相关的埋点信息,应当如何扩展 gin 的 `HandlerFunc` 函数

首先可以知道的是,在 `ServeHTTP()` 最终会调用到 gin 重写的 `ServerHTTP()` 方法

里面 gin 通过遍历路由树,找到请求对应的路由节点,然后顺序执行节点的 `[]HanderFunc()` 函数列表

其中有个很关键的函数 `c.Next()` 就是用来遍历整个 `[]HandlerFunc()` 中间件函数的

所以我们的两个目标: 1. 新增请求埋点 2. 统计接口耗时  就需要在这个 `c.Next()` 函数上动文章

简单回顾下这个 `c.Next()` 函数

```go
package gin

func (c *Context) Next() {
	c.index++
	for c.index < int8(len(c.handlers)) {
		c.handlers[c.index](c)
		c.index++
	}
}
```

可以看到,在 gin 的自定义上下文 `Context` 通过 `index` 字段记录了当前执行的中间件函数的位置,一旦所有函数执行完成之后,就会返回

首先记录请求埋点肯定是先于业务函数的,所以在初始化 gin 容器的时候,在其他业务 `api` 注册之前,就应该先行注册埋点函数

改造 `NewServer()` 函数

```go
package ginx

func NewServer(addr string) *GinxServer {
	// 省略前面的初始化

    // 首先添加埋点函数的注册
    // 这样每次遍历 []HandlerFunc 的时候第一个都是 这个埋点函数
    s.proxy.engine.Use(func (c *gin.Context) {
        // req := c.Request
		// header := c.Request.Header
		// body := c.Request.Body
		// do something with req or header or body
		fmt.Printf("do something with request")
    })

    // 接着注册我们的统计耗时的函数
	s.proxy.engine.Use(func(c *gin.Context) {
		start := time.Now() // 计算当前时间
		// 注意这里调用 c.Next() 很关键
		c.Next()
		cost := time.Since(start) // 计算耗时
		fmt.Printf("cost time is %s", cost.String())
	})

    // 省略业务路由的注册
	return s
}
```

这里着重关注统计耗时的函数,在其内部再次调用了 `c.Next()` 函数

来看看调用链 :

![1.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7nmkujkcsj31240kegus.jpg)

看看最终效果

![2.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7nqwvrx9zj30te0cowl4.jpg)

可以看到埋点函数和统计耗时函数都能正常工作

**发现个很奇怪的地方,没来得及分析**

就是每次 gin 服务启动之后,第一次请求耗费的时间远长于后面的请求

有些情况下甚至第一次请求耗时是后面的几十倍

这个还没找到原因,如果有谁知道,可以通过博客的 **About** 页面取得联系方式告知我一下