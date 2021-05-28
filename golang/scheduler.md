## 调度器

调度器的作用是在工作线程上分发ready-to-run的goroutine

### 调度器的发展：

+ 单线程调度器

+ 多线程调度器

+ 任务窃取调度器

  分为两类g的队列：本地队列和全局队列，本地队列绑定至具体的p，当某个p的g队列为空时，则从其他g本地队列或全局g队列中窃取一半的g来执行

  缺点：只能依靠g主动让出cpu资源才能触发调度，例如

  某些g可能长时间占用线程，导致对应g队列中后续的g饥饿

  垃圾回收STW时会暂停整个程序，导致整个程序无法使用

+ 抢占式调度器

  + 基于协作的抢占式调度

  即在g中添加了一个发起抢占请求的标记字段`stackguard0`，编译器在函数调用前插入`runtime.morestack`，该函数调用`runtime.newstack`函数以检查g的`stackguard0`字段，最终决定是否触发抢占以让出当前线程。这里抢占的出发点在函数调用的入口，因此被称为**基于协作的抢占式调度**

  那么，`stackguard0`字段的值在什么情况下会被更改？

  1. gc
  2. 系统监控到某个g的运行时间超过10ms

  缺点：无法覆盖一些边缘情况，如for循环或者是垃圾回收长时间占用线程(gc虽然更改了stackguard0标记，但抢占调度需要在下一次函数调用时才会触发)

  + 基于信号的抢占式调度

  可以认为是一种主动式的抢占调度，主要的优化是在gc时将当前g挂起，并向线程发送`SIGURG`信号，操作系统会中断当前线程，休眠当前函数并让出当前线程，调度器重新选择下一个g执行

+ 非均匀存储访问调度器 · 提案

### g/m/p

可以简单认为，调度器就是将m绑定到p，然后在 m 中不断循环执行调度函数(runtime.schedule)，寻找可用的 g 来执行。

g的队列有三类：

runqget

globrunqget

steal

### 初始化调度器

runtime.schedinit --> runtime.procresize

`schedinit`完成一些初始化数据，设置程序创建线程的上限为1000，从环境变量`GOMAXPROCS`中获取cpu数量，并调用`runtime.procresize`，更新程序中当前可运行的最大线程数。

~~~go
func schedinit() {
	_g_ := getg()
	...

	sched.maxmcount = 10000

	...
	sched.lastpoll = uint64(nanotime())
	procs := ncpu
	if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
		procs = n
	}
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}
}
~~~

`runtime.procresize`的主要作用是，处理p数组的全局变量`allp`，使其存储的p的数量与`GOMAXPROCS`的值保持一致(新建/扩容)，将allp[0]与主线程m0绑定，将allp剩余的p组成链表放置到全局调度器`schd`的`pidl`队列中。

~~~go
func procresize(nprocs int32) *p {
    ...
    if nprocs > int32(len(allp)) {
        lock(&allpLock)
		if nprocs <= int32(cap(allp)) {
			allp = allp[:nprocs]
		} else {
			nallp := make([]*p, nprocs)
			// Copy everything up to allp's cap so we
			// never lose old allocated Ps.
			copy(nallp, allp[:cap(allp)])
			allp = nallp
		}
    }
    
    ...
    _g_ := getg()
    for i := nprocs - 1; i >= 0; i-- {
    p := allp[i]

    // ...
    // 设置 p 的状态
    p.status = _Pidle
    if _g_.m.p != 0 && _g_.m.p.ptr().id < nprocs {
		// continue to use the current P
        ...
	} else {
		....
		_g_.m.p = 0
		p := allp[0]
		p.m = 0
		p.status = _Pidle
		acquirep(p)
	}
    // 初始化时，所有 p 的 runq 都是空的，所以一定会走这个 if
    if runqempty(p) {
        // 将 p 放到全局调度器的 pidle 队列中
        pidleput(p)
    } else {
        // ...
    } 
}
~~~

### 创建goroutine

使用`go`关键字来创建一个goroutine:

~~~go
go func() {
    // to do something
}()
~~~

会被翻译成runtime.newproc并执行

~~~go
func newproc(siz int32, fn *funcval) {
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	gp := getg()
	pc := getcallerpc()
	systemstack(func() {
        // 获取新的goroutine结构体(新建/复用)
		newg := newproc1(fn, argp, siz, gp, pc)
        // 把该goroutine加入到处理器p的运行队列
		_p_ := getg().m.p.ptr()
		runqput(_p_, newg, true)

		if mainStarted {
			wakep()
		}
	})
}
~~~

由上可知，新建一个goroutine最重要的有两点：

+ 通过newproc1获取一个新的goroutine结构体，这里，会首先从当前p的gFree列表中查找空闲的groutine，查找不到的话再新建(原因是，当一个goroutine退出时，g对象会被放置到对应p的gFree列表中，减少内存开销)
+ 通过getg获取当前goroutine(main是一个特殊的goroutine`g0`)，~~并将新建的goroutine挂载到当前goroutine对应的p的执行队列runq上(runqput)~~并将其放置于执行队列(当前p的本地执行队列/全局执行队列)中(runqput)

### m的工作机制

用户代码最终在m上才能被执行，runtime中有三种线程：

+ 主线程m0

  用来执行`runtime.main`

  调用`newm(sysmon, nil)`启动sysmon

+ 用来执行系统监控`runtime.sysmon`的线程

+ 普通用户的线程

  Go语言的运行时通过`runtime.startm`启动线程来执行p，与对象g的处理方式类似，空闲的m会被放置于全局处理器的midle队列中，以方便后续需要m时直接获取，如果midle队列为空，则会调用`newm`创建一个新的m

### m和p的绑定过程：

初始化完成的工作线程m会调用`runtime.schedule`进入调度循环，核心逻辑就是，找到ready-to-run的g，将其调度到当前m上执行(`runtime.excute`)：

~~~mermaid
graph TD
schedule --> A[schedtick%61 == 0]
A --> |yes|globrunqget
A --> |no|runqget
globrunqget --> C[gp == nil]
C --> |no|execute
C --> |yes|runqget
runqget --> B[gp == nil]
B --> |no|execute
B --> |yes|findrunnable
findrunnable --> execute
~~~

#### 寻找一个 runnable 状态的 goroutine

`runtime.schedule` 函数会从下面几个地方查找待执行的 Goroutine：

1. 为了保证公平，当全局运行队列中有待执行的 Goroutine 时，通过 `schedtick` 保证有一定几率会从全局的运行队列中查找对应的 Goroutine；

2. 从处理器本地的运行队列中查找待执行的 Goroutine；

3. 如果前两种方法都没有找到 Goroutine，会通过 `runtime.findrunnable` 进行阻塞地查找 Goroutine；

   `runtime.findrunnable`的实现非常复杂，这个 300 多行的函数通过以下的过程获取可运行的 Goroutine：

   1. 从本地运行队列、全局运行队列中查找；
   2. 从网络轮询器中查找是否有 Goroutine 等待运行；
   3. 通过 `runtime.runqsteal` 尝试从其他随机的处理器中窃取待运行的 Goroutine，该函数还可能窃取处理器的计时器；

### 触发调度 #TODO

