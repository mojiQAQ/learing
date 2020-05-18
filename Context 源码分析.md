1. ##### Context 的作用

   ​	在并发编程中，比如有一个网络请求 Request，每个 Request 都需要开启一个 goroutine 做一些事情，这些 goroutine 又可能会开启其他的goroutine。而多个 goroutine 之间如何做到同步呢？此时 channel 就必不可少了，可以通过 channel 来跟踪这些 goroutine 以达到协程之间的同步。

   ​	go 在1.7版本中实现了 Context，通过Context 来简单的控制多个 goroutine 的树形关系，维护一次请求中的资源分配、回收，中文可以称之为“上下文”。

   

2. ##### Context 的接口定义

   ```go
   type Context interface {
   	// 获取 Context 的截止时间，当到了该时间点，该 Context 会自动取消，或者在到达截止时间前调用 calcel 函数主动取消
   	Deadline() (deadline time.Time, ok bool)
   
   	// 返回一个只读的 channel，当该 channel 可读时，代表 parent Context 已经被取消
   	Done() <-chan struct{}
   
   	// 返回该 Context 取消的原因，Context 未取消时返回 nil
   	Err() error
   
   	// 通过 key 获取该 Context 上绑定的值，该值为线程安全
   	Value(key interface{}) interface{}
   }
   ```

   

3. ##### Context 的实现

   emptyCtx 实现了 Context 接口，其不能被取消、没有绑定值也没有设置截止时间。对 Context 的所有操作都是对于 emptyCtx 的再次封装作为了 parent Context。

   ```go
   // 所有的 Context 都必须有不同的地址，因此类型不能是 struct
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
   
   func (*emptyCtx) Value(key interface{}) interface{} {
   	return nil
   }
   
   func (e *emptyCtx) String() string {
   	switch e {
   	case background:
   		return "context.Background"
   	case todo:
   		return "context.TODO"
   	}
   	return "unknown empty Context"
   }
   
   var (
   	background = new(emptyCtx)
   	todo       = new(emptyCtx)
   )
   
   // 返回一个不能被取消的 Context 作为根 Context, 通常用于 main 函数、初始化和测试
   func Background() Context {
   	return background
   }
   
   // 当不清楚使用哪一种 Context 时使用该 Context, 并且不能被取消
   func TODO() Context {
   	return todo
   }
   ```

   

4. ##### Context 的使用

   ```go
   // 传入 parent context，返回 child context 和一个 cancel 函数， 当 child context 被取消时，其所有衍生出的 child context 也会被取消
   func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {...}
   
   // 传入 parent context 和截止时间，返回 child context 和一个 cancel 函数，当到达截止时间时，child context 和其所有衍生出的 child context 会被取消，或者主动调用 cancel 函数去取消 child context
   func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {...}
   
   // 类似于 WithDeadline，不过传入的是一个 time.Duration，当超时后自动取消
   func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {...}
   
   // 与 cancel 函数无关，是为了生成一个绑定了 k-v 对的 Context，通常用于通过上下文传递数据，如 trace 追踪系统
   func WithValue(parent Context, key, val interface{}) Context {...}
   ```

   

5. ##### WithCancel 函数分析

   1. 初始化 cancelCtx，其继承了 Context，并实现了 cancel 方法

   ```go
   func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
   	c := newCancelCtx(parent)
   	propagateCancel(parent, &c)
   	return &c, func() { c.cancel(true, Canceled) }
   }
   
   // 初始化一个 cancelCtx，其继承了 Context interface
   func newCancelCtx(parent Context) cancelCtx {
   	return cancelCtx{Context: parent}
   }
   
   type cancelCtx struct {
   	Context
   
   	mu       sync.Mutex            // 互斥锁保护以下参数读写安全
     done     chan struct{}         // 懒创建，第一次 cancel 时关闭该 channel
   	children map[canceler]struct{} // 第一次 cancel 时被设置为 nil
   	err      error                 // 从第一次被 cancel 时不再为 nil
   }
   
   // 懒创建，第一次调用 Done() 方法时被初始化
   func (c *cancelCtx) Done() <-chan struct{} {
   	c.mu.Lock()
   	if c.done == nil {
   		c.done = make(chan struct{})
   	}
   	d := c.done
   	c.mu.Unlock()
   	return d
   }
   
   func (c *cancelCtx) Err() error {
   	c.mu.Lock()
   	err := c.err
   	c.mu.Unlock()
   	return err
   }
   
   var Canceled = errors.New("context canceled")
   
   type canceler interface {
   	cancel(removeFromParent bool, err error)
   	Done() <-chan struct{}
   }
   ```

   2. partent context与 child context 的关联与 cancel 函数传递

   ```go
   func propagateCancel(parent Context, child canceler) {
   	// 如果是顶级 Context（如 emptyCtx），直接 return  
   	if parent.Done() == nil {
   		return
   	}
     
   	// 判断 parent context 是否为 cancelCtx
   	if p, ok := parentCancelCtx(parent); ok {
       // 加锁是为了保证 parent 的并发安全，可能或多个 child 同时被初始化
   		p.mu.Lock()
       // 当 parent 被 cancel 后，parent.err 不等于 nil 
   		if p.err != nil {
   			// 如果 parent 已经被 cancel，将本次初始化的 CancelCtx 也直接被 cancel
   			child.cancel(false, p.err)
   		} else {
   			if p.children == nil {
   				p.children = make(map[canceler]struct{})
   			}
         // 将 parent 和本次初始化的 cancelCtx 关联起来
   			p.children[child] = struct{}{}
   		}
   		p.mu.Unlock()
   	} else {
       // 如果 parent ctx 不是 cancelCtx，阻塞等待 parent done 或着 child done
   		go func() {
   			select {
   			case <-parent.Done():
   				child.cancel(false, parent.Err())
   			case <-child.Done():
   			}
   		}()
   	}
   }
   
   func parentCancelCtx(parent Context) (*cancelCtx, bool) {
   	for {
   		switch c := parent.(type) {
   		case *cancelCtx:
   			return c, true
   		case *timerCtx:
   			return &c.cancelCtx, true
   		case *valueCtx:
   			parent = c.Context
   		default:
   			return nil, false
   		}
   	}
   }
   ```
   
   3. cancel 函数分析
   
```go
   func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
   		panic("context: internal error: missing cancel error")
   	}
   	c.mu.Lock()
     // 已经被 cancel 后再次调用 cancel
   	if c.err != nil {
   		c.mu.Unlock()
   		return
   	}
   	c.err = err
     // 没有 child 的情况时，初始化为空的 channel，否则直接关闭 channel
   	if c.done == nil {
   		c.done = closedchan
   	} else {
   		close(c.done)
   	}
     
     // 在 parent context 的锁内，cancel 掉所有的 child，此时也会开启 child 的锁
   	for child := range c.children {
   		child.cancel(false, err)
   	}
   	c.children = nil
   	c.mu.Unlock()
   
     // 从外部调用 cancel 函数时，将该 context 从 parent context 中移除
   	if removeFromParent {
   		removeChild(c.Context, c)
   	}
   }
   
   func removeChild(parent Context, child canceler) {
   	p, ok := parentCancelCtx(parent)
   	if !ok {
   		return
   	}
   	p.mu.Lock()
   	if p.children != nil {
   		delete(p.children, child)
   	}
   	p.mu.Unlock()
   }
   ```
   
6. ##### 测试

   ```go
   func test2(ctx context.Context) {
   
   	fmt.Println("ctx2 is starting .")
   	ctx2, cancel2 := context.WithCancel(ctx)
   
   	idx := 0
   	for {
   		select {
   		case <- ctx2.Done():
   			fmt.Println("ctx2 done", ctx2.Err())
   			return
   		default:
   			time.Sleep(time.Second)
   			fmt.Println("ctx2 is running.", ctx2.Err())
   			idx += 1
   			if 3 == idx {
   				cancel2()
   				fmt.Println("use ctx2 cancel()", ctx2.Err())
   			}
   		}
   	}
   }
   
   func main() {
   	ctx1, cancel1 := context.WithCancel(context.TODO())
   	go test2(ctx1)
   
   	idx := 0
   	rangeFlag := true
   	for rangeFlag {
   		select {
   		case <-ctx1.Done() :
   			time.Sleep(1 * time.Second)
   			fmt.Println("ctx1 canceled", ctx1.Err())
   			rangeFlag = false
   			return
   		default:
   			time.Sleep(1 * time.Second)
   			fmt.Println("ctx1 is running", ctx1.Err())
   			idx += 1
   			if 5 == idx {
   				cancel1()
   				fmt.Println("use ctx1 cancel()", ctx1.Err())
   			}
   		}
   	}
   
   	time.Sleep(3 * time.Second)
   }
   ```

   

7. ##### 思考

   1. 当调用 parent cancel 方法时，开启了锁，而在 parent 主动 cancel child 时，child 也加了锁，如果该parent context 往下的树形关系较大时，整个 cancel 过程是否也会持续很久？
   
      整个 cancel 的传递过程以 parent 为根会向下传递，当最下层的 child 完成时，parent 才会完成，是以栈的方式进行调用的。
   
   2. 在实际使用过程中，由于 parent 的 cancel，当 child 收到 done 消息时，child 和 parent 之间如何协同以实现优雅结束？ context 包只是提供了 goroutine 之间的信号传递，而业务层面的中止，需要用户自己来实现。
   
   3. cancelCtx 的 children 字段，类型为 map[canceler]struct{}，map 的 value 貌似没有什么实际意义，仅仅是为了使用map的o(1)复杂度的寻址功能么？ struct{}{} 在 go 中并没有被实力化，且其内存地址
   
   4. 多个 context 之间是如何保证 channel 安全呢？仅从 parent 侧关闭 channel 

