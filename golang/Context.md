# Go 并发模式: Context

## 介绍

在Go服务中，每一个到达的请求会使用自己独有的协程处理。请求的处理者通常会起一个额外的协程取进入后端服务，例如数据库和RPC服务。这些协程集合工作在同一个请求中，通常需要访问请求特定的值，例如最终用户的标识，token认证和请求deadline。当一个请求被取消或者超时后，所有工作在这个请求的协程应该尽可能的块退出，这样系统能回收请求使用的资源。

context包可以从API边界到请求设计的所有协程过程中，轻松的传递请求作用域内的值，取消型号和deadline。

## Context

context包的核心时Content类型：

```golang
// src/context/context.go
// A Context carries a deadline, cancelation signal, and request-scoped values
// across API boundaries. Its methods are safe for simultaneous use by multiple
// goroutines.
type Context interface {
    // Done returns a channel that is closed when this Context is canceled
    // or times out.
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```

- Done方法返回一个channel，当times out或者调用cancel方法时，将会close掉。

- Err方法返回一个错误信息，用于说明为什么Context被取消了。
  
- Deadline()，返回截止时间和ok。
  
- Value()，返回值，携带请求作用域的数据。这些数据对多个协程必须是安全的。
  
## Derived contexts

context包提供一些函数去根据已有的Context去派生新的Context值。这些值是树的形式：当一个Context被取消，所有派生的Context同样或被取消。

### Backgroud

Background是所有Context树的root，永远不会被取消。TODO和Backgroud是一样的实现和作用。

```golang
// Background returns an empty Context. It is never canceled, has no deadline,
// and has no values. Background is typically used in main, init, and tests,
// and as the top-level Context for incoming requests.
func Background() Context
```

### WithCancel

### WithTimeout


### WithValue



- [Go Concurrency Patterns: Context](https://blog.golang.org/context)
- [Context godoc](https://golang.org/pkg/context/)