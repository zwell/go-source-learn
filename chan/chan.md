### 介绍

Channel 是 Go 语言中一种并发通信机制， 让不同的 goroutine 可以安全地交换数据。 

可以理解为：线程安全的管道。

在 Go 中有一句经典话术：

> "不要通过共享内存来通信，而是通过通信来共享内存。"

#### 核心特性

| 特性 | 说明 |
|:---|:---|
| 类型安全 | 每个 channel 都有一个元素类型，比如 `chan int` 只能传 `int`。 |
| 同步/异步 | 无缓冲 channel 是同步通信，有缓冲 channel 是异步通信。 |
| 阻塞/唤醒 | send 和 receive 在必要时会阻塞，直到条件满足。 |
| 关闭(close) | 可以显式关闭 channel，通知接收方数据已发送完。 |
| 单向通道 | `chan<- T`（只发送）或 `<-chan T`（只接收）。 |

#### 基本操作

| 操作 | 示例 | 说明 |
|:---|:---|:---|
| 创建 | `c := make(chan int)` | 创建一个无缓冲 channel |
| 发送 | `c <- 1` | 往 channel 发送一个值 |
| 接收 | `v := <-c` | 从 channel 接收一个值 |
| 关闭 | `close(c)` | 关闭 channel，不能再发送，只能接收 |
| 非阻塞发送 | `select { case c <- 1: default: }` | 如果不能发送，就走 `default` |
| 非阻塞接收 | `select { case v := <-c: default: }` | 如果没有数据，就走 `default` |

#### 使用场景

##### goroutine 间传递数据

```go
c := make(chan int)

go func(){
    c <- 10	
}()

v := <-c
fmt.Println(v) // 10
```

✅ **应用场景：**
- 子任务执行完，把结果传回主线程
- 一边生产数据，一边消费数据（生产者/消费者模型）

#### 控制 goroutine 的退出，同步

```go
done := make(chan struct{})

go func(){
    // 业务处理
	...
	
	close(done) // 通知主线程
}()

<-done // 阻塞等待
```

✅ **应用场景：**
- 主协程等待子协程完成
- 超时控制（配合 `select`）
- 批量协程并发，然后统一等待（像 WaitGroup）

#### 限制并发数量（信号量模式）

```go
limit := make(chan struct{}, 3)

for i := 0; i < 10; i++ {
    limit <- struct{}{}
    fmt.Println(time.Now())
    go func() {
        defer func() { <-limit }()
        time.Sleep(3 * time.Second)
    }()
}
```

✅ **应用场景：**
- 限流，比如控制同时访问某服务的 goroutine 数量
- 爬虫、批量 API 调用

#### 实现超时机制（select + channel）

```go
c := make(chan int)

go func() {
    time.Sleep(3 * time.Second)
    c <- 1
}()

select {
case v := <-c:
    fmt.Println(v)
case <-time.After(1 * time.Second):
    fmt.Println("timeout")
}
```

✅ **应用场景：**
- 网络请求超时
- 等待任务完成，超时后放弃

#### 事件通知、广播

```go
notify := make(chan struct{})

wg := sync.WaitGroup{}

go func() {
    time.Sleep(3 * time.Second)
    close(notify)
}()

// 任务线程
wg.Add(10)
for i := 0; i < 10; i++ {
    go func() {
        select {
        case <-notify:
            fmt.Printf("receive notify %d\n", i)
            wg.Done()
        }
    }()
}

wg.Wait()
```

✅ **应用场景：**
- 主任务退出，子任务需停止

#### Pipeline（流水线处理）

**多个 goroutine 通过 channel 串联起来，一步步加工数据。**

```go

```


#### 结构体

```go
type hchan struct {
	// 通道中当前存储的元素总数
	qcount uint // total data in the queue
	// 通道的缓冲区大小（即通道可以存放多少元素）
	dataqsiz uint // size of the circular queue
	// 缓冲区的指针。指向存储通道数据的数组，数组的大小为 dataqsiz
	buf unsafe.Pointer // points to an array of dataqsiz elements
	// 每个元素的大小
	elemsize uint16
	// 是否是在同步测试中创建的
	synctest bool // true if created in a synctest bubble
	// 通道是否已经关闭
	closed uint32
	// 定时器。它用于支持通道的超时操作，比如通过 select 语句使用 time.After 函数来设定超时条件。
	timer *timer // timer feeding this chan
	// 通道元素的类型信息
	elemtype *_type // element type
	// 当前发送操作的索引。在缓冲通道中，sendx 记录下一个可以写入数据的位置，即发送数据的位置。
	sendx uint // send index
	// 当前接收操作的索引。在缓冲通道中，recvx 记录下一个可以读取数据的位置，即接收数据的位置。
	recvx uint // receive index
	// 接收等待队列（waitq）。它包含了所有因通道阻塞而等待接收数据的 goroutine。当通道中没有数据时，接收操作会将 goroutine 加入这个队列。
	recvq waitq // list of recv waiters
	// 发送等待队列（waitq）。它包含了所有因通道满而等待发送数据的 goroutine。当通道的缓冲区已满时，发送操作会将 goroutine 加入这个队列。
	sendq waitq // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```

### 主要方法

#### 创建

```go
func makechan(t *chantype, size int) *hchan {
	elem := t.Elem
	
	// 内存分配大小计算
	mem, overflow := math.MulUintptr(elem.Size_, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}
	
	// 分配内存
	var c *hchan
	switch {
	case mem == 0:
		// 元素或队列大小为零
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		c.buf = c.raceaddr()
	case !elem.Pointers():
		// 元素不包含指针
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// 元素包含指针
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	// 初始化 hchan
	c.elemsize = uint16(elem.Size_)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	if getg().syncGroup != nil {
		c.synctest = true
	}
	lockInit(&c.lock, lockRankHchan)
	
	return c
}
```

#### 发送

##### 阻塞发送（正常 `ch <- v`）

| 通道状态       | 结果                            |
|:---------------|:---------------------------------|
| **nil通道**     | 永久阻塞 |
| **正常通道**   | |
| - 有接收方等待 | 立即发送成功（同步） |
| - 无接收方，通道有缓冲且**未满** | 直接放入缓冲区，发送成功 |
| - 无接收方，通道无缓冲或**缓冲满了** | 阻塞，直到接收方到来或缓冲有空位 |
| **已关闭通道** | **panic** |

##### 非阻塞发送（用 `select { case ch <- v: ... default: ... }`）

| 通道状态 | 结果 |
|:---|:---|
| nil通道 | 直接走 `default`，不会发送 |
| 正常通道 |
| - 有接收方等待或缓冲区未满 | 发送成功 |
| - 缓冲区满 | 直接走 `default` |
| 已关闭通道 | **panic**|

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// 通道是否为 nil
	if c == nil {
		// 非阻塞模式直接返回
		if !block {
			return false
		}
		// 挂起 Goroutine，并报告异常
		gopark(nil, nil, waitReasonChanSendNilChan, traceBlockForever, 2)
		throw("unreachable")
	}

    // 非阻塞，未关闭且已满
	if !block && c.closed == 0 && full(c) {
		return false
	}

	lock(&c.lock)

	// 已关闭
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	// 如果有等待接收的 Goroutine，直接往这个 Goroutine 里写数据（此时有接收者被阻塞）
	// 减少额外的处理
	if sg := c.recvq.dequeue(); sg != nil {
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	// 缓冲区未满
	if c.qcount < c.dataqsiz {
		// 将数据写入缓冲区
		qp := chanbuf(c, c.sendx)
		// 复制数据
		typedmemmove(c.elemtype, qp, ep)
		// 更新发送索引和元素数量
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	// 非阻塞模式，发送失败
	if !block {
		unlock(&c.lock)
		return false
	}

	gp := getg()
	// 分配 sudog
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// 初始化 sudog 的字段
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	// 将 sudog 入队到通道的 sendq 队列
	c.sendq.enqueue(mysg)
	// 标记当前 Goroutine 正在被通道挂起，防止栈在此期间被调整。
	gp.parkingOnChan.Store(true)
	reason := waitReasonChanSend
	// 挂起 Goroutine（select会从发送队列取出数据来处理）
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), reason, traceBlockChanSend, 2)
	// 保持发送的数据存活
	KeepAlive(ep)

	// 被唤醒后的处理
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	// 释放 sudog
	releaseSudog(mysg)
	// 处理通道操作关闭
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}
```

发送数据

```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	// 发送数据
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	// 设置接收方参数
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	// 唤醒接收方 Goroutine
	goready(gp, skip+1)
}

// 将数据直接从当前 Goroutine 的栈拷贝到目标 Goroutine 的栈中指定的位置
// 用于处理无缓冲或缓冲已空的通道通信中的直接数据传递。
func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	// 获取目标地址
	dst := sg.elem
	// 内存屏障
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.Size_)
	// 数据拷贝
	memmove(dst, src, t.Size_)
}
```

#### 接收

##### 阻塞接收（正常 `v := <-ch`）
| 通道状态 | 行为 |
|:---|:---|
| nil通道 | 永久阻塞 |
| 正常通道 |
| - 缓冲区有数据 | 立即取出，继续执行 |
| - 无数据，有发送方同步发送 | 立即取出发送方数据 |
| - 无数据，无发送方 | 当前Goroutine阻塞，等待发送 |
| 已关闭通道 |
| - 缓冲区有数据 | 正常取出 |
| - 缓冲区无数据 | 返回零值，并 `ok = false` |

##### 非阻塞接收（用 `select { case v := <-ch: ... default: ... }`）
| 通道状态 | 行为 |
|:---|:---|
| nil通道 | 直接走 `default` |
| 正常通道 |
| - 有数据 | 立即取出 |
| - 无数据 | 直接走 `default` |
| 已关闭通道 |
| - 缓冲区有数据 | 立即取出 |
| - 缓冲区无数据 | 立即返回零值，`ok = false` |

```go
// 当前 select 是否选中, received 是否接收到数据
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // 通道为空
	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceBlockForever, 2)
		throw("unreachable")
	}

	if c.timer != nil {
		c.timer.maybeRunChan()
	}

	// 非阻塞模式，并且通道为空
	if !block && empty(c) {
		// 已关闭
		if atomic.Load(&c.closed) == 0 {
			return
		}
		// 重新check
		if empty(c) {
			if ep != nil {
				// 清空内存，防止获取到脏数据
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}

	lock(&c.lock)

	// 通道未关闭
	if c.closed != 0 {
		// 通道里没有元素
		if c.qcount == 0 {
			unlock(&c.lock)
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	} else {
		// 存在被阻塞的发送者。接收数据，并将发送者数据写入队列
		if sg := c.sendq.dequeue(); sg != nil {
			recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
			return true, true
		}
	}

	// 通道里有元素
	if c.qcount > 0 {
		// 直接从 buf 中获取
		qp := chanbuf(c, c.recvx)
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// 清空内存
		typedmemclr(c.elemtype, qp)
		// 更新接收索引
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	// 非阻塞模式
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// 阻塞模式
	gp := getg()
	// 获取 sudog
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	// 加入接收等待队列
	c.recvq.enqueue(mysg)
	if c.timer != nil {
		blockTimerChan(c)
	}
	
	// 正在阻塞标识
	gp.parkingOnChan.Store(true)
	reason := waitReasonChanReceive
	if c.synctest {
		reason = waitReasonSynctestChanReceive
	}
	// 挂起 Goroutine
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), reason, traceBlockChanRecv, 2)

	// 接收完成
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	if c.timer != nil {
		unblockTimerChan(c)
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	// 释放 sudog
	releaseSudog(mysg)
	return true, success
}
```

接收数据

```go
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	// 非缓冲队列
	if c.dataqsiz == 0 {
		if raceenabled {
			racesync(c, sg)
		}
		if ep != nil {
			// 直接把 sg 里的数据复制到 ep(接收方)
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// 接收数据的位置
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
			racenotify(c, c.recvx, sg)
		}
		// 将 qp 里的数据拷贝到 ep
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// 将 sg 的数据复制到 qp
		// 相当于先从插槽中取数据，然后将发送者的数据写入进去
		typedmemmove(c.elemtype, qp, sg.elem)
		// 更新接收索引
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		// 更新发送索引
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	// 接收成功
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	// 唤醒 sudog 关联的 Goroutine（发送者）
	goready(gp, skip+1)
}

func recvDirect(t *_type, sg *sudog, dst unsafe.Pointer) {
	// 获取目标地址
	src := sg.elem
	// 内存屏障
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.Size_)
	// 数据拷贝
	memmove(dst, src, t.Size_)
}
```

#### 关闭

```go
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	// 已关闭
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}
	
	// 设置关闭标志
	c.closed = 1

	var glist gList

	// 释放所有等待接收的 Goroutine
	for {
		// 从通道的接收队列中取出所有等待接收数据的 sudog
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		// 如果接收方已分配了接收数据存储空间，则清除这部分内存
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		// 如果记录了释放时间，更新为当前时间戳。
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		// 通过 gp.param 传递 sudog 的指针，通知等待的 Goroutine
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		// 加入待唤醒列表
		glist.push(gp)
	}

	// 释放所有等待写入的 Goroutine
	for {
		// 循环从通道的发送队列中取出所有等待发送数据的 sudog
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		// 清空发送数据
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		// 通过 gp.param 传递 sudog 的指针，通知等待的 Goroutine
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		// 加入待唤醒列表
		glist.push(gp)
	}
	unlock(&c.lock)

	// 遍历待唤醒列表
	for !glist.empty() {
		gp := glist.pop()
		// 将其调度链接（schedlink）清零，以避免影响调度链
		gp.schedlink = 0
		// 标记为可运行状态
		goready(gp, 3)
	}
}
```