## Deferred函数

`defer`语句执行的触发点是在包含该`defer`语句的函数执行完毕时(return或panic)，多条`defer`的执行顺序与其声明顺序相反。`defer`语句经常被用于处理成对的操作，如打开、关闭、连接、断开连接、加锁、释放锁。

### 数据结构

[runtime.defer](https://github.com/golang/go/blob/dev.boringcrypto.go1.16/src/runtime/runtime2.go#L907:6) 位于源码`src/runtime/runtime2.go`，`_defer`结构使用`link`字段将其串联起来，部分字段摘录如下：

~~~go
// go 1.16
type _defer struct {
	siz     int32 // 参数和结果的内存大小
	started bool // defer 是否已开始执行?
	heap    bool
	// openDefer indicates that this _defer is for a frame with open-coded
	// defers. We have only one defer record for the entire frame (which may
	// currently have 0, 1, or more defers active).
	openDefer bool
	sp        uintptr  // 存储调用 defer 函数的函数的 sp 寄存器值
	pc        uintptr  // 存储 call deferproc 的下一条汇编指令的指令地址
	fn        *funcval // defer关键字传入的函数
	_panic    *_panic  // 触发延迟调用的结构体，可能为空
	link      *_defer
}
~~~

### newdefer

对于`defer`关键字的处理，其中间表示会产生三种不同的延迟形式：

+ 分配在堆上
+ 分配在栈上
+ 开放编码式(open-coded)

默认的处理方式为分配在堆上，后两种形式，是在编译期间可判断的情况下：defer个数，return语句个数等情况，为了内存分配，执行效率而做的优化，具体的源码分析可以参考：

[defer的优化之路](https://golang.design/under-the-hood/zh-cn/part2runtime/ch09lang/defer/#923--defer1)

[defer执行机制](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#533-%E6%89%A7%E8%A1%8C%E6%9C%BA%E5%88%B6)

### 运行阶段

获取到的`_defer`结构体，会被追加到当前`goroutine`的`_defer`链表最前端，下面比较两种情况`trace`和`trace1`：

~~~go
func trace(msg string) func() {
    start := time.Now()
    log.Printf("enter %s", msg)
    return func() {
        log.Printf("exit %s (%s)", msg,time.Since(start))
    }
}

func trace1(msg string) {
    start := time.Now() 
    log.Printf("enter %s", msg)
    log.Printf("exit %s (%s)", msg,time.Since(start))
}
~~~

当使用`defer`延时调用上述两个函数时，其输出结果会有一些出入

~~~go
func main() {
    fmt.Println("start")
    defer trace("trace")()
    defer trace("trace*")
    defer trace1("trace1")
    time.Sleep(5*time.Second)
}

/* 
 start
 enter trace
 enter trace1
 exit trace1 (141.2µs)
 enter trace*
 exit trace (5.0046587s)
*/
~~~

上述defer链表应该为：

~~~shell
trace1("trace1")->trace("trace*")->trace("trace")的return匿名函数
~~~

