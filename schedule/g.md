### 介绍

代表一个 Goroutine，即 Go 运行时管理的轻量级线程。

### 结构体

```go
type g struct {
	// 实际栈内存范围 [stack.lo, stack.hi)。
	stack stack
	// Go 栈增长的前置检查
	stackguard0 uintptr
	// 与 stackguard0 类似，但用于 //go:systemstack 栈增长检查。
	stackguard1 uintptr

	// 最内部（最深层）的 panic 对象。
	_panic *_panic
	// 最内部（最深层）的 defer 结构。
	_defer *_defer
	// 当前关联的 m
	m *m
	// 调度器的上下文，包括程序计数器（PC）、栈指针（SP）等。
	sched gobuf
	// 在状态为 Gsyscall 时，保存系统调用的栈指针。
	syscallsp uintptr
	// 在状态为 Gsyscall 时，保存系统调用的程序计数器。
	syscallpc uintptr
	// 在状态为 Gsyscall 时，保存系统调用的基指针，用于栈回溯。
	syscallbp uintptr
	// 栈顶的预期栈指针，用于回溯检查。
	stktopsp uintptr
	// 通用指针参数，用于不同场景中传递数据。
	// 在通道操作中指向完成阻塞操作的 sudog。
	// 在垃圾回收辅助分配中传递 GC 完成信号。
	// 用于 debugCallWrap 传递参数给新 Goroutine。
	// 在 panic 恢复时指向保存的延迟状态。
	param unsafe.Pointer
	// 状态
	atomicstatus atomic.Uint32
	// 信号处理和扫描时的锁。
	stackLock uint32 
	// 唯一标识符。
	goid uint64
	// 链接到下一个 g 的指针，用于调度队列。
	schedlink guintptr
	// 当前等待的原因，例如等待锁、通道或网络操作。
	waitreason waitReason

	// 是否有抢占信号。
	preempt bool
	// 抢占是否会导致 Goroutine 的状态变为 _Gpreempted。
	preemptStop bool
	// 是否在同步安全点处缩小栈。
	preemptShrink bool
	
	// 是否停止在异步安全点。此时栈上可能有不精确的指针信息。
	asyncSafePoint bool

	// 如果设置为 true，遇到意外的内存地址错误时会触发 panic，而不是直接崩溃。
	paniconfault bool
	// 栈是否已被扫描。受 _Gscan 状态位保护。
	gcscandone bool
	// 如果设置为 true，表示栈不能分裂（例如遇到栈分裂风险时的保护机制）。
	throwsplit bool
	// 是否存在未加锁的通道指向该 Goroutine 的栈。
	activeStackChans bool
	// 是否正准备在 chansend 或 chanrecv 上阻塞。
	parkingOnChan atomic.Bool
	// 是否正在进行垃圾回收的标记辅助工作。
	inMarkAssist bool
	
	// 被当前 Goroutine 锁定的 m 结构。
	lockedm muintptr
	// 与信号处理相关的状态信息。
	sig uint32
	// 缓存写入操作的缓冲区。
	writebuf []byte
	// 创建此 Goroutine 的父级 Goroutine 的 goid。
	parentGoid uint64
	// 创建此 Goroutine 的 go 语句的程序计数器。
	gopc uintptr
	// 入口函数地址。
	startpc uintptr
	// 正在等待的 sudog 结构，用于通道操作中的等待链。
	waiting *sudog
	// cgo 的回溯上下文。
	cgoCtxt []uintptr
	// Goroutine 是否正在参与一个 select 操作，以及是否已经有一个分支赢得了竞争。
	selectDone atomic.Uint32
	
	// 协程切换期间使用的参数指针。
	coroarg *coro
	
	// 此 Goroutine 的 GC 助力字节数。
	gcAssistBytes int64
}
```

栈地址

```go
type stack struct {
	// 栈的低地址
	lo uintptr
	// 栈的高地址
	hi uintptr
}
```

上下文信息，主要用于调度器在 Goroutine 切换时保存和恢复其状态。

```go
type gobuf struct {
	// 堆栈指针，指向 Goroutine 的栈顶
	sp uintptr
	// 程序计数器，指向下一条待执行指令的地址
	pc uintptr
	// 指向关联的 Goroutine 结构（以 `uintptr` 表示）
	g guintptr
	// 上下文指针，可能指向堆分配的函数值
	ctxt unsafe.Pointer
	// 返回地址，用于函数调用的返回
	ret uintptr
	// 链接寄存器（用于存储返回地址，ARM 等架构特有）
	lr uintptr
	// 基址指针，用于帧指针支持的架构
	bp uintptr
}
```

### Goroutine 生命周期

#### 创建

创建新 Goroutine

```go
func newproc(fn *funcval) {
	// 获取当前协程和调用方地址
	gp := getg()
	pc := sys.GetCallerPC()
	systemstack(func() {
		// 创建新协程
		newg := newproc1(fn, gp, pc, false, waitReasonZero)

		// 将新协程加入运行队列
		pp := getg().m.p.ptr()
		runqput(pp, newg, true)

		// 如果主协程已启动，通知处理器执行
		if mainStarted {
			wakep()
		}
	})
}
```

```go
func newproc1(fn *funcval, callergp *g, callerpc uintptr, parked bool, waitreason waitReason) *g {
	// 获取当前 M 和 P
	mp := acquirem()
	pp := mp.p.ptr()
	// 从 P 的缓存中获取空闲的 G
	newg := gfget(pp)
	if newg == nil {
		// 如果缓存中没有空闲的 Goroutine，分配一个新的 Goroutine
		newg = malg(stackMin)
		// 将新分配的 G 状态从 _Gidle 设置为 _Gdead（暂时不可用）
		casgstatus(newg, _Gidle, _Gdead)
		// 将新分配的 G 加入全局 Goroutine 列表，供调试器和 GC 使用
		allgadd(newg)
	}

	totalSize := uintptr(4*goarch.PtrSize + sys.MinFrameSize)
	totalSize = alignUp(totalSize, sys.StackAlign)
	sp := newg.stack.hi - totalSize

	// 初始化 G 的调度上下文
	memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched)) // 清空 G 的调度上下文 sched
	newg.sched.sp = sp
	newg.stktopsp = sp
	// 新 Goroutine 执行的函数在返回后，PC 会指向 goexit，从而自动调用 goexit() 进行清理。
	newg.sched.pc = abi.FuncPCABI0(goexit) + sys.PCQuantum
	newg.sched.g = guintptr(unsafe.Pointer(newg))
	gostartcallfn(&newg.sched, fn)
	newg.parentGoid = callergp.goid
	newg.gopc = callerpc
	newg.ancestors = saveAncestors(callergp)
	newg.startpc = fn.fn
	// 处理系统 goroutine 和用户 goroutine
	if isSystemGoroutine(newg, false) {
		sched.ngsys.Add(1)
	} else {
		// 继承父 goroutine 的同步组和性能分析标签。
		newg.syncGroup = callergp.syncGroup
		if mp.curg != nil {
			newg.labels = mp.curg.labels
		}
		if goroutineProfile.active {
			newg.goroutineProfiled.Store(goroutineProfileSatisfied)
		}
	}
	// 设置 G 的跟踪信息
	newg.trackingSeq = uint8(cheaprand())
	if newg.trackingSeq%gTrackingPeriod == 0 {
		newg.tracking = true
	}
	gcController.addScannableStack(pp, int64(newg.stack.hi-newg.stack.lo))

	// 是否需要挂起
	var status uint32 = _Grunnable
	if parked {
		status = _Gwaiting
		newg.waitreason = waitreason
	}
	// 从 P 的 ID 缓存中分配一个唯一的 goroutine ID。
	if pp.goidcache == pp.goidcacheend {
		pp.goidcache = sched.goidgen.Add(_GoidCacheBatch)
		pp.goidcache -= _GoidCacheBatch - 1
		pp.goidcacheend = pp.goidcache + _GoidCacheBatch
	}
	newg.goid = pp.goidcache
	// 切换 G 的状态
	casgstatus(newg, _Gdead, status)
	pp.goidcache++
	newg.trace.reset()

	releasem(mp)

	return newg
}
```

分配一个新的 Goroutine

通过 mallocgc 分配内存

```go
func malg(stacksize int32) *g {
	newg := new(g)
	// 是否需要分配栈
	if stacksize >= 0 {
		// 分配栈内存
		stacksize = round2(stackSystem + stacksize)
		systemstack(func() {
			newg.stack = stackalloc(uint32(stacksize))
		})
		// 初始化栈保护字段
		newg.stackguard0 = newg.stack.lo + stackGuard
		newg.stackguard1 = ^uintptr(0)
		// 清理栈底数据
		*(*uintptr)(unsafe.Pointer(newg.stack.lo)) = 0
	}
	return newg
}
```

#### ready

将 G 标记为可运行状态，并将其放入调度队列中。

```go
func ready(gp *g, traceskip int, next bool) {
	status := readgstatus(gp)

	mp := acquirem()

	if status&^_Gscan != _Gwaiting {
		dumpgstatus(gp)
		throw("bad g->status in ready")
	}
	
	// 将 goroutine 标记为可运行
	casgstatus(gp, _Gwaiting, _Grunnable)

	// 将 goroutine 放入运行队列
	runqput(mp.p.ptr(), gp, next)

	// 尝试启动一个 P 来执行
	wakep()
	releasem(mp)
}
```

#### 退出

处理 Goroutine 退出时的清理工作

```go
func goexit1() {
	mcall(goexit0)
}
func goexit0(gp *g) {
    gdestroy(gp)
	// 调度新 Goroutine
    schedule()
}
```

清理 Goroutine 的状态，并将其放回 Goroutine 空闲列表中以供复用

```go
func gdestroy(gp *g) {
	mp := getg().m
	pp := mp.p.ptr()

	// 设置 Goroutine 状态为 _Gdead
	casgstatus(gp, _Grunning, _Gdead)
	// 减少当前 P 的可扫描栈大小，因为该 Goroutine 的栈即将被释放或复用。
	gcController.addScannableStack(pp, -int64(gp.stack.hi-gp.stack.lo))
	// 如果 Goroutine 是系统 Goroutine（例如运行监控任务的 Goroutine），则减少全局系统 Goroutine 计数 sched.ngsys。
	if isSystemGoroutine(gp, false) {
		sched.ngsys.Add(-1)
	}
	// 将 Goroutine 的字段重置为初始值或零值
	gp.m = nil
	locked := gp.lockedm != 0
	gp.lockedm = 0
	mp.lockedg = 0
	gp.preemptStop = false
	gp.paniconfault = false
	gp._defer = nil
	gp._panic = nil
	gp.writebuf = nil
	gp.waitreason = waitReasonZero
	gp.param = nil
	gp.labels = nil
	gp.timer = nil
	gp.syncGroup = nil

	// 处理垃圾回收辅助字节
	if gcBlackenEnabled != 0 && gp.gcAssistBytes > 0 {
		assistWorkPerByte := gcController.assistWorkPerByte.Load()
		scanCredit := int64(assistWorkPerByte * float64(gp.gcAssistBytes))
		gcController.bgScanCredit.Add(scanCredit)
		gp.gcAssistBytes = 0
	}

	// 解除 Goroutine 与 M 的绑定
	dropg()
	
	// 将 Goroutine 放回空闲列表
	gfput(pp, gp)
}
```