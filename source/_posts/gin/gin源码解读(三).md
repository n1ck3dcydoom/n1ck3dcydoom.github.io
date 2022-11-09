---
title: gin 源码解读 (三)
date: 2022-10-30 17:21:50
index_img: /img/gin.png
categories:
  - gin
tags:
  - gin
---

## http 连接的建立和监听

前面用了介绍了 `engine` 容器的初始化, 处理 `http` 请求的流程, `Router` 路由树的生成和注册

接下来将了解下,一个 http 连接究竟是如何建立的,以及如何将流程扭转到 gin 去实际处理一个请求

### http.ListenAndServe() 函数

回到 `gin.Run()` 函数实现,里面有个 `http.ListenAndServe(address, engine)` 函数

通过入参和签名,很明显的告诉我们这个函数将监听在 `address` 的连接, 第二个参数 `engine` 猜测最终会把请求交给 `engine` 容器去处理

![img.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7nfglt6d4j30is06yado.jpg)

首先创建一个 `Server` 对象,将需要监听的地址,和处理器 (和 gin 的中间函数 `HandlerFunc` 区分下)

其实这里的处理器,就是我们之前初始化的 `engine` 容器,不过 `engine` 实现了 `Handler` 接口而已

这里可以充分体现出 gin 在设计上遵循 **依赖接口而不是依赖实现** 的原则

![img_1.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7nfgu7g3rj30i1088q68.jpg)

这里的 `Serve` 对象感觉和 java 的 `Socket` 套接字很类似,监听并 **阻塞** 等待

既然是类似套接字,那么自然就能关闭,所以检查当前的套接字是否关闭,已经关闭了就不能再监听地址接受请求了

后面就是创建 `tcp` 链接,以及处理事件

#### Listen() 函数

跟进 `net.Listen()` 函数和 `lc.Listen()` 函数

```go
package gin

func (lc *ListenConfig) Listen(ctx context.Context, network, address string) (Listener, error) {
	DefaultResolver.resolveAddrList(ctx, "listen", network, address, nil)
}
```

`resolveAddrList()` 函数负责解析 `address` 的监听类型, 通过后面的逻辑可知,大致能够分为两类

1. TCP 类型(通过 ip 地址加 port 端口,UDP 也类似)
2. UNIX 套接字类型???

里面都是 go 原生的网络库实现,大致逻辑如下:

1. 解析 `ip` 地址,解析是 `ipv4` 还是 `ipv6` 的类型
2. 尝试解析 `ip` 类型,如果不是一个合法的 `ip` 地址,就试着当做 `DNS` 地址解析

接下来是为解析后的 `addr` 地址创建对应的的监听者对象 `sysListener` 这没什么好说的

```go
package gin

func (lc *ListenConfig) Listen(ctx context.Context, network, address string) (Listener, error) {
	sl := &sysListener{
		ListenConfig: *lc,
		network:      network,
		address:      address,
	}
```

通过这个 `sysListener` 对象来完成实际的监听行为

```go
package gin

func (lc *ListenConfig) Listen(ctx context.Context, network, address string) (Listener, error) {
	sl.listenTCP(ctx, la)
}
```

![img_2.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7nfgz7a1zj30ff04rgo3.jpg)

这里面都涉及到 `unix` 底层的套接字编程,得到网络文件描述符 `fd` 完成后续操作,哎我也看不懂了

有空还要去拜读下 `unix` 的 `socket` 套接字编程,再回头看看这里应该会清晰很多

最后把创建的套接字对象 `fd` 注入到 `sysListener` 里,这样这个监听者就相当于已经就绪了,可以进行监听操作

返回到 `http.ListenerAndServe(ln)` 将 `listen()` 函数创建得到的监听者 `ln` 当做参数传入

#### Serve() 函数

```go
package gin

func (srv *Server) Serve(l net.Listener) error {
	origListener := l
	l = &onceCloseListener{Listener: l}
	defer l.Close()
}
```

通过 `OnceCloseListener` 对象包裹原始的 `Listener` 对象

从字面意思上来说,就是只允许调用一次 `Close()` 避免重复关闭一个已经关闭了的 `Listener`

```go
package gin

func (srv *Server) Serve(l net.Listener) error {
	if !srv.trackListener(&l, true) {
		return ErrServerClosed
	}
	defer srv.trackListener(&l, false)
}
```

将传入的监听者 `ln` 注入到 `Serve` 对象里, defer 保证了回收 `Serve` 的时候会把 `ln` 移除掉

```go
package gin

func (srv *Server) Serve(l net.Listener) error {
	baseCtx := context.Background()
	// ...
	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
}
```

创建一个空的 `Context` 对象,将 `Serve` 对象注入到上下文里

#### 阻塞监听

**以上完成了所有准备,接下来开启一个死循环阻塞监听请求**

```go
package gin

func (srv *Server) Serve(l net.Listener) error {
	for {
		// 调用监听者的 Accept() 函数,这是个同步阻塞函数
		// 2022-10-30 本人无能,套接字编程,网络文件描述符相关的操作,实在是看不懂啊
		rw, err := l.Accept()

		// 当 l 监听到时间产生后, Accept() 函数返回,开始后面处理请求
		// 可以通过启动 gin 的时候注册一个简单的 /ping 心跳路由
		// curl 请求一下这个 /ping 路由即可

		// 跳过监听得到的异常处理

		// 这个 ConnContext 成员变量可以在初始化 Serve 对象的时候注入进去
		// 本质是一个 func(ctx context.Context, c net.Conn) context.Context 函数变量
		// 起作用是在得到一个新的 c 连接的时候,返回一个继承自 ctx 的新的上下文对象
		// 具体怎么继承,以及要对这个新的上下文对象做什么处理,则有函数自行决定

		// 这里要么使用原始的父 ctx 对象,要么通过 cc 函数得到个继承自 ctx 的新上下文对象
		connCtx := ctx
		if cc := srv.ConnContext; cc != nil {
			connCtx = cc(connCtx, rw)
			if connCtx == nil {
				panic("ConnContext returned nil")
			}
		}
		tempDelay = 0
		// 将 http 裸的连接包成 conn 对象
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew, runHooks) // before Serve can return

		// 启动一个协程去处理接收到的连接 conn
		go c.serve(connCtx)
	}
}
```

`Unix` 网络套接字编程是很复杂的,这里因为之前没有充分准备,导致很多 `Socket` 底层通过 `Unix` 提供的文件描述符进行网络操作的代码看不懂

后面补充了 `Socket` 相关知识后,一定要回来再看看这里

整个流程总结如下:

1. Accept() 函数阻塞监听,直到请求过来返回 net.conn 裸的 http 连接
2. 如果 Accept() 返回有异常,则处理异常
3. 如果 Serve 初始化了 ConnContext 函数成员变量,这个函数会继承全局的 ctx 上下文得到一个新的上下文对象
4. 将裸的 http.conn 包装为 serve 包下面的 conn 结构,新建一个协程,使用第 3 步得到上下文去处理请求
5. 主协程通过死循环仍然继续阻塞监听 Accept() 函数,直到下一个请求进来 go to 1

#### 实际的 serve(connCtx) 函数

接下来进一步解析这个 `serve()` 函数如何处理请求的

`serve()` 函数很长,挑重点解析

```go
package gin

func (c *conn) serve(ctx context.Context) {
	// 上下文的设置
	// ...

	// defer 定义了发生异常后的处理流程
	// ...

	// 检查和处理 tls 协议的链接,即 https 请求
	// ..

	// 非 tls 协议, http 请求的处理
	// 创建一个带有
	ctx, cancelCtx := context.WithCancel(ctx)
	c.cancelCtx = cancelCtx
	defer cancelCtx()

	// 初始化 buffer reader 和 buffer writer
	// 为读取连接和向连接发送内容做好准备
	c.r = &connReader{conn: c}
	c.bufr = newBufioReader(c.r)
	c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)

	for {
		// 解析请求
		w, err := c.readRequest(ctx)
	}
}
```

`c.readRequest()` 函数将会实际处理 `conn` 对象的请求

```go
package gin

func (c *conn) readRequest(ctx context.Context) (w *response, err error) {
	// 设置超时时间,最长读取时间,缓冲最大读取长度,等等参数
	// 兼容一些老的客户端在发送 post 请求会后多带上了 /r/n 等换行符,从缓冲区读取的时候去掉这些字节
	// ...

	// 从缓冲区里读取请求内容
	req, err := readRequest(c.bufr)
}
```

`readRequest(c.bufr)` 函数就是按照 `http 1.0` 规范解析请求报文

![img_3.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7nfh60d1sj30ne03pwi9.jpg)

首先解析报文行,得到请求方法,请求的协议和版本

紧跟着解析报文头,得到 `Header` 里面的值,处理 `Header` 里面约束的各种参数

校验一些协议和版本是否兼容,校验 `Header` 参数是否合法

最后通过传入的 `reader` 构造一个 `bufferWriter` 当做 `response` 的操作对象

#### 通过 ServeHTTP 函数扭转处理流程

在前面通过 `readRequest()` 解析请求后,创建对应的 `http.response`

**注意:重点来了,这个时候 go 原生的 net 网络包已经帮忙把一个 http 请求全部解析好; 这个时候就需要通过对外暴露的 ServeHTTP 接口把控制流程交给其实现类去处理; 否则请求将会由 go net 库自行处理掉**

![img_4.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7nfhd792kj30iy0b1dld.jpg)

跟进这个 `ServeHTTP()` 方法,在这里面实际上调用的是 `server.Handerl.ServeHTTP()` 方法

此时,终于把控制流程交给了 gin 去处理

回到 gin 框架里面,如果对 gin 的流程有些忘记了的话,可以看看第一张里面关于 `gin 如何通过 Handler 接口实现接收和响应请求`
的部分

![img_5.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7kefvra9xj30jf06w42g.jpg)
