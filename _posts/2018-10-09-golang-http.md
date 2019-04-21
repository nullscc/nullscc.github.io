---
layout: post
title: 'golang http标准库简析'
tags: golang
---
golang的http库不需要多少改动就直接可以上生产环境，毕竟有goroutine这种东西存在，不用自己处理什么epoll多路IO复用的东东。但是它本身自带的router效率还是比较低的，好在它可以自定义router，所以催生出了一些比较高效的开源router，比如[httprouter](https://github.com/julienschmidt/httprouter)。基于这个原因，很多golang的web框架都是基于`net/http`+自定义router来封装，比如[gin](https://github.com/gin-gonic/gin)。

这里我们简单说说`net/http`处理流程和对golang本身性质的利用，不过多追求细节，毕竟从实现上还是类似epoll的那一套

这里我们主要分析server端的实现，略过client的实现

## 主要处理流程
首先我们看看一个简单的http server是什么样的？
```go
package main

import (
	"io"
	"log"
	"net/http"
)

func main() {
	helloHandler := func(w http.ResponseWriter, req *http.Request) {
		io.WriteString(w, "Hello, world!\n")
	}

	http.HandleFunc("/hello", helloHandler)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

上述代码的主要点在于：
* 调用`http.HandleFunc`注册一个处理函数的回调，当访问`/hello`调用`helloHandler`来处理这个请求
* 对于`helloHandler`来说，第一个参数可以视为网络IO的文件，向它写入内容，就是给client返回response，第二个参数是请求信息的封装，包括http方法、http body等
* `http.ListenAndServe`大概能猜到了，监听`8080`端口，等待网络IO请求过来

所以我们可以从`http.ListenAndServe`开始分析

### http.ListenAndServe分析
让我们列出大概的调用链，来看看处理流程：
```go
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
        ln, err := net.Listen("tcp", addr) if err != nil {
            ...
        return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
            for {
                rw, e := l.Accept()
                c := srv.newConn(rw)
                go c.serve(ctx)
```
这段代码主要流程如下：
* 调用`net.Listen("tcp", addr)`监听某个地址，就是提供http服务的那个地址
* 调用`l.Accept()`等待连接过来，如果一直没有连接，就一直阻塞
* 如果有新连接请求过来了，`l.Accept()`会返回一个已连接描述符
* 然后调用`go c.serve(ctx)`，这里也能猜到，真正的处理就在这呢，我们接下来就分析这个goroutine的调用

从这里可以看到每来一条新请求就会新开一个goroutine，网上有文章说这里可能会成为问题，实际上我觉得在90%的场景都是没问题的，原因如下：
* 新开goroutine是很廉价的，开个几十万都是没问题的，单机几十万的连接已经很恐怖了
* http server的keepalive设置，意味着用户不会每点击一个链接就会建立一条新的http链接，所以goroutine的创建也没有那么频繁

当然，如果实在担心这个问题，可以像skynet一样，对goroutine进行复用，这样就不会重复创建goroutine了

在分析`c.serve(ctx)`之前，有几点需要说明一下：
* `c.serve(ctx)`是一个context变量，详情见[context包](https://golang.org/pkg/context/)
* 这里Accept看上去阻塞操作，其实它是非阻塞io，只不过golang的runtime为我们处理了，它只是在这个goroutine中是阻塞的，并没有阻塞系统线程，当有连接请求过来时，golang的runtime就会唤醒这个goroutine继续执行

accept的Nonblock怎么设置的？在`/usr/local/go/src/net/sys_cloexec.go`的31行：
```go
if err = syscall.SetNonblock(s, true); err != nil {
```

已连接描述符的Nonblock怎么设置的？在`/usr/local/go/src/internal/poll/sys_cloexec.go`的31行
```go
if err = syscall.SetNonblock(ns, true); err != nil {
```

## c.server(ctx)的主流程分析
调用链：
```go
func (c *conn) serve(ctx context.Context) {
    defer func() {
        c.close()

    for {
        w, err := c.readRequest(ctx)
        serverHandler{c.server}.ServeHTTP(w, w.req)
            handler := sh.srv.Handler
            if handler == nil {
                handler = DefaultServeMux

            handler.ServeHTTP(rw, req)
                func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
                    h, _ := mux.Handler(r)
                    h.ServeHTTP(w, r)
```
流程如下：
* 调用`c.readRequest(ctx)`读到网络请求，这里其实也是通过goroutine+底层runtime的实现将异步io转换成goroutine内部的同步调用
* 读到网络请求以后，调用`serverHandler{c.server}.ServeHTTP(w, w.req)`来处理这个请求
* `ServeHTTP`会调用`handler.ServeHTTP(rw, req)`调用实际的处理函数来处理，至于怎么实现的`http.HandleFunc`注册路由函数，到这里的调用关系，这个下面再说

在分析路由注册与回调之前，有必要弄清楚是怎么处理keepalive的

### keepalive分析
在上面的分析我们知道，调用`l.Accept()`会得到一个已连接描述符，那么这个已连接描述符是怎么复用的呢？可以看到`c.serve(ctx)`里面的调用是一个for循环，循环不断的读取网络请求

那么什么时候关闭这个网络连接呢？我们看到`c.serve(ctx)`函数中的defer有调用一个`c.close()`，defer的调用需要函数执行完返回，但是函数什么时候会返回呢？看看`c.readRequest(ctx)`的代码：
```go
func (c *conn) readRequest(ctx context.Context) (w *response, err error) {
    c.rwc.SetReadDeadline(hdrDeadline)
        func (c *conn) SetReadDeadline(t time.Time) error {
            if err := c.fd.SetReadDeadline(t); err != nil {
                return fd.pfd.SetReadDeadline(t)
                    return setDeadlineImpl(fd, t, 'r')
                        runtime_pollSetDeadline(fd.pd.runtimeCtx, d, mode)
```
可以看到最终对`fd`调用了一个`runtime_pollSetDeadline`，从字面意思我们都可以知道，是runtime底层维护了一个定时器，我们如果设置了`ReadTimeout`之类的东西，那么runtime就会为我们注册一个定时器，如果超过时间没有可读取的内容，`readRequest`会返回一个err，当for循环中检测到`err != nil`整个函数也就`return`了，就会执行`c.close()`，这样这个goroutine也会关闭

这样就实现了连接的复用与服务端的超时关闭功能

当然默认是没有设置这个超时的，这样关闭连接的控制权就在client端了，如果想设置超时，前面的代码可以改成这样：
```go
package main

import (
	"io"
    "net/http"
    "time"
)

func main() {
	helloHandler := func(w http.ResponseWriter, req *http.Request) {
		io.WriteString(w, "Hello, world!\n")
	}

    http.HandleFunc("/hello", helloHandler)
    
	server := &http.Server{
		Addr:           ":8080",
		ReadTimeout:    32 * time.Second,
		WriteTimeout:   32 * time.Second,
	}
	server.ListenAndServe()
}
```

## 路由注册与回调分析
有没有发现我们在`http.ListenAndServe(":8080", nil)`中的第二个参数是`nil`，其实第二个参数起作用的点在于`server.go`的2736行：
```
handler := sh.srv.Handler
if handler == nil {
    handler = DefaultServeMux
}
```
它表示一个路由接口，只要这个路由接口实现了`ServeHTTP(ResponseWriter, *Request)`就可以作为`http.ListenAndServe(":8080", nil)`的第二个参数这里我们只分析默认的路由接口`DefaultServeMux`

`http.HandleFunc("/hello", helloHandler)`调用链：
```
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}

func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    mux.Handle(pattern, HandlerFunc(handler))

```
可以看到路由处理函数的签名必须是`func(ResponseWriter, *Request)`，然后用`HandlerFunc`做一个自定义的类型转换，这样路由处理函数就有一个`ServeHTTP`方法了，而`ServeHTTP`又会调用签名为`func(ResponseWriter, *Request)`路由处理函数本身

`mux.Handle`将`pattern`处理函数用这个map将它们映射起来

当读到网络http请求后会调用如下（前面已经分析过了）：
```
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    h, _ := mux.Handler(r)
    h.ServeHTTP(w, r)
```
这里`mux.Handler(r)`将路由处理函数被转换成`HandlerFunc`后的形式从根据访问路径从map中取出来，然后调用`HandlerFunc`的`ServeHTTP`方法，即相当于调用路由处理函数本身

到这里路由注册和回调就搞清楚了

## 平滑关闭
`net/http`还提供了一个平滑关闭的函数`Shutdown`，签名如下：
```
func (srv *Server) Shutdown(ctx context.Context) error {
    ...
```

## 总结
golang的`net/http`包足以直接运用到生产环境，对于复杂一点的路由功能，只需要提供一个路由接口即可。这让我们免去了自己重复写IO多路复用的代码了

gin就是在`net/http`的基础上提供了一个路由接口，另外对`req`对象做了一些方便的封装

关于非阻塞IO的底层实现可以参考： https://www.cntofu.com/book/3/zh/08.1.md， 虽然没细看，感觉写的还是不错的