# Golang Notes



# 配置环境

```shell
/为了解决此问题，如下为设置永久环境变量
// 修改~/.bashrc
$ vim ~/.bashrc
// 配置环境变量
export PATH=$PATH:/usr/local/go/bin
export GO111MODULE=on
export GOPROXY=https://goproxy.cn,direct
//修改 /etc/profile，我个人习惯两个配置都修改
$ sudo vim /etc/profile
//配置环境变量
export PATH=$PATH:/usr/local/go/bin
// 激活配置
$ source ~/.bashrc
$ source /etc/profile
// 验证
$ go version
go version go1.14.6 linux/amd64

```

# Golang中4种 类型引用有引用类型

- `interface`
- `slice`
- `map`
- `chan`

# Ubuntu 安装docker





# Context 包

The Context should be the first parameter, typically named ctx:

context应该作为第一个参数,写为ctx

```go
func DoSomething(ctx context.Context, arg Arg) error {
	// ... use ctx ...
}
```

Do not pass a nil Context, even if a function permits it. Pass context.TODO if you are unsure about which Context to use.

不要传递空的`Context`,即使这个函数允许这么做.如何你不确定使用哪`Context`, 可以传递`context.TODO`

Use context Values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions.

仅对传递进程和api的请求作用域数据使用上下文值，而不要将可选参数传递给函数.

The same Context may be passed to functions running in different goroutines; Contexts are safe for simultaneous use by multiple goroutines.

相同的Context可以传递给运行在不同goroutines中的函数;Context被多个goroutines同时使用是安全的。

## func Background()

```go
func Background() Context
//Background returns a non-nil, empty Context. It is never canceled, has no values, and has no deadline. It is typically used by the main function, initialization, and tests, and as the top-level Context for incoming requests. 
//Background返回一个非空的Context。它永远不会被取消，没有值，也没有期限。它通常被主函数、初始化和测试使用，并作为传入请求的顶级上下文。
```

## func TODO()

```go
func TODO() Context
//TODO returns a non-nil, empty Context. Code should use context.TODO when it's unclear which Context to use or it is not yet available (because the surrounding function has not yet been extended to accept a Context parameter). TODO is recognized by static analysis tools that determine whether Contexts are propagated correctly in a program. 
//TODO返回一个非空的空上下文。当不清楚要使用哪个Context或者它还不可用时(因为周围的函数还没有被扩展到接受Context参数),代码应该使用context.TODO 。静态分析工具可以识别TODO，确定上下文是否在程序中正确传播。
```

## func WithCancel()

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
//WithCancel returns a copy of parent with a new Done channel. The returned context's Done channel is closed when the returned cancel function is called or when the parent context's Done channel is closed, whichever happens first. 
//WithCancel返回一个带有新的Done channel 的父级副本。当返回的cancel函数被调用或父上下文的Done通道被关闭时(以先发生的方式为准)，返回的context's Done channle 会被关闭，
//Canceling this context releases resources associated with it, so code should call cancel as soon as the operations running in this Context complete. 
//取消这个context会释放与它关联的资源，所以代码应该在这个上下文中运行的操作完成后立即调用cancel。

// gen generates integers in a separate goroutine and
// sends them to the returned channel.
// The callers of gen need to cancel the context once
// they are done consuming generated integers not to leak
// the internal goroutine started by gen.
gen := func(ctx context.Context) <-chan int {
    dst := make(chan int)
    n := 1
    go func() {
        for {
            select {
            case <-ctx.Done():
                return // returning not to leak the goroutine
            case dst <- n:
                n++
            }
        }
    }()
    return dst
}

ctx, cancel := context.WithCancel(context.Background())
defer cancel() // cancel when we are finished consuming integers

for n := range gen(ctx) {
    fmt.Println(n)
    if n == 5 {
        break
    }
}
```

## func WithDeadline()

```go
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
//WithDeadline returns a copy of the parent context with the deadline adjusted to be no later than d. If the parent's deadline is already earlier than d, WithDeadline(parent, d) is semantically equivalent to parent. The returned context's Done channel is closed when the deadline expires, when the returned cancel function is called, or when the parent context's Done channel is closed, whichever happens first.
//WithDeadline返回父上下文的副本，其截止日期调整为不迟于d。如果父上下文的截止日期已经早于d，则WithDeadline(parent, d)在语义上等同于父上下文。当截止日期到期时，返回的上下文的Done通道被关闭，当返回的cancel函数被调用时，或者当父上下文的Done通道被关闭时，以先发生的方式为准。

//Canceling this context releases resources associated with it, so code should call cancel as soon as the operations running in this Context complete. 
//取消这个上下文会释放与它关联的资源，所以代码应该在这个上下文中运行的操作完成后立即调用cancel。

//This example passes a context with a arbitrary deadline to tell a blocking function that it should abandon its work as soon as it gets to it. 
d := time.Now().Add(50 * time.Millisecond)
ctx, cancel := context.WithDeadline(context.Background(), d)

// Even though ctx will be expired, it is good practice to call its
// cancelation function in any case. Failure to do so may keep the
// context and its parent alive longer than necessary.
defer cancel()

select {
case <-time.After(1 * time.Second):
    fmt.Println("overslept")
case <-ctx.Done():
    fmt.Println(ctx.Err())
}
```

## func WithTimeout()

```GO
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
// WithTimeout returns WithDeadline(parent, time.Now().Add(timeout)).

//Canceling this context releases resources associated with it, so code should call cancel as soon as the operations running in this Context complete: 
//取消这个context会释放与它关联的资源，所以代码应该在这个context中运行的操作完成后立即调用cancel

//Pass a context with a timeout to tell a blocking function that it
// should abandon its work after the timeout elapses.
ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
defer cancel()

select {
case <-time.After(1 * time.Second):
    fmt.Println("overslept")
case <-ctx.Done():
    fmt.Println(ctx.Err()) // prints "context deadline exceeded"
}
```

## func WithValue()

```go
func WithValue(parent Context, key, val interface{}) Context

//WithValue returns a copy of parent in which the value associated with key is val.
//WithValue返回parent的副本，其中与key关联的值为val

//Use context Values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions.
//仅对传递进程和api的请求作用域数据使用上下文值，而不要将可选参数传递给函数。

//The provided key must be comparable and should not be of type string or any other built-in type to avoid collisions between packages using context. Users of WithValue should define their own types for keys. To avoid allocating when assigning to an interface{}, context keys often have concrete type struct{}. Alternatively, exported context key variables' static type should be a pointer or interface. 
//提供的键必须是可比较的，并且不应该是字符串类型或任何其他内置类型，以避免使用上下文的包之间的冲突。WithValue的用户应该为键定义自己的类型。为了避免在分配给接口{}时进行分配，上下文键通常具有具体的类型结构{}。或者，导出的上下文关键变量的静态类型应该是指针或接口。

type favContextKey string

f := func(ctx context.Context, k favContextKey) {
    if v := ctx.Value(k); v != nil {
        fmt.Println("found value:", v)
        return
    }
    fmt.Println("key not found:", k)
}

k := favContextKey("language")
ctx := context.WithValue(context.Background(), k, "Go")

f(ctx, k)
f(ctx, favContextKey("color"))
```

