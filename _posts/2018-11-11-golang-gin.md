---
layout: post
title: 'golang web框架gin简析'
tags: golang
---
gin是一个基于golang`net/http`的web框架，其中封装了一些很多方便的方法供我们调用，用起来还是挺舒服的

## 特色
包括但不限于：
* custome [httprouter](https://github.com/julienschmidt/httprouter)
* 可自定义json包
* 中间件支持
* 数据bind
* 数据校验
* RouterGroup

## 主要流程

```go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run() // listen and serve on 0.0.0.0:8080
}
```
这样看有点迷糊，我们把上面代码分成三段来看吧

### gin.Default
```go
type Engine struct {
	RouterGroup
    ...
}

func Default() *Engine {
    engine := New()
        engine := &Engine{
            RouterGroup: RouterGroup{
                Handlers: nil,
                basePath: "/",
                root:     true,
            },
        trees: make(methodTrees, 0, 9),
        ...
        }
        engine.RouterGroup.engine = engine
	return engine
}
```
这里只是创建了一个engine对象，返回`*Engine`，这里可以看到`Engine`有个`RouterGroup`匿名成员，说明`Engine`可以调用`RouterGroup`拥有的方法，这个将在下文体现

### r.GET
```go
func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
    return group.handle("GET", relativePath, handlers)
        absolutePath := group.calculateAbsolutePath(relativePath)
        handlers = group.combineHandlers(handlers)
        group.engine.addRoute(httpMethod, absolutePath, handlers)
        	root := engine.trees.get(method)
            if root == nil {
                root = new(node)
                engine.trees = append(engine.trees, methodTree{method: method, root: root})
            }
            root.addRoute(path, handlers)

        return group.returnObj()

```
这里调用`group.engine.addRoute(httpMethod, absolutePath, handlers)`注册处理函数，以便以后通过`engine.trees`来找到相应的处理函数。不过这里还是有几个个疑点：
1. GET方法的第二个参数为什么是一个slice
2. `handlers = group.combineHandlers(handlers)`是什么作用

这两个疑点放在说明中间件再来分析

### r.Run
```go
func (engine *Engine) Run(addr ...string) (err error) {
    address := resolveAddress(addr)
    err = http.ListenAndServe(address, engine)
```
这里主要将创建出来的engine指针传递给了`http.ListenAndServe`，从上篇文章中我们知道`engine`需要有个`ServeHTTP`方法：
```go
type Context struct {
	writermem responseWriter
	Request   *http.Request
    ...
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    c := engine.pool.Get().(*Context)
    c.Request = req
    engine.handleHTTPRequest(c)
    engine.pool.Put(c)
```
这里的`engine.pool`是一个`sync.Pool`对象，它的作用是防止`Context`对象重复创建销毁，以减轻gc压力

看来`engine.handleHTTPRequest(c)`会调用路由处理函数
```go
func (engine *Engine) handleHTTPRequest(c *Context) {
	httpMethod := c.Request.Method
	path := c.Request.URL.Path
	t := engine.trees
	for i, tl := 0, len(t); i < tl; i++ {
		if t[i].method != httpMethod {
			continue
		}
		root := t[i].root
		// Find route in tree
		handlers, params, tsr := root.getValue(path, c.Params, unescape)
		if handlers != nil {
			c.handlers = handlers
			c.Params = params
			c.Next()
			c.writermem.WriteHeaderNow()
			return
		}
        ...
        break
    }
    serveError(c, http.StatusNotFound, default404Body)
```
这里流程简述如下：
* 根据http method找到之前注册的handlers路由处理函数
* 调用`c.Next()`执行路由处理函数

`c.Next()`代码如下：
```go
func (c *Context) Next() {
	c.index++
	for s := int8(len(c.handlers)); c.index < s; c.index++ {
		c.handlers[c.index](c)
	}
```
这里的代码我们暂时只需要知道它调用了路由处理函数就够了，所以这里将原声的http路由函数的签名`func(w http.ResponseWriter, req *http.Request)`变成了`func(c *Context)`

接下来我们看看中间件的实现就知道这里为什么这么写

## 中间件
我们来看看最简单的`Logger`中间件是怎么实现的

调用`gin.Default()`会默认使用`Logger`和`Recover`中间件，我们这里只看`Logger`中间件
```go
func Default() *Engine {
    engine.Use(Logger(), Recovery())
    engine.RouterGroup.Use(middleware...)
        func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
            group.Handlers = append(group.Handlers, middleware...)
```
可以看到这里将`Use`的参数放到了`group.Handlers`slice中

在看看之前分析`r.GET()`的疑点之一：`handlers = group.combineHandlers(handlers)`是什么作用？
```go
func (group *RouterGroup) combineHandlers(handlers HandlersChain) HandlersChain {
	mergedHandlers := make(HandlersChain, finalSize)
	copy(mergedHandlers, group.Handlers)
	copy(mergedHandlers[len(group.Handlers):], handlers)
	return mergedHandlers
```
这里的`mergedHandlers`是中间件加路由处理函数的的三个元素的slice，也就是说调用`gin.Default()`，再调用`r.GET()`后，`mergedHandlers`的成员为：
1. `Logger()`
2. `Recovery()`
3. `r.GET`的第二个参数

`r.Run()`最后会找到这里的`mergedHandlers`，并赋值给`Context`
```go
c.handlers = handlers
c.Next()
```
所以处理函数调用的入口还是在`c.Next()`：
```go
func (c *Context) Next() {
	c.index++
	for s := int8(len(c.handlers)); c.index < s; c.index++ {
		c.handlers[c.index](c)
	}
```
每调用一次c.Next()就会调用处理函数下一个元素

再看看`Logger`中间件的实现
```go
func Logger() HandlerFunc {
    return LoggerWithWriter(DefaultWriter)
        return func(c *Context) {
            start := time.Now()
            path := c.Request.URL.Path
            c.Next()
            if _, ok := skip[path]; !ok {
                // Stop timer
                end := time.Now()
                fmt.Fprintf(out, "[GIN] %v |%s %3d %s| %13v | %15s |%s %-7s %s %s\n%s",
                    end.Format("2006/01/02 - 15:04:05"),
                    statusColor, statusCode, resetColor,
                    latency,
                    clientIP,
                    methodColor, method, resetColor,
                    path,
                    comment,
                )
        }
```
这里`Logger()`返回一个函数，每个请求过来都是先执行这个路由函数，再执行下一个，也就是先执行多个中间件函数，在如果通过了中间件的执行，最后才执行路由处理函数，返回response给client

## 跨域中间件的实现
在说明跨域中间件之前，先说明跨域的一些背景：
* 跨域请求之前，浏览器会先发起一个`OPTIONS`请求，用于向服务器询问是否允许跨域
* 对于简单跨域请求，比如GET，简单的表单POST请求，跨域请求不会提前OPTIONS
* 对于复杂跨域请求，比如非表单POST请求,PUT,DELETE,CONNECT,TRACE,PATCH等，会先请求OPTOINS

一个gin的跨域中间件为：[gin-cors](https://github.com/gin-contrib/cors)，一个demo如下：
```go
router.Use(cors.New(cors.Config{
    AllowOrigins:     []string{"https://foo.com"},
    AllowMethods:     []string{"PUT", "PATCH"},
    AllowHeaders:     []string{"Origin"},
    ExposeHeaders:    []string{"Content-Length"},
    AllowCredentials: true,
    AllowOriginFunc: func(origin string) bool {
        return origin == "https://github.com"
    },
    MaxAge: 12 * time.Hour,
}))
```

`cors.New`会返回一个中间件处理函数
```go
func New(config Config) gin.HandlerFunc {
	cors := newCors(config)
	return func(c *gin.Context) {
		cors.applyCors(c)
	}
}
```
`cors.applyCors`的实现：
```go
func (cors *cors) applyCors(c *gin.Context) {
	origin := c.Request.Header.Get("Origin")
	if len(origin) == 0 {
		// request is not a CORS request
		return
	}
	if c.Request.Method == "OPTIONS" {
		cors.handlePreflight(c)
		defer c.AbortWithStatus(http.StatusNoContent) // Using 204 is better than 200 when the request status is OPTIONS
	} else {
        cors.handleNormal(c)
            func (cors *cors) handleNormal(c *gin.Context) {
                header := c.Writer.Header()
                for key, value := range cors.normalHeaders {
                    header[key] = value
                }
	}
```
从之前的分析我们可以看到，查找路由处理函数根据http method做映射的，而我们从来自然也无需调用`r.OPTIONS`之类的方法，所以OPTIONS的路由处理函数入口：
```go
func (engine *Engine) handleHTTPRequest(c *Context) {
    httpMethod := c.Request.Method
    path := c.Request.URL.Path
    for i, tl := 0, len(t); i < tl; i++ {
        ...
    }
    serveError(c, http.StatusNotFound, default404Body)
        c.Next()
        if c.writermem.Status() == code {
            c.writermem.Header()["Content-Type"] = mimePlain
            c.Writer.Write(defaultMessage)
            return
        }
        c.writermem.WriteHeaderNow()
```
这里通过`serveError`来处理没有注册过的http method路由处理函数

## RouterGroup
到目前为止，我们还没有说过`RouterGroup`，路径连接问题倒是很好理解，但是为什么`RouterGroup`也能使用`gin.Default`的中间件呢？我们来看看`RouterGroup`的创建过程就知道了，一个`RouterGroup`的demo
```go
v1 := router.Group("/v1")
{
    v1.POST("/login", loginEndpoint)
}
```
分析下`router.Group("/v1")`：
```go
func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup {
	return &RouterGroup{
		Handlers: group.combineHandlers(handlers),
		basePath: group.calculateAbsolutePath(relativePath),
		engine:   group.engine,
	}
```
这里`Group`的`receiver`是`gin.Default()`返回的指针，所以这里新创建的`RouterGroup`的`handlers`成员会是`gin.Default()`的`Handlers`+`Group`的第二个参数(当然这里没有)，所以这里就把`gin.Default()`或根`RouterGroup`的中间件copy过来了，自然Group Router也能使用`gin.Default`的中间件啦

## 怎么换json序列化包
gin默认使用的是golang提供的json标准包，但是也支持将其换成`jsoniter`(主要性能更好)

使用如下命令行编译二进制文件即可
```bash
go build -tags=jsoniter .
```

## 话外
这里只是说了下核心流程，`Context`下封装了很多方便我们处理请求与返回响应的方法，特别是数据绑定、ORM、数据校验都是特色，这些都值得一观