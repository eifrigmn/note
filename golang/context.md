## Context

`Context`把多个`goroutine`连接起来，以完成一个完整的业务逻辑处理，使用`Context`可以为一系列`goroutine`设置“父子关系”，从而实现数据和信号量的自上而下传递的效果。

~~~shell
                      |------> goroutine --->goroutine
req ---> goroutine----|------> goroutine
                      |------> goroutine
~~~

`Context`在相关`goroutine`之间传递上下文信息，可以归类为两类：

+ 元数据

  通过传递`k-v`数据来明确调用链，如分布式系统中的trace_id，特定业务场景下的order_id，payment_id等

  业务相关的跨api数据，如鉴权相关的token等

+ 信号

  超时信号(timeout)， 取消信号(cancel)，到期时间(deadline)

### `Context`接口

第一个创建`Context`的`goroutine`被称为`root`节点，`root`节点负责创建一个实现Context接口的具体对象，并向下传递，下游的`goroutine`对该`Context`进行封装(或什么都不做)并继续向下传递。

~~~go
// go 1.16
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
~~~

#### 四种context

> 参考：https://github.com/cch123/golang-notes/blob/master/context.md

##### emptyCtx

> 参考： https://mp.weixin.qq.com/s?__biz=Mzg2MDU1Mjc3MQ==&mid=2247489865&idx=1&sn=410ac290560160942b2f3cd137d25565&source=41#wechat_redirect

即root节点，对应的初始化方式为：`context.TODO()`或`context.Background`，二者的唯一区别是，调用`String()`方法时返回的信息不同。

##### valueCtx

主要就是为了在 ctx 中嵌入上下文数据，一个简单的 k 和 v 结构，同一个 ctx 内只支持一对 kv，需要更多的 kv 的话，会形成一棵树形结构。

~~~go
// emptyCtx ---> ctx("key1","val1") ---> ctx1("key2","val2")
//                                  |
//									|---> ctx2("key2","val3")
ctx := context.TODO()
ctx = context.WithValue(ctx, "key1", "val1")
ctx1 := context.WithValue(ctx, "key2", "val2")
ctx2 := context.WithValue(ctx, "key2", "val3")
~~~

##### cancelCtx

既然`Context`可以将相关`goroutine`组织成树状结构，那么就可以在业务逻辑结束或出现错误的时候，使用cancelCtx来(提前)结束这一执行树，后续节点便可以从`Context`中接收到通知而快速退出，相关资源被系统及时回收。

###### cancelCtx的结构如下：

~~~go
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
~~~

###### 使用：

~~~~go
func main() {
    ...
    // 初始化根节点
    // 不要使用context.TODO()，因为它是为你在不确定是否使用context时准备的
    ctx, cancel := context.WithCancel(context.Background())
    go worker(ctx)
    cancel()
    ...
}

func worker(ctx context.Context) {
    ...
    for {
        select {
            // 监听获取退出信号
            case <-ctx.Done():
            	return // returning not to leak the goroutine
			}
    }
    ...
}
~~~~

可以看出，每一个节点都需要监听`ctx.Done()`以获取退出信号，从而被系统回收。

而退出信号源于`cancel()`函数

##### 原理分析

通过`context.WithCancel()`方法来获取一个`cancelCtx`

~~~go
var Canceled = errors.New("context canceled")
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
~~~

###### newCancelCtx

作用是生成一个新的`cancelCtx`

###### propagateCancel

这里做了两件事：

+ 如果`parent`已经被取消(done或p.err)，那么直接取消`child`，否则把`child`挂载到`parent`的`children`中
+ 监控`parent`如果被取消，则取消该`child`（child.cancel）,`chiled.cancel`是一个递归调用，会遍历该节点的children结构，依次执行child.cancel，最终将这些child从根节点移除(removeChild)

需要注意

1. context是并发安全的，因为其使用了`sync.Mutex`，执行顺序为：加锁-->操作`children` --> 解锁

2. 如果用户把Context包在自定义的结构体中，如

   ~~~go
   type myCtx {
       context.Context
   }
   ~~~

   需要为`myCtx`实现`Done()`方法，例如：

   ~~~go
   func (d *myDoneCtx) Done() <-chan struct{} {
   	c := make(chan struct{})
   	return c
   }
   ~~~

   否则无法将`parent`和`child`串联起来，导致执行`cancel`方法时，`cancel`信号丢失，这也是官方博客中"Do not store Contexts inside a struct type"的原因。

##### timerCtx

~~~go
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}
~~~

可以看出，`timerCtx`是对`cancelCtx`的封装，使用`WithDeadline`和`WithTimeout`可以获得一个`timerCtx`，而`WithTimeout`在底层调用的还是`WithDeadline`函数：

~~~go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
~~~

每次创建`timerCtx`，会新建一个timer，达到设定的到期时间点时，会自动执行取消操作，关于到期时间的设定，整个`context`调用链以父节点的`deadline`为准。

具体的源码分析可以参考：

https://github.com/cch123/golang-notes/blob/master/context.md#%E4%BD%BF%E7%94%A8-timerctx-%E8%B6%85%E6%97%B6%E5%8F%96%E6%B6%88

https://golang.design/go-questions/stdlib/context/cancel/#timerctx
