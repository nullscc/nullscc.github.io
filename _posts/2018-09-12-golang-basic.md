---
layout: post
title: '关于golang的一些理解'
tags: golang
---
golang是google开发的语言，虽然它是强类型语言，用惯了python陡然用它可能有些微的不习惯，但是习惯过后就挺喜欢这门语言了，可能是它和docker配合起来太顺手了吧。golang已经足以满足很大部分业务需求了，虽然轮子的确不如java多，但是golang的生态也逐渐的在变好，这篇文章从不同方面说说我对golang的一些理解。

## 多态
golang的以下几个特性提供了多态的特性：
* 函数的变参数，虽然函数变参的支持极其有限，只有`...`的支持，不像python里面`*args, **kw`那样多变，但是聊胜于无
* 结构体匿名成员，这里的结构体相当于类，而相当于类的集成
* 接口与接口嵌套
* 反射

## 指针与引用
golang同时支持指针与引用类型，golang在一个结构体指针访问其方法时帮助我们自动对reciever做`p.x->(*p).x`的隐式转换，但是不像c/c++一样支持指针运算，而引用和c++中的引用差不多，golang中引用的应用比指针较普遍一些

说到引用和指针，就不得不提一下make和new方法了，简单来说我们通常会调用make来为slice, channel, map得到一个引用，而new会得到一个指针，在golang中make的用法很普遍，而new的用途相对比较稀少一点

## 函数的参数
关于golang的函数参数传递我们需要知道它是值传递，但是由于golang中引用用的很普遍，所以有时候参数的传递看起来像引用传递

比如我们声明一个slice，它的实质是一个引用，底层有一个数组和它对应，所以我们向一个函数传递一个引用类型的值时，这个值会先被复制一遍，然后由函数处理，但是如果在函数中根据这个slice值的下标进行值的修改，它修改的是底层数组元素的值，所以看起来像是引用传递

## goroutine的调度模型
goroutine的调度模型为GPM模型，它是操作系统线程+用户线程的结合，和skynet的设计有点类似。概括起来如下
* G，Goroutine，对应每个goroutine
* M，Machine，对应操作系统线程
* P，Processor，相当于用户线程的调度CPU
* 每个P维护了一个G队列，P的运行需要轮流的从P中取出来，而每个P必须绑定一个实际操作系统线程M才能运行，即在同一时间最多能并行运行的G的个数等于M的个数
* 如果某个P的G队列没有G了，那么需要从另外的P的G队列中偷一半过来
* 抢占式调度：如果某个G一直运行怎么办？通过在每个函数运行前设置抢占点

## golang的网络IO
golang的网络IO都是非阻塞的，golang的runtime通过底层epoll类似的实现来帮助我们高效轮询网络IO，让我们能在goroutine下以阻塞的方式处理网络IO，见[netpoller](http://morsmachine.dk/netpoller)

## gc与避免gc压力
高级程序语言一半都不用关系gc，它们通过一些gc算法来自动进行内存回收，垃圾回收算法主要有两种：
* 标记回收，主要问题是 stop the world
* 引用计数，主要问题是循环引用问题和维护引用的计数降低程序效率

golang采用标记回收算法，并且采用三色标记法与起一个独立的线程来处理gc，大大减少`stop the world`的影响

一般情况下，gc不会造成问题，但是如果有大量的对象创建，就有可能造成问题了，一般可以通过以下方法来避免这个问题：
* Sync.Pool，比如web框架gin就是用这种方式来避免为每次请求的context分配内存
* LeakyBuffer， 见[https://golang.google.cn/doc/effective_go.html#leaky_buffer](https://golang.google.cn/doc/effective_go.html#leaky_buffer)
* bytes.Buffer，见标准包

## 内存泄漏
一篇[关于内容泄漏的文章](https://go101.org/article/memory-leaking.html)，其实实际场景中goroutine泄漏的情况比较常见

## 避免goroutine hanging
当我们用goroutine模式编程时，如果处理不当，很容易引起goroutine hanging，导致goroutine层面的内存泄漏，所以我们需要一种机制来保证即便生产者完成了，消费者也能提前退出，解决办法有两个：
* 使用context，一般应用在web框架中，它维护一棵树语义，每个节点都能保存一份key-value组成的类似map数据的副本，当我们在goroutine之间通过context传递副本，相当于基于父节点创建一个子节点，当我们决定要关闭父节点的context时，其下所有的子节点都能收到关闭通知，从而做相应的处理
* 使用channel显示通知退出，典型的案例是pipeline并发编程模式，见[https://blog.golang.org/pipelines](https://blog.golang.org/pipelines)

## 反射与其三定律
反射是针对接口的语义，它代表了程序语言描述它自身结构的能力，它的立足点在于：接口在底层维护了(值, 值的具体类型)，反射只不过是从接口中拿出这两个对象来而已

来自： [https://blog.golang.org/laws-of-reflection](https://blog.golang.org/laws-of-reflection)

具体三定律如下：
1. Reflection goes from interface value to reflection object, 它表示能从接口值得到反射对象类型的能力，这里的反射对象类型指的是：reflect.Type和reflect.Value，这两个对象提供了很多方法让我们来检索接口的(值，值的具体类型)
2. Reflection goes from reflection object to interface value, 它主要体现在reflect.Value类型的Interface方法，它能将reflect.Value类型里面存储的(值，具体的类型)将它打包成为接口值
3. To modify a reflection object, the value must be settable, 它表示只有发射类型本身持有原对象地址的能力，这种能力决定了它是否可以通过反射对象来修改底层值

反射功能强大，但是不能过度使用，否则可能导致严重性能问题，是把双刃剑

## struct tag
struct tag用于struct，例子: 
```go
type User struct {
    UserId   int    `json:"user_id" bson:"b_user_id"`
    UserName string `json:"user_name" bson:"b_user_name"`
}
```
后面的`json:"user_id" bson:"b_user_id"`就是struct tag，它的用途包括但不限于：
* json编解码
* orm
* 字段校验

golang通过反射检索struct tag来提供这些为这些功能提供方便

## defer, panic与recover
defer的注意点：
* 执行顺序是先入后出，典型的栈
* defer对应的值是在函数定义的时候就已经确定了，而不是在执行时才确认，典型的closure语义
* defer可以读取并赋值匿名返回函数，因为它在return之后执行

recover用来恢复panic调用之后的执行点，但是它必须在defer函数中才有效

关于调用recover之后的执行点的说明：
* 假如调用recover的函数为A
* A调用B函数，B函数再调用C函数
* B函数调用完成以后执行D语句
* 如果在C中调用panic，那么recover之后的恢复执行点为D语句

上述描述的伪代码：
```go
func A() {
    defer func() {
        recover
    }
    B()
    D
}
func B() {
    C()
}
func C() {
    panic
}
```

## 总结
golang是一种抽象程度较高的程序语言，这种程序语言可以满足一般场景的使用需求，在某些特殊场景可能需要注意一些坑。语言是工具，是否强大看使用者。