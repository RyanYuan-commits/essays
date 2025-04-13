context 是 Golang 中的经典工具，主要在异步场景中用于实现 **并发协调** 以及对 goroutine 的生命周期控制；除此之外， context 还具有一定的 **数据存储** 能力。
## 1 核心数据结构
### 1.1 context.Context
```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)

	Done() <-chan struct{}

	Err() error

	Value(key any) any
}
```
Context 是一个 interface，定义了四个核心的 API
- `Deadline`：返回 context 的过期时间
- `Done`：返回 context 中的 channel
- `Err`：返回错误
- `Value`：返回 context 中对应的 key 值

### 1.2 标准 error
> 当一个上下文被取消时，`context` 中的 `Err()` 方法会返回一个 `error` 对象，这个对象包含了上下文被取消的具体原因。

```go
// Canceled is the error returned by Context.Err when the context is canceled.
var Canceled = errors.New("context canceled")

// DeadlineExceeded is the error returned by Context.Err when the context's
// deadline passes.
var DeadlineExceeded error = deadlineExceededError{}

type deadlineExceededError struct{}

func (deadlineExceededError) Error() string   { return "context deadline exceeded" }
func (deadlineExceededError) Timeout() bool   { return true }
func (deadlineExceededError) Temporary() bool { return true }
```
- `Canceled`：context 被 cancel 会传输该 error
- `DeadlineExceeded`：context 超时时传输该 error

## 2 emptyCtx
### 2.1 类的实现
```go
// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key any) any {
	return nil
}
```
- `emptyCtx` 是一个空的 context，本质上是一个 int
- `Deadline` 返回的是一个公元元年时间以及 false 的 flag，标识当前 context 不存在过期时间
- `Done` 方法返回一个 nil，用户向 nil 中写入或者读取数据，都会阻塞
- `Err` 方法返回的永远为 nil
- `Value` 返回的也永远为 nil

### 2.2 context.Backgroud() & context.TODO()
```go
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

// Background returns a non-nil, empty Context. It is never canceled, has no
// values, and has no deadline. It is typically used by the main function,
// initialization, and tests, and as the top-level Context for incoming
// requests.
func Background() Context {
	return background
}

// TODO returns a non-nil, empty Context. Code should use context.TODO when
// it's unclear which Context to use or it is not yet available (because the
// surrounding function has not yet been extended to accept a Context
// parameter).
func TODO() Context {
	return todo
}
```
使用 `context.Backgroud()` 和 `context.TODO()` 返回的都是 `emptyCtx` 类型的一个公共实例。
这两者主要是语义上的差别：
- `Background` 返回一个非空的 Context。它永远不会被取消，没有值，也没有截止日期。它通常由main函数、初始化和测试使用，并作为传入请求的顶级上下文。
- `TODO` 返回一个非空的 Context。当不清楚使用哪个上下文或上下文还不可用时（因为周围的函数还没有扩展到接受上下文参数），代码中应该使用 `context.TODO`

## 3 cancelCtx
### 3.1 cancelCtx 数据结构
```go
// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
	cause    error                 // set to non-nil by the first cancel call
}
```

### 3.2 Deadline 方法
cancelCtx 没有实现这个方法，仅仅是 embed 了一个带有 DeadLine 方法的 Context Interface，会调用父 Context 的 Deadline 方法。

### 3.3 Done 方法
```go
func (c *cancelCtx) Done() <-chan struct{} {
	d := c.done.Load()
	if d != nil {
		return d.(chan struct{})
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	d = c.done.Load()
	if d == nil {
		d = make(chan struct{})
		c.done.Store(d)
	}
	return d.(chan struct{})
}
```
- 基于 atomic 包，读取 cancelCtx 中的 chan，如果已经存在，直接返回
- 加锁后，再次检查 chan 是否存在，如果存在则返回
- 初始化 chan 存储到 atomic.Value 中，并返回（双重检查锁）

### 3.4 Err 方法
```go
func (c *cancelCtx) Err() error {  
  c.mu.Lock()  
  err := c.err  
  c.mu.Unlock()  
  return err  
}
```
加锁、读取、解锁、返回

### 3.5 Value 方法
```go
// &cancelCtxKey is the key that a cancelCtx returns itself for.  
var cancelCtxKey int

func (c *cancelCtx) Value(key any) any {
	if key == &cancelCtxKey {
		return c
	}
	return value(c.Context, key)
}
```
- 倘若 key 为 &cancelCtxKey，则返回 cancelCtx 自身的
- 否则循序 valueCtx 的思路取值返回

### 3.6 context.WithCancel()
#### 3.6.1 context.WithCancel()
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {  
  c := withCancel(parent)  
  return c, func() { c.cancel(true, Canceled, nil) }  
}

func withCancel(parent Context) *cancelCtx {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
	propagateCancel(parent, c)
	return c
}
```
- 验证父 Context 非空
- 注入父 Context 构造好一个新的 cancelContext
- 在 propagateCancel 方法中注册一个守护协程，以保证父 Context 终止的时候，该 cancelCtx 也会终止
- 返回一个构建好的 cancelContext 以及终止该 context 的函数

#### 3.6.2  newCancelCtx
```go
func newCancelCtx(parent Context) *cancelCtx {
	return &cancelCtx{Context: parent}
}
```
注入父 context 后，返回新的子 context

#### 3.6.3 propagateCancel
```go
// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // parent is never canceled
	}

	select {
	case <-done:
		// parent is already canceled
		child.cancel(false, parent.Err(), Cause(parent))
		return
	default:
	}

	// 如果 父 context 就是一个 cancelCtx，只需要保证 child 位于 cancelCtx 的 children 数组中即可
	// 当 cancelCtx 被取消的时候，会自动取消所有的子 Context
	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err, p.cause)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		goroutines.Add(1)
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err(), Cause(parent))
			case <-child.Done():
			}
		}()
	}
}
```
propagateCancel 方法用于传递父子 context 之间的 cancel 事件
- 如果 parent 不会被 cancel，比如 emptyCtx，直接返回
- 如果 parent 已经被 cancel，。直接 cancel 终止子节点
- 如果 parent 是 cancelCtx，只需要将子 context 添加到 parent 的 children map 即可
- 如果 parent 不是 cancelCtx，但仍然拥有 cancel 能力，就启动一个协程，通过 epoll 去监控 parent 状态，如果 parent 终止，则终止子 context
#### 3.6.4 parentCancelCtx
```go
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	done := parent.Done()
	if done == closedchan || done == nil {
		return nil, false
	}
	p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
	if !ok {
		return nil, false
	}
	pdone, _ := p.done.Load().(chan struct{})
	if pdone != done {
		return nil, false
	}
	return p, true
}
```
用于判断传入的父 context 是否是一个 cancelCtx
- 如果父 context 已经关闭或者为 nil，返回 false
- 调用父 context Value 方法，传入 &cancelCtxKey，此时判断返回的值是不是父 context 自身的指针。（基于 &cancelCtxKey 为 key 取值，返回的是 cancelCtx 自身， 这是 cancelCtx 特有的协议）
#### 3.6.5 cancelCtx.cancel
```go
// cancel closes c.done, cancels each of c's children, and, if
// removeFromParent is true, removes c from its parent's children.
// cancel sets c.cause to cause if this is the first time c is canceled.
func (c *cancelCtx) cancel(removeFromParent bool, err, cause error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	if cause == nil {
		cause = err
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	c.cause = cause
	d, _ := c.done.Load().(chan struct{})
	if d == nil {
		c.done.Store(closedchan)
	} else {
		close(d)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err, cause)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```