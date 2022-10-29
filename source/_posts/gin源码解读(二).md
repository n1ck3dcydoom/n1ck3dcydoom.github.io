---
title: gin 源码解读 (二)
date: 2022-10-29 15:44:44
index_img: /img/gin.png
categories:
- gin
tags:
- 源码
---

简单回顾: gin 从 `Run()` 函数启动容器之后

注册路由,添加路由中间函数的调用链

到使用 go 底层的 `net` 库绑定地址和端口,阻塞监听端口上的请求

到最后通过中间函数的调用链完成请求的处理

这里来看看 gin 容器是如何初始化的,都分别作了什么

### gin 容器的初始化

gin 提供了两个最常用的初始化方法

1. gin.New()
2. gin.Default()

其中 `Default()` 就是在 `New()` 的层面上,多添加了两个中间函数而已

![img.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7m78mblrej30ct049dhi.jpg)

对于 `New()` 函数来说,在这里面初始化设置了 `engine` 对象的各种属性配置

比较关键的几个列出来单独说明下:

```go
package gin

func New() *Engine {
	engine := &Engine{
		RouteGroup: RouteGroup{ // 初始化路由组对象
			Handlers: nil,
			basePath: "/",
			root:     true, // 设置为根节点
		},
		// {省略} 
		trees: make(methodTrees, 0, 9) // 为 9 种 http 方法初始化路由树
	}
	// 初始化 Context 池,减少上下文的频繁创建,提高内存复用率
	engine.RouterGroup.engine = engine
	engine.pool.New = func() interface{} {
		return engine.allocateContext()
	}
}
```

### 路由组的初始化

在有了 gin 容器之后,接下来需要做的就是为我们的 api 分类并设置路由组

就是把具有类似路由 path,隶属于同一个领域的 api 聚合到一个路由组里面,方便快速的查找路由

使用函数 `engine.Group()` 可以快速的聚合一组 api 路由

![img_1.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7m794m97bj30o305dn26.jpg)

继续跟进 `engine.combineHandlers()` 函数一探究竟

![img_2.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7m799u5raj30mw06eaf6.jpg)

之前接收到请求后,有个很关键的 `Next()` 函数,其遍历的所有 `handler`,都来自于路由组绑定的 `handlers`

至于计算绝对路径的函数很简单,就是把上一个绝对路径和当前路由的 path 拼接到一块

### 具体某个路由如何注册

以 `engine.GET()` 函数为例,说一下一个具体的路由是如何注册到对应方法的路由组里去的

![img_3.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7m79eu2cwj30kt02z76o.jpg)

调用 `engine.handler()` 函数完成实际的注册

![img_4.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7m79jhejxj30od047q6k.jpg)

其中 `calculateAbsolutePath()` 和 `combineHandlers()` 不在赘述

着重关注 `addRoute()` 是如何在路由树上添加路由

```go
package gin

func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
	// 省略前置校验

	// 检查当前 method 路由树的 root 节点是否存在,没有的话就创建为 root 节点
	root := engine.trees.get(method)
	if root == nil {
		root = new(node)
		root.fullPath = "/"
		engine.trees = append(engine.trees, methodTree{method: method, root: root})
	}
	// 重中之重,对路由树(前缀树)的处理
	root.addRoute(path, handlers)

	// 省略不重要的
}
```

### addRoute() 函数前缀树应用一探究竟

有两个关键分支:

1. root 为空创建前缀树根节点
2. root 非空,在前缀树上插入节点

#### root 为空创建空节点

以这组 RESTful api 为例说明:

```
r.GET("/cat/:id/children")
r.GET("/cat/play")
r.GET("/dog")
```

第一次添加 `method` 的路由前缀树时,创建了空的 root 节点,在判断当前节点 `path` 长度为 0,并且没有 `children` 子节点时,就需要初始化 root 节点

直接调用 `insertChild()` 函数将当前路由插入到 root 节点当中

首先判断当前 path 是否包含通配符,例如 `/cat/:id`

首次创建 root 前缀树时,不应该直接使用 `/cat/:id` 作为 `full path` 而是需要把通配符 `:id` 单独切割出来,正确的 `full path` 应该是 `/cat/`

函数 `findWildcard()` 返回通配符 `:` 或者 `*` ,以及通配符规则是否符合 `valid` 和通配符所处的下标

```go
package gin

func (n *node) insertChild(path string, fullPath string, handlers HandlersChain) {
	for {
		// 找到通配符的位置
		wildcard, i, valid := findWildcard(path)
		// 省略校验逻辑
		if wildcard[0] == ':' { // param
			// 如果有通配符,则把通配符前面的 path 当做 root 节点的 path,而不是带有通配符的 path
			if i > 0 {
				// Insert prefix before the current wildcard
				n.path = path[:i]
				path = path[i:]
			}
			// 为当前 root 节点创建带有通配符的子节点
			child := &node{
				// 通配符节点类型为 param,表示在路由树上有个通过参数来区分的路由节点
				nType:    param,
				path:     wildcard,
				fullPath: fullPath,
			}
			// 将通配符路由当做子路由添加到当前路由节点 '/cat/' 上
			n.addChild(child)
			// 设置当前路由节点 '/cat/' 带有通配符节点
			n.wildChild = true
			// 更新路由树的指针,继续对通配符节点查找后续的路由树,例如 '/cat/:id/children`
			// 表示列出 :id 的 cat 的所有小猫咪 :)
			// 所以这里需要更新指针指向子节点,继续递归解析
			n = child
			// 注意:这里设置的是子节点的 priority 优先级属性
			// 可以看到在对 root 节点每添加一个子路由,都会让全链路的节点 priority 值 +1
			// 越靠近根节点的 prioriy 值越大
			n.priority++

			// 如果当前路径不是以通配符结尾, 例如 `/cat/:id/children` 
			// 那么会继续解析后续的 `/children` 为更深层次的路由子节点
			if len(wildcard) < len(path) {
				path = path[len(wildcard):]
				// 新添加的节点,其 prioriy 值从 1 开始计算,每过一个父节点都比下面的孩子节点大 1
				child := &node{
					priority: 1,
					fullPath: fullPath,
				}
				n.addChild(child)
				n = child
				continue
			}
			// 添加中间函数
			n.handlers = handlers
			return
		}
		// 省略
	}
}

```

对于第一个带有通配符的路由 `/cat/:id`

在切割 `:id` 之后,得到路由为 `:id` 的通配符子节点

![img_5.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7m79o3oj4j30bp050gnm.jpg)

#### root 节点非空,往路由前缀树继续添加子路由

回到 `addRoute()` 函数,继续看 root 非空的情况是如何处理的

首先一上来就是一个 100 行的无限循环,代码有点长,仔细解读

```go
package gin

func addRoute() {
	for {
		// 从 root(n) 节点开始查找最长前缀的位置索引 i
		// 这个函数很简单,双指针遍历比较找到第一个不相等的位置
		i := longestCommonPrefix(path, n.path)

		// 前面说过 gin 使用的紧凑型的前缀树,而非标准的字典前缀树,节省节点内存分配
		// 如果当前节点是插入节点的子节点,则需要调整父子节点关系
		// 例如已经插入 即 n = /cat/play/ball
		// 现在插入 /cat/play
		// 最后 n 变成 /cat/play, n.children = /ball
		// 同理,如果继续添加路由 /cat
		// n = /cat, n.c = /cat/play, n.c.c = /cat/play/ball
		if i < len(n.path) {
			child := node{
				path:      n.path[i:],
				wildChild: n.wildChild,
				indices:   n.indices,
				children:  n.children,
				handlers:  n.handlers,
				priority:  n.priority - 1,
				fullPath:  n.fullPath,
			}

			n.children = []*node{&child}
			// []byte for proper unicode char conversion, see #65
			n.indices = bytesconv.BytesToString([]byte{n.path[i]})
			n.path = path[:i]
			n.handlers = nil
			n.wildChild = false
			n.fullPath = fullPath[:parentFullPathIndex+i]
		}
		// 如果在先后顺序上符合父子关系,可以直接在节点 n 上创建对应的子节点
		if i < len(path) {
			path = path[i:]
			c := path[0]

			// 处理参数节点后带有 '/' 的情况
			if n.nType == param && c == '/' && len(n.children) == 1 {
				parentFullPathIndex += len(n.path)
				n = n.children[0]
				n.priority++
				continue walk
			}

			// 处理带有公共前缀的节点
			// 解释一下 indices 的作用,这个字符串保存当前节点下所有孩子节点最长公共前缀之后出现的第一个字符
			// 例如 /cat/run
			// 添加 /cat/run_with_me
			// run 和 run_with_me 有着公共前缀 run
			// 这样 /cat/run_with_me 就会被继续拆分为子节点 _with_me
			// 父节点为 /cat/run => _with_me

			// 在举例,若有 /person/eat 和 /person/laugh
			// 此时 /person 节点有 indince = le
			// 添加新路由 /person/lying,此时发现 lying 和 l 有公共前缀 l
			// 就会将 /laugh 节点拆分为 /l 作为父节点,包含两个子节点 augh 和 ying
			// 最终 /person/ => eat 和 l
			// /l => augh 和 ying ,且 a 的 indince = ay,eat 没有孩子,所以没有 indinces
			for i, max := 0, len(n.indices); i < max; i++ {
				if c == n.indices[i] {
					parentFullPathIndex += len(n.path)
					i = n.incrementChildPrio(i)
					n = n.children[i]
					continue walk
				}
			}

			// 如果插入节点和当前节点不在同一个父节点上,则直接插入到前缀树里
			if c != ':' && c != '*' && n.nType != catchAll {
				// []byte for proper unicode char conversion, see #65
				n.indices += bytesconv.BytesToString([]byte{c})
				child := &node{
					fullPath: fullPath,
				}
				n.addChild(child)
				n.incrementChildPrio(len(n.indices) - 1)
				n = child
			} else if n.wildChild {
				// 如果当前节点是参数节点,往参数节点后面添加子节点
				n = n.children[len(n.children)-1]
				n.priority++

				// Check if the wildcard matches
				if len(path) >= len(n.path) && n.path == path[:len(n.path)] &&
					// Adding a child to a catchAll is not possible
					n.nType != catchAll &&
					// Check for longer wildcard, e.g. :name and :names
					(len(n.path) >= len(path) || path[len(n.path)] == '/') {
					continue walk
				}

				// Wildcard conflict
				pathSeg := path
				if n.nType != catchAll {
					pathSeg = strings.SplitN(pathSeg, "/", 2)[0]
				}
				prefix := fullPath[:strings.Index(fullPath, pathSeg)] + n.path
				panic("'" + pathSeg +
					"' in new path '" + fullPath +
					"' conflicts with existing wildcard '" + n.path +
					"' in existing prefix '" + prefix +
					"'")
			}

			n.insertChild(path, fullPath, handlers)
			return
		}

		// Otherwise add handle to current node
		if n.handlers != nil {
			panic("handlers are already registered for path '" + fullPath + "'")
		}
		n.handlers = handlers
		n.fullPath = fullPath
		return
	}
}
```

如果后插入的节点在 **关系上** 是先插入节点的父节点,则需要调整父子节点的关系

原先的父节点 `/cat/play/ball` ,后插入的节点 `/cat/play`

从关系上来说,后者应该是前者的前驱 `/cat/play` ; 前者截断后变为子节点 `/ball`

![img_6.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7m79vblj0j30a4055jsx.jpg)

![img_7.png](https://tva1.sinaimg.cn/large/008vK57jgy1h7m7aa05qlj30bb0a8q6w.jpg)

### 总结

1. gin 使用紧凑前缀树实现路由树,减少树节点的个数,提高查找效率
2. gin 对通配符 :xxx 和 *xxx 两种类型的参数有些特殊限制,尤其小心在 *xxx/other 这种路由,会导致 /other 节点失效,具体原因是因为 *xxx 节点被设置为 cathAll 类型,
   其后面的路由不再匹配,全部由 *xxx 提供服务
3. 其中 priority 字段的设计,是为了让出现次数更多的路由前缀尽可能早的匹配到请求上,提升前缀树检索的性能
4. 其中 indinces 的设计,是为了将路由子节点尽可能多的拆分出公共前缀,这样做的目的也是为了减少路由树中路由节点的个数,提升检索性能

总之,还是有很多细节没有列举完成,包括对通配符节点的特殊处理和判断,都没有一一解释代码了
