### 介绍

代表一个 OS 线程，负责执行 G。

### 结构体

```go
type m struct {
	// 指向调度栈 (scheduling stack) 所在的 goroutine（g）。
	// g0 是专用的调度栈，不用于用户代码的执行，主要用于运行时（runtime）的调度和垃圾回收操作。
	g0 *g
	// 切换到更大栈时的上下文信息。
	morebuf gobuf
	
	// 为此线程绑定的信号处理专用 goroutine。
	gsignal *g
	// 信号处理分配的栈。
	goSigStack gsignalStack
	// 当前线程的信号屏蔽设置。
	sigmask sigset
	// 线程局部变量。
	tls [tlsSlots]uintptr
	// 线程启动时调用的函数。
	mstartfn func()
	// 当前正在执行的 goroutine。
	curg *g
	// 致命信号（如 SIGSEGV、SIGBUS）触发时正在运行的 goroutine。
	caughtsig guintptr
	// 当前绑定的 P。
	p puintptr
	// 线程即将绑定的 P。
	nextp puintptr
	// 系统调用前绑定的 P。
	oldp puintptr
	// 唯一标识符。
	id int64
	// 是否正在执行内存分配。
	mallocing int32
	// 是否正在处理 panic 或致命错误。
	throwing throwType
	// 用于禁止当前 goroutine 的抢占调度。
	// 如果该字段非空，则当前正在运行的 goroutine（curg）不会被其他任务抢占。
	preemptoff string
	// 持有的锁数量。
	locks int32
	// 是否正在退出。
	dying int32
	// 是否在空闲状态下积极寻找任务。
	spinning bool
	// 是否因等待某种条件而被阻塞。
	blocked bool
	// 是否已初始化信号处理栈。
	newSigstack bool
	// 是否可以释放 M 及其关联资源。
	freeWait atomic.Uint32
	// 是否需要创建额外的 M。
	needextram bool
	// 将 M 挂起，等待其他事件唤醒。
	park note
	// 链接到全局 allm 列表。
	alllink *m
	// 链接到调度器的空闲队列。
	schedlink muintptr
	// 是否被绑定到特定的 goroutine。
	lockedg guintptr
	// 创建当前线程时的调用栈信息。
	createstack [32]uintptr
	// 等待任务列表。
	mWaitList mWaitList

	// 在挂起 M 时传递解锁函数。
	waitunlockf func(*g, unsafe.Pointer) bool
	// 正在等待的锁。
	waitlock unsafe.Pointer
	// 阻塞的具体原因。
	waitTraceBlockReason traceBlockReason
	
	// 链接到空闲 M 的回收队列（sched.freem）。
	freelink *m
	
	// 当前 M 是否有待处理的抢占信号。
	signalPending atomic.Uint32
	
	// 持有的锁数量。
	locksHeldLen int
	// 持有的最多 10 个锁的信息。
	locksHeld [10]heldLockInfo
}
```

### 生命周期

#### 启动

启动或唤醒一个 M 来运行给定 P。

```go
func startm(pp *p, spinning, lockheld bool) {
	mp := acquirem()
	if !lockheld {
		lock(&sched.lock)
	}
	// 如果未传递有效的 P，并且非自旋模式，则尝试从 _Pidle 获取一个空闲的 P。
	if pp == nil {
		// 从调度器的 _Pidle 列表中获取一个空闲的 P
		pp, _ = pidleget(0)
		if pp == nil {
			// 如果无法获取 P，则释放锁并退出。
			if !lockheld {
				unlock(&sched.lock)
			}
			releasem(mp)
			return
		}
	}
	// 尝试从调度器中获取一个空闲的 M。
	nmp := mget()
	if nmp == nil {
		// 如果没有空闲的 M，则创建一个新的 M。
		id := mReserveID()
		unlock(&sched.lock)

		var fn func()
		if spinning {
			// 调用方增加了 nmspinning，因此在新的 M 中设置 m.spinning。
			fn = mspinning
		}
		// 创建一个新的 M 并将 P 绑定到它。
		newm(fn, pp, id)

		if lockheld {
			lock(&sched.lock)
		}
		releasem(mp)
		return
	}
	if !lockheld {
		unlock(&sched.lock)
	}
	// 设置 M 的状态，并将 P 绑定到 M。
	nmp.spinning = spinning
	nmp.nextp.set(pp)
	// 唤醒 M。
	notewakeup(&nmp.park)
	releasem(mp)
}
```

#### 创建

启动 M 的时候，如果没有空闲的 M，会创建一个新的

##### newm

创建一个新的操作系统线程（M），并将其加入 Go 调度系统中。

```go
func newm(fn func(), pp *p, id int64) {
	acquirem()

	// 分配新的 M
	mp := allocm(pp, fn, id)
	// 设置线程的初始状态
	mp.nextp.set(pp)
	mp.sigmask = initSigmask
	// 启动新的操作系统线程
	newm1(mp)
	releasem(getg().m)
}

func newm1(mp *m) {
    execLock.rlock()
    // 创建操作系统线程
    newosproc(mp)
    execLock.runlock()
}
```

##### allocm

分配一个新的 m，并与 Go 调度器关联。

```go
func allocm(pp *p, fn func(), id int64) *m {
	allocmLock.rlock()

	acquirem()

	// 如果当前 Goroutine 没有 p，则借用传入的 p 进行分配。
	gp := getg()
	if gp.m.p == 0 {
		acquirep(pp)
	}

	// 释放空闲 M 列表
	if sched.freem != nil {
		lock(&sched.lock)
		var newList *m
		// 循环释放
		for freem := sched.freem; freem != nil; {
			// 是否可以释放 M 及其关联资源
			wait := freem.freeWait.Load()
			// 使用中，记录当前节点并跳过
			if wait == freeMWait {
				next := freem.freelink
				freem.freelink = newList
				newList = freem
				freem = next
				continue
			}
			// 可释放
			if wait == freeMStack {
				systemstack(func() {
					// 释放栈内存
					stackfree(freem.g0.stack)
				})
			}
			freem = freem.freelink
		}
		sched.freem = newList
		unlock(&sched.lock)
	}

	// 初始化新的 m
	mp := new(m)
	mp.mstartfn = fn
	// 常规初始化
	mcommoninit(mp, id)
	
	// 分配栈空间
	if iscgo || mStackIsSystemAllocated() {
		mp.g0 = malg(-1)
	} else {
		mp.g0 = malg(16384 * sys.StackGuardMultiplier)
	}
	// 将 m 的指针赋给其 g0 中的 m 字段，表示这个 m 是由该 g0 管理的。
	mp.g0.m = mp

	// 释放 P
	if pp == gp.m.p.ptr() {
		releasep()
	}

	// 释放 M
	releasem(gp.m)
	allocmLock.runlock()
	return mp
}
```

##### mcommoninit

对 m 结构体的常规初始化，包括分配唯一 ID、设置随机数种子、初始化信号栈以及将 m 添加到全局 allm 列表中。

```go
func mcommoninit(mp *m, id int64) {
	gp := getg()

	lock(&sched.lock)

	// 分配线程 ID
	if id >= 0 {
		mp.id = id
	} else {
		mp.id = mReserveID()
	}

	// 初始化随机数种子
	mrandinit(mp)

	// 预初始化 m
	mpreinit(mp)
	// 如果 m 的信号栈已分配，设置其栈保护字段 stackguard1
	if mp.gsignal != nil {
		mp.gsignal.stackguard1 = mp.gsignal.stack.lo + stackGuard
	}

	// 将 m 添加到全局 allm 列表
	mp.alllink = allm
	
	// 将 mp 设置为 allm 的头节点
	atomicstorep(unsafe.Pointer(&allm), unsafe.Pointer(mp))
	unlock(&sched.lock)
	
	// 初始化性能分析栈信息
	mProfStackInit(mp)
}
```

#### 锁定

暂停当前 M，直到它绑定的 G 再次变为可运行状态

```go
func stoplockedm() {
	gp := getg()
	
	// 如果 M 当前持有 P，释放 P，并交给其他 M 继续调度
	if gp.m.p != 0 {
		pp := releasep()
		// 交给其他 M 处理
		handoffp(pp)
	}
	// 标记这个 locked M 进入 idle 状态
	incidlelocked(1)
	// 休眠 M
	mPark()
	// 被唤醒后，检查 G 是否变为 _Grunnable（可运行状态）
	status := readgstatus(gp.m.lockedg.ptr())
	if status&^_Gscan != _Grunnable {
		print("runtime:stoplockedm: lockedg (atomicstatus=", status, ") is not Grunnable or Gscanrunnable\n")
		dumpgstatus(gp.m.lockedg.ptr())
		throw("stoplockedm: not runnable")
	}
	// 重新获取 P 并恢复执行
	acquirep(gp.m.nextp.ptr())
	gp.m.nextp = 0
}
```

启动被锁定到某个 M 的 G

```go
func startlockedm(gp *g) {
	mp := gp.lockedm.ptr()
	// 递减空闲 locked M 计数
	incidlelocked(-1)
	// 释放当前 M 持有的 P
	pp := releasep()
	// 让这个 locked M 持有 P
	mp.nextp.set(pp)
	// 唤醒这个 locked M
	notewakeup(&mp.park)
	// 停止当前 M
	stopm()
}
```

#### 停止

##### gcstopm

在垃圾回收的 STW 阶段停止当前运行的 M

```go
func gcstopm() {
	gp := getg()

	// 检查垃圾回收是否处于等待状态
	if !sched.gcwaiting.Load() {
		throw("gcstopm: not waiting for gc")
	}
	// 如果是自旋状态，则将其标记为非自旋状态，并减少全局的自旋 M 计数器
	if gp.m.spinning {
		gp.m.spinning = false
		if sched.nmspinning.Add(-1) < 0 {
			throw("gcstopm: negative nmspinning")
		}
	}
	// 释放当前 P，更新状态
	pp := releasep()
	lock(&sched.lock)
	pp.status = _Pgcstop
	// 更新 GC 停止时间
	pp.gcStopTime = nanotime()
	// 减少等待计数器
	sched.stopwait--
	// 如果所有 M 都已停止，唤醒等待的线程（通常是 GC 线程）。
	if sched.stopwait == 0 {
		notewakeup(&sched.stopnote)
	}
	unlock(&sched.lock)
	// 停止当前 M
	stopm()
}
```

##### stopm

停止当前 M 的执行，直到有新的 G 可供运行

```go
func stopm() {
	gp := getg()

	// 检查当前 M 的状态
	if gp.m.locks != 0 {
		throw("stopm holding locks")
	}
	if gp.m.p != 0 {
		throw("stopm holding p")
	}
	if gp.m.spinning {
		throw("stopm spinning")
	}

	// 将当前 M 放入空闲列表
	lock(&sched.lock)
	mput(gp.m)
	unlock(&sched.lock)
	// 挂起当前 M
	mPark()
	// 唤醒后重新获取 P
	acquirep(gp.m.nextp.ptr())
	gp.m.nextp = 0
}
```

### 主要方法

##### becomeSpinning

将当前 M 标记为自旋状态

自旋状态的 M 会主动尝试从各个队列中获取可运行的 Goroutine，以提高任务调度的响应速度。

```go
func (mp *m) becomeSpinning() {
	mp.spinning = true
	sched.nmspinning.Add(1)
	sched.needspinning.Store(0)
}
```