### 介绍

执行 Goroutine 所需的上下文，负责管理 G 队列。

调度核心，维护 G 队列，并提供执行环境。

### 结构体

```go
type p struct {
	// 唯一标识符。
	id int32
	// 当前状态。
	status uint32
	// 将空闲的 P 串联成链表。
	link puintptr
	// 绑定的 M。
	m muintptr
	// 内存分配的本地缓存。
	mcache *mcache
	// 页缓存，用于内存分配。
	pcache pageCache

	// 可复用的 _defer 结构体的池。
	deferpool []*_defer
	// 固定大小的 _defer 缓冲池。
	deferpoolbuf [32]*_defer

	// 缓存 Goroutine ID 的分配范围起点。
	goidcache uint64
	// 缓存 Goroutine ID 的分配范围终点。
	goidcacheend uint64

	// 运行队列头部
	runqhead uint32
	// 运行队列尾部
	runqtail uint32
	// P 的本地运行队列，存储可运行的 Goroutine。
	runq [256]guintptr
	// 下一个可执行的 Goroutine
	runnext guintptr

	// 状态为 Gdead 的 Goroutine 的空闲列表。
	gFree struct {
		// 存储空闲 Goroutine 的链表。
		gList
		// 空闲 Goroutine 的数量。
		n int32
	}

	// 缓存 sudog 对象。
	sudogcache []*sudog    // 动态缓存。
	sudogbuf   [128]*sudog // 固定大小的缓冲区。

	// 缓存 mspan 对象。它的主要目的是提高内存分配的性能，减少频繁从全局堆中获取 mspan 的开销。
	mspancache struct {
		len int         // 缓存中 mspan 对象的数量。
		buf [128]*mspan // 固定大小的缓存数组。
	}
	
	// 缓存单个 pinner 对象。
	pinnerCache *pinner

	// P 本地的持久性分配器。
	palloc persistentAlloc

	// P 在 mutator assists 中花费的时间（纳秒）。
	gcAssistTime int64
	// P 在分数标记工作中花费的时间（纳秒）。
	gcFractionalMarkTime int64

	// 与 GC CPU 限制器相关的事件。GC CPU 限制器用于确保垃圾回收器不会使用超过分配的 CPU 预算。
	limiterEvent limiterEvent
	
	// 决定下一个垃圾回收标记工作线程的运行模式。
	gcMarkWorkerMode gcMarkWorkerMode
	// 最近一次垃圾回收标记工作线程启动的时间戳（以纳秒为单位）。
	gcMarkWorkerStartTime int64
	
	// GC 工作缓冲区。
	gcw gcWork
	
	// P 的 GC 写屏障缓冲区。
	wbBuf wbBuf

	// 运行时在这些安全点执行的操作，用于协调调度和垃圾回收。
	// 如果值为 1，则在下一个安全点运行调度器的 safePointFn 函数。
	runSafePointFn uint32
	
	// 是否需要尽快进入调度器。
	preempt bool
}
```

### P 空闲列表

#### 写入

P 放入空闲列表

```go
func pidleput(pp *p, now int64) int64 {
	assertLockHeld(&sched.lock)
	
	if now == 0 {
		now = nanotime()
	}
	if pp.timers.len.Load() == 0 {
		timerpMask.clear(pp.id)
	}
	// 标记处理器为空闲
	idlepMask.set(pp.id)
	// 将处理器加入空闲列表
	pp.link = sched.pidle
	sched.pidle.set(pp)
	sched.npidle.Add(1)
	if !pp.limiterEvent.start(limiterEventIdle, now) {
		throw("must be able to track idle limiter event")
	}
	return now
}
```

##### 获取

从 _Pidle 列表中尝试获取一个空闲的 P（Processor），并将其与调用方（通常是一个处于“自旋”状态的 M，工作线程）绑定，以便执行工作。

```go
func pidlegetSpinning(now int64) (*p, int64) {
	assertLockHeld(&sched.lock)

	// 获取一个空闲的 P。
	pp, now := pidleget(now)
	if pp == nil {
		// 如果没有获取到空闲的 P，设置调度器的 needspinning 标志。
		// 目的是与非自旋线程同步，通知它们可能需要分配资源。
		sched.needspinning.Store(1)
		return nil, now
	}

	return pp, now
}
```

##### pidleget

从调度器的 _Pidle 列表中获取一个空闲的 P，并将其所有权交给调用方。

```go
func pidleget(now int64) (*p, int64) {
	assertLockHeld(&sched.lock)

	// 从 _Pidle 列表中获取第一个空闲的 P。
	pp := sched.pidle.ptr()
	if pp != nil {
		if now == 0 {
			now = nanotime()
		}
		// 将 P 标记为“活动”。
		timerpMask.set(pp.id)
		// 从“空闲”标记中移除。
		idlepMask.clear(pp.id)
		// 从链表中移除 P，并减少空闲 P 的计数。
		sched.pidle = pp.link
		sched.npidle.Add(-1)
		// 停止 P 的“空闲事件”计时器。
		pp.limiterEvent.stop(limiterEventIdle, now)
	}
	return pp, now
}
```

### Goroutine 空闲列表

#### 写入

将一个 Goroutine 放回当前 P 的空闲 Goroutine 列表

```go
func gfput(pp *p, gp *g) {
	if readgstatus(gp) != _Gdead {
		throw("gfput: bad status (not Gdead)")
	}

	// 栈大小检查
	stksize := gp.stack.hi - gp.stack.lo

	if stksize != uintptr(startingStackSize) {
		// non-standard stack size - free it.
		stackfree(gp.stack)
		gp.stack.lo = 0
		gp.stack.hi = 0
		gp.stackguard0 = 0
	}

	// 将 Goroutine 放入本地空闲列表
	pp.gFree.push(gp)
	pp.gFree.n++
	// 如果当前 P 的空闲列表中的 Goroutine 数量超过 64 个，则触发转移逻辑
	if pp.gFree.n >= 64 {
		var (
			inc      int32
			stackQ   gQueue
			noStackQ gQueue
		)
		// 从当前 P 的空闲列表中弹出 Goroutine，直到列表中的 Goroutine 数量少于 32 个。
		for pp.gFree.n >= 32 {
			gp := pp.gFree.pop()
			pp.gFree.n--
			if gp.stack.lo == 0 {
				noStackQ.push(gp)
			} else {
				stackQ.push(gp)
			}
			inc++
		}
		// 将 Goroutine 转移到全局空闲列表
		lock(&sched.gFree.lock)
		sched.gFree.noStack.pushAll(noStackQ)
		sched.gFree.stack.pushAll(stackQ)
		sched.gFree.n += inc
		unlock(&sched.gFree.lock)
	}
}
```

#### 获取

从空闲 Goroutine 列表（gFree）中获取一个 Goroutine，并在必要时从全局列表中批量获取 Goroutine。

```go
func gfget(pp *p) *g {
retry:
	// 如果当前 P 的空闲列表为空，并且全局空闲列表中有 Goroutine（无论是有栈的还是无栈的）
	// 将全局空闲列表空获取部分空闲 Goroutine，放入 P 的空闲列表
	if pp.gFree.empty() && (!sched.gFree.stack.empty() || !sched.gFree.noStack.empty()) {
		lock(&sched.gFree.lock)
		// 批量获取 Goroutine
		for pp.gFree.n < 32 {
			gp := sched.gFree.stack.pop()
			if gp == nil {
				gp = sched.gFree.noStack.pop()
				if gp == nil {
					break
				}
			}
			sched.gFree.n--
			pp.gFree.push(gp)
			pp.gFree.n++
		}
		unlock(&sched.gFree.lock)
		goto retry
	}
	// 从本地空闲列表中获取 Goroutine
	gp := pp.gFree.pop()
	if gp == nil {
		return nil
	}
	pp.gFree.n--
	// 如果 Goroutine 的栈存在但大小不匹配当前要求的栈大小，则释放旧栈
	if gp.stack.lo != 0 && gp.stack.hi-gp.stack.lo != uintptr(startingStackSize) {
		systemstack(func() {
			stackfree(gp.stack)
			gp.stack.lo = 0
			gp.stack.hi = 0
			gp.stackguard0 = 0
		})
	}
	// 如果 Goroutine 没有栈，则为其分配一个新栈
	if gp.stack.lo == 0 {
		systemstack(func() {
			gp.stack = stackalloc(startingStackSize)
		})
		gp.stackguard0 = gp.stack.lo + stackGuard
	}
	return gp
}
```

#### 转移

将当前 P 的空闲 Goroutine 列表中的所有 Goroutine 移动到全局的空闲 Goroutine 列表中。

```go
func gfpurge(pp *p) {
	var (
		inc      int32
		stackQ   gQueue
		noStackQ gQueue
	)
	// 遍历本地空闲列表
	for !pp.gFree.empty() {
		gp := pp.gFree.pop()
		pp.gFree.n--
		if gp.stack.lo == 0 {
			noStackQ.push(gp)
		} else {
			stackQ.push(gp)
		}
		inc++
	}
	// 将 Goroutine 移动到全局列表
	lock(&sched.gFree.lock)
	sched.gFree.noStack.pushAll(noStackQ)
	sched.gFree.stack.pushAll(stackQ)
	sched.gFree.n += inc
	unlock(&sched.gFree.lock)
}
```

### 生命周期

#### 动态调整数量

启动时或 GC 启动时动态调整 P 数量

```go
func procresize(nprocs int32) *p {
	assertLockHeld(&sched.lock)
	// 确保当前处于 STW 状态
	assertWorldStopped()

	old := gomaxprocs

	maskWords := (nprocs + 31) / 32

	// 如果 nprocs 大于当前 allp 的长度，则需要扩展 allp
	if nprocs > int32(len(allp)) {
		lock(&allpLock)
		// 如果 nprocs 小于 allp 的容量，则直接调整切片长度；否则，创建一个新的 allp 数组，并复制旧数据
		if nprocs <= int32(cap(allp)) {
			allp = allp[:nprocs]
		} else {
			nallp := make([]*p, nprocs)
			copy(nallp, allp[:cap(allp)])
			allp = nallp
		}

		if maskWords <= int32(cap(idlepMask)) {
			idlepMask = idlepMask[:maskWords]
			timerpMask = timerpMask[:maskWords]
		} else {
			nidlepMask := make([]uint32, maskWords)
			copy(nidlepMask, idlepMask)
			idlepMask = nidlepMask

			ntimerpMask := make([]uint32, maskWords)
			copy(ntimerpMask, timerpMask)
			timerpMask = ntimerpMask
		}
		unlock(&allpLock)
	}

	// 初始化新的 P
	for i := old; i < nprocs; i++ {
		pp := allp[i]
		if pp == nil {
			pp = new(p)
		}
		pp.init(i)
		atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
	}

	gp := getg()
	// 如果当前 M 绑定的 P 仍然有效，则继续使用该 P
	if gp.m.p != 0 && gp.m.p.ptr().id < nprocs {
		gp.m.p.ptr().status = _Prunning
		gp.m.p.ptr().mcache.prepareForSweep()
	} else {
		// 释放当前 P 并绑定到 allp[0]（第一个 P）
		if gp.m.p != 0 {
			gp.m.p.ptr().m = 0
		}
		gp.m.p = 0
		pp := allp[0]
		pp.m = 0
		pp.status = _Pidle
		acquirep(pp)
	}

	mcache0 = nil

	// 释放未使用的 P
	for i := nprocs; i < old; i++ {
		pp := allp[i]
		pp.destroy()
	}

	// 裁剪 allp
	if int32(len(allp)) != nprocs {
		lock(&allpLock)
		allp = allp[:nprocs]
		idlepMask = idlepMask[:maskWords]
		timerpMask = timerpMask[:maskWords]
		unlock(&allpLock)
	}

	// 遍历所有的 P，将空闲的 P 放入空闲列表，将有本地任务的 P 放入 runnablePs 链表。
	var runnablePs *p
	for i := nprocs - 1; i >= 0; i-- {
		pp := allp[i]
		if gp.m.p.ptr() == pp {
			continue
		}
		pp.status = _Pidle
		if runqempty(pp) {
			pidleput(pp, now)
		} else {
			pp.m.set(mget())
			pp.link.set(runnablePs)
			runnablePs = pp
		}
	}
	stealOrder.reset(uint32(nprocs))
	var int32p *int32 = &gomaxprocs
	atomic.Store((*uint32)(unsafe.Pointer(int32p)), uint32(nprocs))
	if old != nprocs {
		gcCPULimiter.resetCapacity(now, nprocs)
	}
	return runnablePs
}
```

初始化 P

```go
func (pp *p) init(id int32) {
	pp.id = id
	pp.status = _Pgcstop
	pp.sudogcache = pp.sudogbuf[:0]
	pp.deferpool = pp.deferpoolbuf[:0]
	pp.wbBuf.reset()
	// 初始化 mcache
	if pp.mcache == nil {
		if id == 0 {
			pp.mcache = mcache0
		} else {
			pp.mcache = allocmcache()
		}
	}
	lockInit(&pp.timers.mu, lockRankTimers)
	
	timerpMask.set(id)
	idlepMask.clear(id)
}
```

#### 执行

##### wakep

尝试向调度系统中添加一个新的 P 来执行待调度的 Goroutine

```go
func wakep() {
	// 检查是否需要启动新的线程
	if sched.nmspinning.Load() != 0 || !sched.nmspinning.CompareAndSwap(0, 1) {
		return
	}
	
	// 禁用抢占
	mp := acquirem()

	var pp *p
	lock(&sched.lock)
	// 尝试获取空闲的 P
	pp, _ = pidlegetSpinning(0)
	if pp == nil {
		// 减去前面的修改 sched.nmspinning.CompareAndSwap(0, 1)
		if sched.nmspinning.Add(-1) < 0 {
			throw("wakep: negative nmspinning")
		}
		unlock(&sched.lock)
		releasem(mp)
		return
	}
	unlock(&sched.lock)

	// 启动 M
	startm(pp, true, false)

	// 恢复抢占
	releasem(mp)
}
```

##### handoffp

将 P 从系统调用状态或锁定状态中释放出来

```go
func handoffp(pp *p) {
	// 检查本地队列是否有任务
	if !runqempty(pp) || sched.runqsize != 0 {
		startm(pp, false, false)
		return
	}
	// 检查垃圾回收任务
	if gcBlackenEnabled != 0 && gcMarkWorkAvailable(pp) {
		startm(pp, false, false)
		return
	}
	// 检查是否需要新的调度线程
	if sched.nmspinning.Load()+sched.npidle.Load() == 0 && sched.nmspinning.CompareAndSwap(0, 1) {
		sched.needspinning.Store(0)
		startm(pp, true, false)
		return
	}
	lock(&sched.lock)
	// 检查垃圾回收等待状态
	if sched.gcwaiting.Load() {
		pp.status = _Pgcstop
		pp.gcStopTime = nanotime()
		sched.stopwait--
		if sched.stopwait == 0 {
			notewakeup(&sched.stopnote)
		}
		unlock(&sched.lock)
		return
	}
	// 检查安全点函数
	if pp.runSafePointFn != 0 && atomic.Cas(&pp.runSafePointFn, 1, 0) {
		sched.safePointFn(pp)
		sched.safePointWait--
		if sched.safePointWait == 0 {
			notewakeup(&sched.safePointNote)
		}
	}
	// 检查全局任务队列
	if sched.runqsize != 0 {
		unlock(&sched.lock)
		startm(pp, false, false)
		return
	}
	// 检查网络轮询
	if sched.npidle.Load() == gomaxprocs-1 && sched.lastpoll.Load() != 0 {
		unlock(&sched.lock)
		startm(pp, false, false)
		return
	}
	
	// 无任务，放入空闲队列
	when := pp.timers.wakeTime()
	pidleput(pp, 0)
	unlock(&sched.lock)

	// 如果有定时器唤醒事件，则唤醒网络轮询器。
	if when != 0 {
		wakeNetPoller(when)
	}
}
```

#### M 绑定，释放 P

##### 绑定

将当前 M 与 P 关联起来

```go
func acquirep(pp *p) {
	wirep(pp)
	
	// 清理 mcache
	pp.mcache.prepareForSweep()
}

func wirep(pp *p) {
	gp := getg()

	// 绑定处理器和线程
	gp.m.p.set(pp)
	pp.m.set(gp.m)
	pp.status = _Prunning
}
```

##### 释放

```go
func releasep() *p {
	return releasepNoTrace()
}

func releasepNoTrace() *p {
	gp := getg()

	gp.m.p = 0
	pp.m = 0
	pp.status = _Pidle
	return pp
}
```