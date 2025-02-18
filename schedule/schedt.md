### 介绍

Go 运行时中用于管理 Goroutine 调度、线程池、全局任务队列等

todo 安全点 safePoint


##### 作用

- 管理 Goroutine 调度（负责 G 运行在哪个 P 上）
- 全局任务队列（当 P 本地队列满了，部分 G 会放入 sched.runq）
- 控制 M（线程） 数量（M 不能无限增长）
- 维护空闲 P 和 M，避免资源浪费
- GOMAXPROCS 限制 P 数量
- sysmon 监控 G 是否长期阻塞，并触发调度

### 结构体

```go
type schedt struct {
	// 最后一次网络轮询的时间，0 表示当前正在轮询。
	lastpoll atomic.Int64
	// 当前轮询需要休眠到的时间。
	pollUntil atomic.Int64

	lock mutex

	// 空闲 M 链表的头节点。
	midle muintptr
	// 空闲 M 的数量。
	nmidle int32
	// 被锁定的空闲 M 的数量。
	nmidlelocked int32
	// 已创建的 M 数量，且是下一个 M 的 ID。
	mnext int64
	// 允许的最大 M 数量，超过此值会导致崩溃。
	maxmcount int32
	// 不参与死锁检测的系统 M 数量。
	nmsys int32
	// 已经释放的 M 的总数量。
	nmfreed int64

	// 系统级 Goroutine 的数量。
	ngsys atomic.Int32

	// 空闲 P 的链表头。
	pidle puintptr
	// 空闲 P 的数量。
	npidle atomic.Int32
	// 当前自旋中的 M 的数量。
	nmspinning atomic.Int32
	// 是否需要自旋线程。
	needspinning atomic.Uint32

	// 全局可运行队列。
	runq gQueue
	// 全局可运行队列的大小。
	runqsize int32
	
	// 动态禁用调度器，暂停用户 Goroutine 的调度。
	disable struct {
		// 是否禁用用户 Goroutine 调度。
		user bool
		// 禁用时的待运行 G 队列。
		runnable gQueue 
		// runnable 队列长度。
		n int32
	}

	// 全局缓存已释放的 G。
	gFree struct {
		lock mutex
		// 带有栈的 G 链表。
		stack gList
		// 不带栈的 G 链表。
		noStack gList
		// 当前 G 的总数量。
		n int32
	}

	// 保护 sudog 缓存的互斥锁。
	sudoglock mutex
	// sudog 缓存的头节点。
	sudogcache *sudog

	// 保护 defer 缓存的互斥锁。
	deferlock mutex
	// defer 缓存的头节点。
	deferpool *_defer
	
	// 等待被释放的 M 链表，使用 m.freelink 进行链接。
	freem *m

	// GC 是否正在等待运行。
	gcwaiting atomic.Bool
	// 等待停止的 P 的数量。
	stopwait int32
	// 用于协调调度器停止所有 P 的通知机制。
	stopnote note
	// 系统监控线程是否正在等待。
	sysmonwait atomic.Bool
	// 系统监控线程的通知机制。
	sysmonnote note
	
	// 在 GC 安全点需要执行的回调函数。
	safePointFn func(*p)
	// 安全点等待的 Goroutine 数量。
	safePointWait int32
	// 安全点的同步机制。
	safePointNote note

	// 保护系统监控线程与运行时其他部分交互的锁。
	sysmonlock mutex
	
	// P 的空闲时间总量，每个 GC 周期重置。
	idleTime atomic.Int64
	
	// Goroutine 在 _Gwaiting 状态下，由于等待 sync.Mutex 或 sync.RWMutex 锁而消耗的总时间。
	totalMutexWaitTime atomic.Int64
	
	// 由于垃圾回收（GC）引起的 STW 停止延迟分布（从调用 stopTheWorldWithSema 到所有 P 停止的时间）。
	stwStoppingTimeGC timeHistogram
	// 非 GC 原因引起的 STW 停止延迟分布（如调度器暂停）。
	stwStoppingTimeOther timeHistogram
	
	// GC 相关的 STW 总时间分布（从调用 stopTheWorldWithSema 到 startTheWorldWithSema 的总时间）。
	stwTotalTimeGC timeHistogram
	// 其他原因引起的 STW 总时间分布。
	stwTotalTimeOther timeHistogram
	
	// Goroutine 在 _Grunnable 状态且绑定到 M 上时，等待运行时内部锁的总时间。
	totalRuntimeLockWaitTime atomic.Int64
}
```

### 调度

##### schedule

从可运行的 goroutine 队列中找到一个 goroutine 并执行它。

这个函数永远不会返回，因为它会不断地寻找并执行 goroutine。

```go
func schedule() {
	mp := getg().m

	if mp.locks != 0 {
		throw("schedule: holding locks")
	}

	// 处理 lockedg
	if mp.lockedg != 0 {
		// 如果当前 M 有一个被锁定的 goroutine（lockedg），则停止当前 M 并执行这个被锁定的 goroutine。
		stoplockedm()
		execute(mp.lockedg.ptr(), false)
	}

top:
	pp := mp.p.ptr()
	pp.preempt = false
	
	// 安全检查（自旋的 M 不应该有本地工作，否则可能会导致调度混乱。）
	if mp.spinning && (pp.runnext != 0 || pp.runqhead != pp.runqtail) {
		throw("schedule: spinning with local work")
	}

	// 查找可运行的 goroutine
	gp, inheritTime, tryWakeP := findRunnable() // blocks until work is available

	// 如果系统处于 freezing 状态（即全局停止状态），则死锁在这里，以防止调度器状态被破坏。
	if debug.dontfreezetheworld > 0 && freezing.Load() {
		lock(&deadlock)
		lock(&deadlock)
	}
	
	// 重置自旋状态
	if mp.spinning {
		resetspinning()
	}

	// 如果用户调度被禁用，并且当前 goroutine 不能被调度，则将其放入禁用调度队列中，并跳回 top 标签重新调度。
	if sched.disable.user && !schedEnabled(gp) {
		lock(&sched.lock)
		if schedEnabled(gp) {
			unlock(&sched.lock)
		} else {
			sched.disable.runnable.pushBack(gp)
			sched.disable.n++
			unlock(&sched.lock)
			goto top
		}
	}
	
	// 唤醒其他 P
	if tryWakeP {
		wakep()
	}
	// 如果找到的 goroutine 有一个锁定的 M，则将当前 P 交给这个锁定的 M，并跳回 top 标签重新调度。
	if gp.lockedm != 0 {
		startlockedm(gp)
		goto top
	}

	// 执行 goroutine
	execute(gp, inheritTime)
}
```

##### 查找可运行的 Goroutine

尝试从多个来源获取 Goroutine，包括本地队列、全局队列、GC 任务、trace 任务、网络轮询等。

```go
func findRunnable() (gp *g, inheritTime, tryWakeP bool) {
	mp := getg().m

top:
	pp := mp.p.ptr()
	// 如果 GC 正在运行，让当前 M 停止执行普通任务，然后重新进入循环，等待 GC 结束。
	if sched.gcwaiting.Load() {
		gcstopm()
		goto top
	}
	// 执行 Safe Point 相关的函数
	if pp.runSafePointFn != 0 {
		runSafePointFn()
	}
	
	// 记录当前时间，用于后续定时器检查
	now, pollUntil, _ := pp.timers.check(0)

	// 尝试获取 GC 任务
	if gcBlackenEnabled != 0 {
		gp, tnow := gcController.findRunnableGCWorker(pp, now)
		if gp != nil {
			return gp, false, true
		}
		now = tnow
	}

	// 偶尔（61次调度后）从全局队列获取 Goroutine，保证公平性。
	if pp.schedtick%61 == 0 && sched.runqsize > 0 {
		lock(&sched.lock)
		gp := globrunqget(pp, 1)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}

	// 从本地队列获取 Goroutine
	if gp, inheritTime := runqget(pp); gp != nil {
		return gp, inheritTime, false
	}

	// 从全局队列获取 Goroutine
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(pp, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}
	
	// 从网络轮询器获取可运行的 goroutine。
	if netpollinited() && netpollAnyWaiters() && sched.lastpoll.Load() != 0 {
		// 尝试非阻塞地检查网络事件
		if list, delta := netpoll(0); !list.empty() { // non-blocking
			// 获取一个因 I/O 解除阻塞的 goroutine
			gp := list.pop()
			// 把剩余的可运行 goroutine 放入调度队列
			injectglist(&list)
			netpollAdjustWaiters(delta)
			trace := traceAcquire()
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.ok() {
				trace.GoUnpark(gp, 0)
				traceRelease(trace)
			}
			return gp, false, false
		}
	}
	
	// 如果当前 M 处于 spinning 状态，或者需要更多 spinning M，则进入工作窃取模式
	if mp.spinning || 2*sched.nmspinning.Load() < gomaxprocs-sched.npidle.Load() {
		if !mp.spinning {
			mp.becomeSpinning()
		}

		// 尝试从其他 P 那里窃取任务
		gp, inheritTime, tnow, w, newWork := stealWork(now)
		if gp != nil {
			return gp, inheritTime, false
		}
		if newWork {
			goto top
		}

		now = tnow
		if w != 0 && (pollUntil == 0 || w < pollUntil) {
			// Earlier timer to wait for.
			pollUntil = w
		}
	}
	
	// 如果当前处于 GC 标记阶段，且 GC 任务队列有可执行的任务，尝试创建一个 GC 线程
	if gcBlackenEnabled != 0 && gcMarkWorkAvailable(pp) && gcController.addIdleMarkWorker() {
		// 从 gcBgMarkWorkerPool 取出一个 GC 任务
		node := (*gcBgMarkWorkerNode)(gcBgMarkWorkerPool.pop())
		if node != nil {
			pp.gcMarkWorkerMode = gcMarkWorkerIdleMode
			gp := node.gp.ptr()

			casgstatus(gp, _Gwaiting, _Grunnable)
			return gp, false, false
		}
		gcController.removeIdleMarkWorker()
	}
	
	// 提前记录 allp、idlepMask、timerpMask，防止 调度器状态变化导致竞争条件。
	allpSnapshot := allp
	idlepMaskSnapshot := idlepMask
	timerpMaskSnapshot := timerpMask

	lock(&sched.lock)
	// 检查 GC 状态
	if sched.gcwaiting.Load() || pp.runSafePointFn != 0 {
		unlock(&sched.lock)
		goto top
	}
	// 从全局队列获取 Goroutine
	if sched.runqsize != 0 {
		gp := globrunqget(pp, 0)
		unlock(&sched.lock)
		return gp, false, false
	}
	// 决定是否进入 spinning 状态
	if !mp.spinning && sched.needspinning.Load() == 1 {
		mp.becomeSpinning()
		unlock(&sched.lock)
		goto top
	}
	// 释放当前 P
	if releasep() != pp {
		throw("findrunnable: wrong p")
	}
	now = pidleput(pp, now)
	unlock(&sched.lock)
	
	wasSpinning := mp.spinning
	if mp.spinning {
		mp.spinning = false
		if sched.nmspinning.Add(-1) < 0 {
			throw("findrunnable: negative nmspinning")
		}
		
		lock(&sched.lock)
		// 获取一个空闲的 P，然后从全局队列获取 Goroutine
		if sched.runqsize != 0 {
			pp, _ := pidlegetSpinning(0)
			if pp != nil {
				gp := globrunqget(pp, 0)
				if gp == nil {
					throw("global runq empty with non-zero runqsize")
				}
				unlock(&sched.lock)
				acquirep(pp)
				mp.becomeSpinning()
				return gp, false, false
			}
		}
		unlock(&sched.lock)

		// 在没有 P 的情况下检查所有 P 的本地可运行队列，寻找可以窃取的 Goroutine。
		pp := checkRunqsNoP(allpSnapshot, idlepMaskSnapshot)
		if pp != nil {
			acquirep(pp)
			mp.becomeSpinning()
			goto top
		}

		// 在没有 P 的情况下，检查是否可以启动一个 GC 任务。
		pp, gp := checkIdleGCNoP()
		if pp != nil {
			acquirep(pp)
			mp.becomeSpinning()

			pp.gcMarkWorkerMode = gcMarkWorkerIdleMode
			casgstatus(gp, _Gwaiting, _Grunnable)
			return gp, false, false
		}

		// 检查所有 P 是否有比 pollUntil 更早触发的定时器，并返回新的最早触发时间。
		pollUntil = checkTimersNoP(allpSnapshot, timerpMaskSnapshot, pollUntil)
	}

	// 是否需要进行 netpoll
	if netpollinited() && (netpollAnyWaiters() || pollUntil != 0) && sched.lastpoll.Swap(0) != 0 {
		sched.pollUntil.Store(pollUntil)
		// netpoll 轮询的超时时间
		delay := int64(-1)
		if pollUntil != 0 {
			if now == 0 {
				now = nanotime()
			}
			delay = pollUntil - now
			if delay < 0 {
				delay = 0
			}
		}
		if faketime != 0 {
			// When using fake time, just poll.
			delay = 0
		}
		// 网络事件轮询
		list, delta := netpoll(delay)
		now = nanotime()
		sched.pollUntil.Store(0)
		sched.lastpoll.Store(now)
		// 处理 netpoll 结果
		if faketime != 0 && list.empty() {
			stopm()
			goto top
		}
		lock(&sched.lock)
		// 尝试获取 P
		pp, _ := pidleget(now)
		unlock(&sched.lock)
		if pp == nil {
			// 把 list 里的 G 注入全局队列
			injectglist(&list)
			netpollAdjustWaiters(delta)
		} else {
			// 如果获取到 P，处理 G
			acquirep(pp)
			if !list.empty() {
				gp := list.pop()
				injectglist(&list)
				netpollAdjustWaiters(delta)
				casgstatus(gp, _Gwaiting, _Grunnable)
				return gp, false, false
			}
			if wasSpinning {
				mp.becomeSpinning()
			}
			goto top
		}
	} else if pollUntil != 0 && netpollinited() {
		// 处理 netpoll 监听
		pollerPollUntil := sched.pollUntil.Load()
		if pollerPollUntil == 0 || pollerPollUntil > pollUntil {
			netpollBreak()
		}
	}
	stopm()
	goto top
}
```

##### 工作窃取

当某个 P 的本地队列为空时，它会尝试从其他 P 的本地队列中窃取 Goroutine 或运行 Timer，以平衡负载并提高并发性能。

```go
func stealWork(now int64) (gp *g, inheritTime bool, rnow, pollUntil int64, newWork bool) {
	pp := getg().m.p.ptr()

	ranTimer := false

	// 做多尝试窃取4次
	const stealTries = 4
	for i := 0; i < stealTries; i++ {
		stealTimersOrRunNextG := i == stealTries-1

		// 遍历所有 P
		for enum := stealOrder.start(cheaprand()); !enum.done(); enum.next() {
			// 如果 GC 正在等待，则返回并标记可能有新的工作
			if sched.gcwaiting.Load() {
				return nil, false, now, pollUntil, true
			}
			// 获取当前遍历到的 P
			p2 := allp[enum.position()]
			if pp == p2 {
				continue
			}
			
			// 在最后一次尝试时，检查目标 P 是否有 Timer 可以运行
			if stealTimersOrRunNextG && timerpMask.read(enum.position()) {
				// 检查并运行 Timer
				tnow, w, ran := p2.timers.check(now)
				now = tnow
				if w != 0 && (pollUntil == 0 || w < pollUntil) {
					pollUntil = w
				}
				// 如果运行了 Timer，检查当前 P 的本地队列是否有可运行的 Goroutine（因为运行 Timer 可能使一些 Goroutine 变为可运行状态）。
				if ran {
					if gp, inheritTime := runqget(pp); gp != nil {
						return gp, inheritTime, now, pollUntil, ranTimer
					}
					ranTimer = true
				}
			}

			// 如果目标 P 不是空闲的，则尝试从其本地队列中窃取 Goroutine
			if !idlepMask.read(enum.position()) {
				if gp := runqsteal(pp, p2, stealTimersOrRunNextG); gp != nil {
					return gp, false, now, pollUntil, ranTimer
				}
			}
		}
	}


	return nil, false, now, pollUntil, ranTimer
}
```

##### execute

将指定的 goroutine 调度到当前 M 上执行

```go
func execute(gp *g, inheritTime bool) {
	mp := getg().m
	
	// 绑定 goroutine 和 M
	mp.curg = gp
	gp.m = mp
	// 更新 goroutine 状态
	casgstatus(gp, _Grunnable, _Grunning)
	// 初始化 goroutine 的调度字段
	gp.waitsince = 0
	gp.preempt = false
	gp.stackguard0 = gp.stack.lo + stackGuard

	// 切换到 goroutine 执行
	gogo(&gp.sched)
}
```

### Goroutine 全局运行队列

#### 注入

将一个 gList 中的所有可运行的 Goroutine 添加到运行队列，并根据当前的调度状态启动适当数量的工作线程（M）来执行它们。

```go
func injectglist(glist *gList) {
	if glist.empty() {
		return
	}

	// 将 glist 中的所有 goroutine 设置为可运行状态
	head := glist.head.ptr()
	var tail *g
	qsize := 0
	for gp := head; gp != nil; gp = gp.schedlink.ptr() {
		tail = gp
		qsize++
		casgstatus(gp, _Gwaiting, _Grunnable)
	}

	// 创建 gQueue 并清空 glist
	var q gQueue
	q.head.set(head)
	q.tail.set(tail)
	*glist = gList{}

	// 启动空闲的 M（工作线程）
	startIdle := func(n int) {
		for i := 0; i < n; i++ {
			mp := acquirem()
			lock(&sched.lock)

			// 从 _Pidle 列表中尝试获取一个空闲的 P
			pp, _ := pidlegetSpinning(0)
			if pp == nil {
				unlock(&sched.lock)
				releasem(mp)
				break
			}

			// 启动或唤醒一个 M（工作线程）来运行给定 P。
			startm(pp, false, true)
			unlock(&sched.lock)
			releasem(mp)
		}
	}

	// 判断当前 P 是否为空
	pp := getg().m.p.ptr()
	if pp == nil {
		lock(&sched.lock)
		globrunqputbatch(&q, int32(qsize))
		unlock(&sched.lock)
		startIdle(qsize)
		return
	}

	// 分配空闲 P 来执行任务
	npidle := int(sched.npidle.Load())
	var (
		globq gQueue
		n     int
	)
	for n = 0; n < npidle && !q.empty(); n++ {
		g := q.pop()
		globq.pushBack(g)
	}
	if n > 0 {
		lock(&sched.lock)
		globrunqputbatch(&globq, int32(n))
		unlock(&sched.lock)
		startIdle(n)
		qsize -= n
	}

	// 将剩余的 goroutine 添加到当前 P 的运行队列
	if !q.empty() {
		runqputbatch(pp, &q, qsize)
	}

	// 唤醒 P
	wakep()
}
```

#### 写入

将 Goroutine 放入全局可运行队列。

```go
func globrunqput(gp *g) {
	assertLockHeld(&sched.lock)

	sched.runq.pushBack(gp)
	sched.runqsize++
}

func globrunqputhead(gp *g) {
    assertLockHeld(&sched.lock)
    
    sched.runq.push(gp)
    sched.runqsize++
}
```

将一批可运行的 Goroutines 放入全局可运行队列。

```go
func globrunqputbatch(batch *gQueue, n int32) {
	assertLockHeld(&sched.lock)

	// 将 batch 中的所有 Goroutines 添加到全局可运行队列。
	sched.runq.pushBackAll(*batch)
	sched.runqsize += n
	*batch = gQueue{}
}
```

#### 获取

从全局可运行队列中获取一批 G，并将它们分配到当前 P 的本地队列

```go
func globrunqget(pp *p, max int32) *g {
	assertLockHeld(&sched.lock)

	if sched.runqsize == 0 {
		return nil
	}

	// 计算需要获取的 Goroutine 数量
	n := sched.runqsize/gomaxprocs + 1
	if n > sched.runqsize {
		n = sched.runqsize
	}
	if max > 0 && n > max {
		n = max
	}
	if n > int32(len(pp.runq))/2 {
		n = int32(len(pp.runq)) / 2
	}

	// 更新全局队列大小
	sched.runqsize -= n

	// 从全局队列中获取 Goroutine
	gp := sched.runq.pop()
	n--
	for ; n > 0; n-- {
		gp1 := sched.runq.pop()
		// 将取出的 Goroutine 放入当前 P 的本地队列中
		runqput(pp, gp1, false)
	}
	// 返回从全局队列中取出的第一个 Goroutine，供当前 P 立即执行。
	return gp
}
```

#### Goroutine 运行队列

##### 写入

将 Goroutine 放入当前 P 的本地可运行队列

```go
func runqput(pp *p, gp *g, next bool) {
	// 将 Goroutine 放入 runnext 插槽
	if next {
	retryNext:
		oldnext := pp.runnext
		if !pp.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
        // 将旧的 runnext 放入常规队列
		gp = oldnext.ptr()
	}

retry:
	// 将 Goroutine 放入本地队列
	h := atomic.LoadAcq(&pp.runqhead)
	t := pp.runqtail
	if t-h < uint32(len(pp.runq)) {
		pp.runq[t%uint32(len(pp.runq))].set(gp)
		atomic.StoreRel(&pp.runqtail, t+1)
		return
	}
	// 将 Goroutine 放入全局队列
	if runqputslow(pp, gp, h, t) {
		return
	}
	goto retry
}
```

将当前 P 的本地可运行队列中的一部分 Goroutine 以及一个新的 Goroutine 放入全局可运行队列。

```go
func runqputslow(pp *p, gp *g, h, t uint32) bool {
	// 初始化批次数组
	var batch [len(pp.runq)/2 + 1]*g

	// 从本地队列中获取批次
	n := t - h
	n = n / 2
	if n != uint32(len(pp.runq)/2) {
		throw("runqputslow: queue is not full")
	}
	for i := uint32(0); i < n; i++ {
		batch[i] = pp.runq[(h+i)%uint32(len(pp.runq))].ptr()
	}
	// 更新本地队列头指针（如果 CAS 失败，表示本地队列已被其他线程修改）
	if !atomic.CasRel(&pp.runqhead, h, h+n) {
		return false
	}
	// 将新的 Goroutine 加入批次
	batch[n] = gp

	// 随机化调度
	if randomizeScheduler {
		for i := uint32(1); i <= n; i++ {
			j := cheaprandn(i + 1)
			batch[i], batch[j] = batch[j], batch[i]
		}
	}

	// 将批次数组中的 Goroutine 链接成一个链表
	for i := uint32(0); i < n; i++ {
		batch[i].schedlink.set(batch[i+1])
	}
	var q gQueue
	q.head.set(batch[0])
	q.tail.set(batch[n])

	// 将批次放入全局队列
	lock(&sched.lock)
	globrunqputbatch(&q, int32(n+1))
	unlock(&sched.lock)
	return true
}
```

将一批 Goroutine（G）放入当前 P（Processor）的本地可运行队列（runq）。

如果本地队列已满，则将剩余的 Goroutine 放入全局队列。

```go
func runqputbatch(pp *p, q *gQueue, qsize int) {
	h := atomic.LoadAcq(&pp.runqhead)
	t := pp.runqtail
	n := uint32(0)
	// 将 Goroutine 放入本地队列
	for !q.empty() && t-h < uint32(len(pp.runq)) {
		gp := q.pop()
		pp.runq[t%uint32(len(pp.runq))].set(gp)
		t++
		n++
	}
	// 更新剩余 Goroutine 数量
	qsize -= int(n)

	// 随机化调度
	if randomizeScheduler {
		off := func(o uint32) uint32 {
			return (pp.runqtail + o) % uint32(len(pp.runq))
		}
		for i := uint32(1); i < n; i++ {
			j := cheaprandn(i + 1)
			pp.runq[off(i)], pp.runq[off(j)] = pp.runq[off(j)], pp.runq[off(i)]
		}
	}

	// 更新本地队列尾指针
	atomic.StoreRel(&pp.runqtail, t)
	// 将剩余 Goroutine 放入全局队列
	if !q.empty() {
		lock(&sched.lock)
		globrunqputbatch(q, int32(qsize))
		unlock(&sched.lock)
	}
}
```

##### 获取

从本地可运行队列获取一个 goroutine

```go
func runqget(pp *p) (gp *g, inheritTime bool) {
	// 尝试取出 runnext
	next := pp.runnext
	if next != 0 && pp.runnext.cas(next, 0) {
		return next.ptr(), true
	}

	// 从本地队列获取 G
	for {
		h := atomic.LoadAcq(&pp.runqhead)
		t := pp.runqtail
		if t == h {
			return nil, false
		}
		gp := pp.runq[h%uint32(len(pp.runq))].ptr()
		if atomic.CasRel(&pp.runqhead, h, h+1) {
			return gp, false
		}
	}
}
```

##### 窃取

从一个 P 的本地可运行队列中窃取一半的 Goroutine，并放入另一个 P 的本地队列

```go
func runqsteal(pp, p2 *p, stealRunNextG bool) *g {
	t := pp.runqtail
	// 尝试窃取 Goroutine
	n := runqgrab(p2, &pp.runq, t, stealRunNextG)
	if n == 0 {
		return nil
	}
	n--
	gp := pp.runq[(t+n)%uint32(len(pp.runq))].ptr()
	if n == 0 {
		return gp
	}
	h := atomic.LoadAcq(&pp.runqhead)
	if t-h+n >= uint32(len(pp.runq)) {
		throw("runqsteal: runq overflow")
	}
	// 存储窃取的 Goroutine
	atomic.StoreRel(&pp.runqtail, t+n)
	return gp
}
```

从 P 的本地可运行队列中批量获取 Goroutine，并存入 batch 这个环形缓冲区。

```go
func runqgrab(pp *p, batch *[256]guintptr, batchHead uint32, stealRunNextG bool) uint32 {
	for {
		h := atomic.LoadAcq(&pp.runqhead)
		t := atomic.LoadAcq(&pp.runqtail)
		// 可用的 G 数量
		n := t - h
		// 获取一半的 G
		n = n - n/2
		// 如果没有 G 可获取，尝试窃取 runnext
		if n == 0 {
			if stealRunNextG {
				if next := pp.runnext; next != 0 {
					if pp.status == _Prunning {
						// 如果 `P` 正在运行，休眠一段时间，避免 G 频繁切换导致抖动
						if !osHasLowResTimer {
							usleep(3)
						} else {
							osyield()
						}
					}
					// 尝试获取 runnext
					if !pp.runnext.cas(next, 0) {
						continue
					}
					batch[batchHead%uint32(len(batch))] = next
					return 1
				}
			}
			return 0
		}
		// 防止 h 和 t 不一致
		if n > uint32(len(pp.runq)/2) {
			continue
		}
		// 批量获取 G
		for i := uint32(0); i < n; i++ {
			g := pp.runq[(h+i)%uint32(len(pp.runq))]
			batch[(batchHead+i)%uint32(len(batch))] = g
		}
		// 更新本地队列
		if atomic.CasRel(&pp.runqhead, h, h+n) {
			return n
		}
	}
}
```