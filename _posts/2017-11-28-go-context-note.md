---
layout: post
tags : [go]
title: Go 语言Context 源码分析

---

golang 中的创建一个新的 goroutine , 并不会返回像c语言类似的pid，所有我们不能从外部杀死某个goroutine. 一个请求衍生出的各个 goroutine 之间需要满足一定的约束关系，以实现一些诸如有效期，中止goroutine树，传递请求全局变量之类的功能. 

Golang 提供的解决方案是 Context.

---

## 基本Context

```go
type Context interface {
    // Done returns a channel that is closed when this `Context` is canceled
    // or times out.
    Done() <-chan struct{}

    // Err indicates why this Context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```

Context 是最重要最基本的接口.


```go
func Background() Context {
  return background
}

func TODO() Context {
  return todo
}
```

2个包级别的导出方法Background,TODO. 输出的是空context(emptyCtx), Background 是所有 Context 对象树的根，它不能被取消.

```go
var (
  background = new(emptyCtx)
  todo       = new(emptyCtx)
)
```

---

## 可取消的Context

包级别方法WithCancel返回一个组合parent Context的可取消的新Context:

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
   c := newCancelCtx(parent)
   propagateCancel(parent, &c)
   // CancelFunc调用了自己的私有方法cancel
   // true 表示把自己从parent中移除, 因为自己已经cancel掉了, parent没有必要再保存自己
   // Canceled 是一个预定义测错误
   return &c, func() { c.cancel(true, Canceled) }
 }
```

通过propagateCancel方法, WithCancel 会将parent和新的cancelCtx组织成一颗树形结构:

```go
// 核心目的是把新的child放到parent的孩子map中
func propagateCancel(parent Context, child canceler) {
  if parent.Done() == nil {
    return // parent is never canceled
  }
  // parentCancelCtx 用来找到parent中的cancelCtx
  if p, ok := parentCancelCtx(parent); ok {
    p.mu.Lock()
    if p.err != nil {
      // parent 已经被cancel了, 那么触发下级马上cancel掉
      child.cancel(false, p.err)
    } else {
      if p.children == nil {
        p.children = make(map[canceler]struct{})
      }
      // 把自己放到parent的孩子map中
      p.children[child] = struct{}{}
    }
    p.mu.Unlock()
  } else {
    // 这里表示parent 不是一个cancelCtx, 缺乏children属性, 没法进行树形结构组织
    // 只能通过监控parent的Done来传播向下取消
    go func() {
      select {
      case <-parent.Done():
        child.cancel(false, parent.Err())
      case <-child.Done():
      }
    }()
  }
}
```

WithCancel返回的Context的Done channel在以下2种情况下会被关闭:

1. 返回的CancelFunc被调用
2. parent context's Done被关闭, 也就是说, 可取消树是从parent开始的, 而不是从新的context开始.

可取消的Context实际结构如下:

```go
type cancelCtx struct {
  Context
  done chan struct{} // closed by the first cancel call.
  mu       sync.Mutex
  children map[canceler]struct{} // set to nil by the first cancel call
  err      error                 // set to non-nil by the first cancel call
}
```

cancelCtx 实现了自己的Done方法和Err, 这些并不是直接委托到自己的Context上:

```go
func (c *cancelCtx) Done() <-chan struct{} {
  return c.done
}

func (c *cancelCtx) Err() error {
  c.mu.Lock()
  defer c.mu.Unlock()
  return c.err
}
```

cancelCtx 有一个私有方法cancel, 此方法在WithCancel()的返回CancelFunc会被调用. cancel内部主要是去close 自己的done, 并且取消自己下级(children)的context.


```go
 // cancel closes c.done, cancels each of c's children, and, if
 // removeFromParent is true, removes c from its parent's children.
 func (c *cancelCtx) cancel(removeFromParent bool, err error) {
   if err == nil { // 在向下传播cancel时, 必须带上原始的error
     panic("context: internal error: missing cancel error")
   }
   c.mu.Lock()
   if c.err != nil { // 如果自己的错误不为空, 表示已经取消过了
     c.mu.Unlock()
     return
   }
   c.err = err
   // 关闭自己的done
   close(c.done)
   // 遍历下级, 取消掉
   for child := range c.children {
     // TODO 这里不知道为什么不从parent中去掉自己?
     child.cancel(false, err)
   }
   c.children = nil
   c.mu.Unlock()

   if removeFromParent {
     removeChild(c.Context, c)
   }
 }

```

---

## 可超时控制的Context

WithDeadline 通过指定绝对时间点, 控制超时取消:

```go
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) {
  if cur, ok := parent.Deadline(); ok && cur.Before(deadline) {
    // 如果parent可以更早结束, 那么返回一个包装parent的cancelCtx
    return WithCancel(parent)
  }
  c := &timerCtx{
    // 组合一个新的cancelCtx
    cancelCtx: newCancelCtx(parent),
    deadline:  deadline,
  }
  propagateCancel(parent, c) // 组织树形结构
  d := time.Until(deadline)
  if d <= 0 {
    // 如果时间已经到了, 直接触发取消
    c.cancel(true, DeadlineExceeded)
    return c, func() { c.cancel(true, Canceled) }
  }
  c.mu.Lock()
  defer c.mu.Unlock()
  if c.err == nil {
    // 新建定时器, 到期触发取消
    c.timer = time.AfterFunc(d, func() {
      c.cancel(true, DeadlineExceeded)
    })
  }
  // 返回值还有用于直接取消的CancelFunc
  return c, func() { c.cancel(true, Canceled) }
```

timerCtx 的结构如下:

```go
type timerCtx struct {
  cancelCtx
  timer *time.Timer // Under cancelCtx.mu.

  deadline time.Time
}
```

另一个导出方法WithTimeout, 可以指定相对的超时时间:

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
  return WithDeadline(parent, time.Now().Add(timeout))
}
```

---

## 带Value的Context

```go
func WithValue(parent Context, key, val interface{}) Context {
  if key == nil {
    panic("nil key")
  }
  if !reflect.TypeOf(key).Comparable() {
    panic("key is not comparable")
  }
  return &valueCtx{parent, key, val}
}
```

valueCtx 数据结构如下:

```go
type valueCtx struct {
  Context
  key, val interface{}
}
```


Value 用于取值:

```go
func (c *valueCtx) Value(key interface{}) interface{} {
  if c.key == key {
    return c.val
  }
  return c.Context.Value(key)
}
```

---

## 最佳实践

Context 的设计理念:

* Cancelation should be advisory

  取消操作应该是建议性质的, 调用者并不知道被调用者内部实现, 调用者不应该interrupt/panic 被调用者.

  调用者应该通知被调用者处理不再必要, 被调用者来决定如何处理后续操作.

  实现: 调用者和被调用者之间利用一个单向channel来实现取消信息的传递, 调用者发送取消信号(close), 被调用者通过监听此信号, 来捕获到取消操作.

* Cancelation should be transitive

  取消操作应该被传播.

  实现: Context是线程安全的, 可以传递给多个被调用者, channel 的close信号是广播性质的; 另外Context在组织上实现了父子关系的存储, 取消操作会自动向下传播.

使用时应遵循context规则:

1. 不要将 Context放入结构体, Context应该作为第一个参数传入，命名为ctx.
2. 即使函数允许, 也不要传入nil的 Context. 如果不知道用哪种Context，可以使用context.TODO().
3. 使用context的Value相关方法,只应该用于在程序和接口中传递和请求相关数据，不能用它来传递一些可选的参数
4. 相同的 Context 可以传递给在不同的goroutine; Context 是并发安全的.

