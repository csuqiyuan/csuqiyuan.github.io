---
title: Golang Context 深入理解
author:
  name: qiyuan
  avatar: >-
    https://res.cloudinary.com/dkzvjuptx/image/upload/v1578820041/info/favicon_s4pmzz.jpg
pin: false
toc: true
date: 2020-08-09 16:38:19
description: golang 中控制并发有几种经典方式，一种是 WaitGroup，一种是 channel ，一种是 Context
thumbnail:
tags: golang
categories: golang
keywords: golang,context,并发控制
---

golang 中控制并发有几种经典方式，一种是 WaitGroup，一种是 channel ，一种是 Context。

我认为三者的功能并不冲突，分别负责了并发控制的两个部分；WaitGroup 的目的是为了让 main goroutine 等待其他 goroutine 执行结束，channel 是为了可以主动终止 goroutine，而 Context 的目的是为了管理多个、嵌套的 goroutine ，比如对多个 goroutine 同时执行关闭操作。三者可以分离使用，可以可以结合使用。

## WaitGroup

WaitGroup 是 sync 包下的一个类型，是一种并发控制的方式，其核心在于 Add、Done、Wait 三个方法的使用

比如我在 main 方法中创建了一个 goroutine 去做一些事情，go 会为 main 创建一个 goroutine，被称作 main goroutine ，但有个问题就是 main 函数结束时，它所创建的子 goroutine 也会结束，如何让 main goroutine 等待其他 goroutine 执行结束？

```go
var wg sync.WaitGroup

func main() {
   wg.Add(1)
   go func() {
      handle1()
      wg.Done()
   }()

   wg.Add(1)
   go handle2()
   wg.Wait()
}

func handle1() {
   for i:=0;i<4;i++ {
      log.Print("handle1")
      time.Sleep(time.Second)
   }
}

func handle2() {
   for i:=0;i<4;i++ {
      log.Print("handle2")
      time.Sleep(time.Second)
   }
   wg.Done()
}
```

上面是 WaitGroup 使用的一个例子，可以把 wg 相关的注销掉试试，handle 函数根本来不及输出就被终止了。

WaitGroup 的本质其实是一个计数器。

- Add 函数添加一个数字，这个数字要与你之后创建的 goroutine 数量一样。在上面例子我创建了两个 goroutine ，使用了两次 Add(1) ，其实与使用一次 Add(2) 的效果一样，为计数器加了对应的值。
- Done 函数是用于将计数器减去一个值，里面调用的是 Add(-1)，可以把 Done 函数写在 defer 里，以防分支太多在退出前忘了调用 Done 。
- Wait 函数是用于等待，当计数器不为 0 时，就会让 main goroutine 等待下去，直到所有的其他 goroutine 都执行完且调用 Done 了

另外有没有注意到，两个 goroutine 使用 handle 函数的方式不一样，其实是为了表示两种使用 Done 函数的方式，如果 wg 定义的是 main 函数中的局部量，那么只能使用第一种方式，这里我定义为了全局量，那么就可以在 handle2 中使用 Done 函数。

但在实际业务中，可能存在这样一种场景，我们

## Channel + Select 

大部分情况下我们都是主动等待 goroutine 结束，但如果这个 goroutine 不会自己结束呢？使用 chan + select 可以主动终止一个 goroutine ，这种比较常用于 goroutine 无限循环用于做某件事的场景，当不再需要它做这件事时，传入 chan 一个值，将它终止。下面是一个例子。

```go
var wg sync.WaitGroup
var stop chan bool

func main() {
	stop = make(chan bool)

	wg.Add(1)
	go func() {
		defer wg.Done()
		handle1()
	}()

	wg.Add(1)
	go func() {
		defer wg.Done()
		handle2()
	}()
	time.Sleep(5 * time.Second)
	stop<- true
	wg.Wait()
}

func handle1(){
	for {
		select {
		case <-stop:
			fmt.Println("stop")
			return
		default:
			fmt.Println("do something")
			time.Sleep(time.Second)
		}
	}
}

func handle2(){
	for i:=0;i<10;i++ {
		fmt.Println("handle2 do something")
		time.Sleep(time.Second)
	}
}
```

这里我将 WaitGroup 结合在一起使用了，因为我想表示两个 goroutine ，handle1 在 5 秒后被终止，但 handle2 需要执行 10 秒，这样一种情况。handle1 就是被 chan + select 模式终止的例子。

但这样的方式也有局限性，如果很多 goroutine 都需要控制结束，而这些 goroutine 又创建了很多新的子 goroutine ，即嵌套 goroutine，通过定义很多乱七八糟的 chan 来实现么？那将会非常复杂。

## Context

中文名为“上下文”，是用来设置截止日期、同步信号、传递请求相关值的结构体。这是 go 语言中独特的设计，在其他语言中很少见到类似的概念。

个人觉得 Context 是上一节 chan + select 的加强版，因为二者有很多相似之处，比如都是使用了 chan 来传达终止命令，都是使用了 select 来监控 chan。

### Context 的使用

context 包的核心是 Context 接口，里面提供了 Deadline、Done、Err、Value 四个方法。

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}

```

1. Deadline 方法第一个返回值是截止时间，到了这个时间，Context 会自动发起取消请求；
2. Done 返回一个只读的 chan ，用于判断是否发起了取消请求，在 goroutine 中使用 select 来判断，其方式可以参考上一节内容；
3. Err 返回取消的错误原因，为什么被取消；
4. Value 方法可以获取 Context 上绑定的值，Context 中可以携带 key-value 对，用于传递信息，这个值一般是线程安全的。

下面是一个使用例子，其中各种方法和函数的用法将在下一小节详细描述

```go
func main() {
  // 返回一个 ctx ，一个取消函数 cancel，用于手动终止 goroutine
	ctx, cancel := context.WithCancel(context.Background())
	go watch(ctx,"【监控1】")
	go watch(ctx,"【监控2】")
	go watch(ctx,"【监控3】")

	time.Sleep(10 * time.Second)
	fmt.Println("可以了，通知监控停止")
	cancel()
	//为了检测监控过是否停止，如果没有监控输出，就表示停止了
	time.Sleep(5 * time.Second)
}

func watch(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println(name,"监控退出，停止了...")
			return
		default:
			fmt.Println(name,"goroutine监控中...")
			time.Sleep(2 * time.Second)
		}
	}
}
```

1. 使用 WithCancel 创建了一个 ctx ，和其对应的取消方法 cancel，调用 cancel 可以使所有使用了此 ctx 的 goroutine 终止；
2. 这里启动了三个监控 goroutine ，不断循环执行监控命令，当然没有进行真的监控，用打印代替；
3. 监控函数中使用了无限循环和 select ，监控 Done 返回的 chan 是否有取消命令传来；
4. 主函数中休眠 10 s，然后调用 cancel 函数，这时使用了 ctx 的三个 goroutine 都被结束了。

> 如果一个 goroutine 要使用 ctx ，ctx 作为参数传入时一般放在参数的第一位，这是约定俗成的
>
> 在一个 goroutine 中仍然可以创建子 goroutine 并把 ctx 传入，当调用了 cancle 函数，子 goroutine 也会被终止

### Context 原理

先看看 goroutine 与 context 的设计思想。在 goroutine 构成的树形结构中对信号同步以减少资源浪费是 context 最大的作用，如图示意了 goroutine 构成的树形结构。



![](https://res.cloudinary.com/dkzvjuptx/image/upload/v1596974381/Golang/Golang WaitGroup 与 Context 深入理解/golang-context-usage_tecqg7.png)



每个 context 会从顶层 goroutine 一层一层传递下去，当上层 goroutine 出现错误需要退出时，下层 goroutine 也会及时停止无用的工作，减少额外资源的消耗。



![](https://res.cloudinary.com/dkzvjuptx/image/upload/v1596974571/Golang/Golang WaitGroup 与 Context 深入理解/golang-with-context_ynold4.png)



现在我们从源码层面上来解释一下 context 是如何做到这些的。

先来看看使用示例中 context.Background 是什么

```go
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```

这里已经实现了两个 context ，一个是 background ，一个是 todo，但其实它们两个本质上都是 emptyCtx 类型的，是一个不可取消，没设置截止时间，没携带任何值的 Context，只是名字上对其使用场景进行了区分。

- 在 main 函数、初始化及测试代码中，background 可以作为 Context 树的最顶层，也就是根 context。
- 如果我们不知道该用什么 context 时，可以使用 todo，但很少能用到它。

```go
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
```

从 emptyCtx 的代码中可以看出，它实现了 Context 接口，但所有的方法都返回了 nil ，也就意味着 emptyCtx 没有任何特殊的功能。

有了根 context ，如何衍生出更多的子 context 呢？这里说到上面示例里提到的 WithCancel 函数了，总共有四种 With 函数，分别是

```go
// 传递一个父 context 作为参数，返回子 context，并返沪i一个 cancel 函数用来取消 context
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
// 和 WithCancel 差不多，但传入了一个截止时间参数，到达这个时间点会自动取消，也可以调用 cancel 函数进行取消
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
// 与 WithDeadline 基本一样，这个表示超时取消，在多少时间后自动取消。
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
// 生成了一个绑定一个键值对的 context ，这个数据可以通过 Value 函数访问
func WithValue(parent Context, key, val interface{}) Context
```

WithTimeout 方法异常狡猾，直接复用了 WithDeadline 的全部逻辑，它的代码如下所示

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

可以看到它调用了 WithDeadline 并把第二个参数设为当前时间 + timeout

其余几个函数分别对应了几个不同类型的 context ，而且它们都实现了另一个接口 canceler，With 函数返回的 cancel 函数就是实现了 canceler 接口的 cancel 函数。

```go
type canceler interface {
	cancel(removeFromParent bool, err error)
	Done() <-chan struct{}
}

type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}

type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}

type valueCtx struct {
	Context
	key, val interface{}
}
```

看结构体名称，基本都知道上述三个结构对应的哪个 With 函数了。

下面以 WithCancel 为例分析代码。

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
  // 将父 ctx 包装为 cancelCtx 结构体
	c := newCancelCtx(parent)
  // 构建父子之间的关联，当父上下文被取消时，子上下文也会被取消
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}

func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}

func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
  // 父上下文不会触发取消信号
	if done == nil {
		return 
	}
  // 监听父 ctx 是否被取消
	select {
	case <-done:
		child.cancel(false, parent.Err()) // 父上下文已经被取消
		return
	default:
	}
	// parentCancelCtx 判断 parent 是否是 cancelCtx 类型
  if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
    // 如果被取消，child 立即被取消；否则，child 被加入 parent 的 children 列表中 
		if p.err != nil {
			child.cancel(false, p.err)
		} else {
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
    // 开启一个 goroutine 监听父、子上下文是否被取消
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}

func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return
	}
  // 向 done 传入取消信号，或者直接调用 close，也可以使关闭信号被捕获
  // closedchan 是一个初始化被关闭的 chan，关闭信号也算输出
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
  // 遍历当前上下文的子上下文列表，逐一关闭
	for child := range c.children {
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```



### 更多使用示例

通过 context.WithValue 来传值

```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())

	valueCtx := context.WithValue(ctx, key, "add value")

	go watch(valueCtx)
	time.Sleep(10 * time.Second)
	cancel()

	time.Sleep(5 * time.Second)
}

func watch(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			//get value
			fmt.Println(ctx.Value(key), "is cancel")

			return
		default:
			//get value
			fmt.Println(ctx.Value(key), "int goroutine")

			time.Sleep(2 * time.Second)
		}
	}
}

复制代码
```

超时取消 context.WithTimeout

```go
package main

import (
	"fmt"
	"sync"
	"time"

	"golang.org/x/net/context"
)

var (
	wg sync.WaitGroup
)

func work(ctx context.Context) error {
	defer wg.Done()

	for i := 0; i < 1000; i++ {
		select {
		case <-time.After(2 * time.Second):
			fmt.Println("Doing some work ", i)

		// we received the signal of cancelation in this channel
		case <-ctx.Done():
			fmt.Println("Cancel the context ", i)
			return ctx.Err()
		}
	}
	return nil
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 4*time.Second)
	defer cancel()

	fmt.Println("Hey, I'm going to do some work")

	wg.Add(1)
	go work(ctx)
	wg.Wait()

	fmt.Println("Finished. I'm going home")
}

复制代码
```

截止时间 取消 context.WithDeadline

```GO
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	d := time.Now().Add(1 * time.Second)
	ctx, cancel := context.WithDeadline(context.Background(), d)

	// Even though ctx will be expired, it is good practice to call its
	// cancelation function in any case. Failure to do so may keep the
	// context and its parent alive longer than necessary.
	defer cancel()

	select {
	case <-time.After(2 * time.Second):
		fmt.Println("oversleep")
	case <-ctx.Done():
		fmt.Println(ctx.Err())
	}
}
```

### 我的理解

其实就算看了上面的示例，也有点似懂非懂的感觉，因为示例都比较简单，都只有一层 goroutine ，没有多重的嵌套啊之类的场景。

下面说说我自己的理解。

前面提到过一个叫 goroutine 组成的树形结构，叫 goroutine 树，那么相对应的也有 context 树存在，但 context 树与 goroutine 树不一定刚好对应，只要没有生成子 context，那么这个树就没有增加新分支。

这几个 With 函数都是对参数传进来的 context 进行了封装生成了子上下文，那么 context 树就增加了新的分支，对这个新的分支可以看作一个新的树，这个子 context 可以看作新树的父 context，对这个子 context 进行 cancel 操作，这个分支下的所有 context 都会发出取消请求，但父 context 不会受到影响。

那么如果不进行封装，直接把 ctx 往更深层的 goroutine 传进去，那么事实上不管传入多深层的 goroutine，context 还是同一个，所以所有的 goroutine 事实上都共享这一个上下文，context 树没有新增分支，就不能对某几个 goroutine 进行操作了。

![](https://res.cloudinary.com/dkzvjuptx/image/upload/v1596987343/Golang/Golang WaitGroup 与 Context 深入理解/goroutine_gruvna.png)

结合上图解释一下，如果从 goroutine 0 开始，都只用了一个初始的 ctx ，之后都没有进行 With 函数的调用，那么结果就是，你只能把 goroutine 0 到 goroutine 7 所有的 goroutine 终止掉，而不能只终止 goroutine 1 和 goroutine 4 这个分支。

如果你在 goroutine 1、2、3 上都调用了 WithCancel 函数，那么 context 就有了三个分支，你可以选择调用根 context ，即 goroutine 0 的上下文的 cancel 来终止所有 goroutine，也可以选择终止某个子 context 分支上的所有 goroutine，比如你可以只终止 goroutine 1、4 这条分支，或者终止 goroutine 3、6、7 这条分支，但其他分支和根上下文的 goroutine 不受影响。

如果还需要更精细的控制，还可以在更深层继承上一层的上下文，构造更深层的子上下文。