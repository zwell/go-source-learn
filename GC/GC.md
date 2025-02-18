### 介绍


#### 主要特点

- 并发执行：GC 与用户程序（Mutator）并发运行，减少停顿时间。

- 类型准确（Precise）：GC 能够精确识别指针和非指针，避免误回收。

- 并行标记：多个 GC 线程可以并行工作。

- 写屏障（Write Barrier）：在并发标记阶段，写屏障用于确保指针更新的正确性。

- 非分代（Non-generational）：Go 的 GC 不区分年轻代和老年代。

- 非压缩（Non-compacting）：GC 不会移动对象来压缩内存。

#### 垃圾回收算法的步骤

1. ##### 清扫终止（Sweep Termination）

   - Stop the World（STW）：

       暂停所有 Goroutine，确保所有 P（Processor）都到达 GC 安全点。

   - 清扫未清扫的 Span：

       如果本次 GC 是被强制触发的，可能会有未清扫的 Span，此时需要完成清扫。

2. ##### 标记阶段（Mark Phase）

   - 准备标记：

       将 GC 阶段设置为 _GCmark，启用写屏障和 Mutator 辅助标记。
    
       将所有根标记任务加入队列。

   - 启动世界（Start the World）：

       恢复 Goroutine 的执行。
    
       标记工作由调度器启动的标记线程和分配时的 Mutator 辅助完成。
    
       新分配的对象会立即被标记为黑色。

   - 根标记：

       扫描所有栈、全局变量和堆外运行时数据结构中的指针。
    
       扫描栈时会暂停 Goroutine，标记栈上的指针，然后恢复 Goroutine。

   - 标记灰色对象：

       从工作队列中取出灰色对象，将其标记为黑色，并扫描其引用的对象。

   - 分布式终止检测：

       使用分布式终止算法检测是否完成所有根标记和灰色对象标记。
    
       完成后，进入标记终止阶段。

3. ##### 标记终止（Mark Termination）

   - Stop the World：

       再次暂停所有 Goroutine。

   - 设置 GC 阶段：

       将 GC 阶段设置为 _GCmarktermination，禁用标记线程和 Mutator 辅助。

   - 清理工作：

       刷新 Mcache 等清理操作。

4. #### 清扫阶段（Sweep Phase）

   - 准备清扫：
    
       将 GC 阶段设置为 _GCoff，设置清扫状态并禁用写屏障。

   - 启动世界：

       恢复 Goroutine 的执行。
    
       新分配的对象为白色，必要时在分配前清扫 Span。

   - 并发清扫：

       在后台和分配时进行并发清扫。

5. ##### 触发下一次 GC

    当分配达到一定阈值时，重新开始上述步骤。

### GC 状态

- _GCoff 没有运行
- __GCmark 标记阶段
- _GCmarktermination 终止阶段

### 主要方法

##### 初始化

```go
func gcinit() {
	// 标记垃圾回收器在第一次运行时不进行 sweep（清扫）操作。
	sweep.active.state.Store(sweepDrainedMask)
	
	// 初始化 gcController
	gcController.init(readGOGC(), readGOMEMLIMIT())

	work.startSema = 1
	work.markDoneSema = 1
	lockInit(&work.sweepWaiters.lock, lockRankSweepWaiters)
	lockInit(&work.assistQueue.lock, lockRankAssistQueue)
	lockInit(&work.strongFromWeak.lock, lockRankStrongFromWeakQueue)
	lockInit(&work.wbufSpans.lock, lockRankWbufSpans)
}
```

##### 启用

go 进程启动时启用 GC

```go
func gcenable() {
	// 创建了一个带缓冲的通道 c，容量为 2。这个通道用于同步后台清扫器（sweeper）和回收器（scavenger）的初始化。
	c := make(chan int, 2)
	// 启动一个后台清扫器的 goroutine。
	go bgsweep(c)
	// 启动一个后台回收器的 goroutine。
	go bgscavenge(c)
	// 阻塞主线程，直到从通道 c 接收到两个信号。
	<-c
	<-c
	// 垃圾回收（GC）已经启用。
	memstats.enablegc = true 
}
```

##### GC

```go
func GC() {
	// 等待当前 GC 周期完成
	n := work.cycles.Load()
	gcWaitOnMark(n)
	
	// 启动新周期
	gcStart(gcTrigger{kind: gcTriggerCycle, n: n + 1})

	// 等待新周期的标记终止
	gcWaitOnMark(n + 1)
	
	// 主动清理未标记的内存块
	for work.cycles.Load() == n+1 && sweepone() != ^uintptr(0) {
		// 让出执行权
		Gosched()
	}

	// 等待并发清扫完成
	for work.cycles.Load() == n+1 && !isSweepDone() {
		Gosched()
	}

    // 发布新的堆配置文件
	mp := acquirem()
	cycle := work.cycles.Load()
	if cycle == n+1 || (gcphase == _GCmark && cycle == n+2) {
		mProf_PostSweep()
	}
	releasem(mp)
}
```

开启 GC

```go
// 垃圾回收
var gcsema uint32 = 1

// STW
var worldsema uint32 = 1
```

```go
gcStart(trigger gcTrigger) {
	// 清理未标记的内存块
	for trigger.test() && sweepone() != ^uintptr(0) {
	}

	// 开始标记阶段
	semacquire(&work.startSema)
	
	// 根据调试模式升级垃圾回收模式。默认情况下，垃圾回收以并发模式运行。
	mode := gcBackgroundMode
	if debug.gcstoptheworld == 1 {
		mode = gcForceMode
	} else if debug.gcstoptheworld == 2 {
		mode = gcForceBlockMode
	}

	// 获取 GC 和 STW 的权限
	semacquire(&gcsema)
	semacquire(&worldsema)
	
	// 判断是否是用户强制触发的 GC。
	work.userForced = trigger.kind == gcTriggerCycle
	
	// 检查所有 P 的 mcache 是否被正确刷新。
	for _, p := range allp {
		// 比较 p.mcache.flushGen（P 的刷新代）和 mheap_.sweepgen（当前全局清扫代）。
		if fg := p.mcache.flushGen.Load(); fg != mheap_.sweepgen {
			println("runtime: p", p.id, "flushGen", fg, "!= sweepgen", mheap_.sweepgen)
			throw("p mcache not flushed")
		}
	}

	// 启动后台标记工作线程。
	gcBgMarkStartWorkers()

	// 重置标记状态。
	systemstack(gcResetMarkState)

	// 计算 STW 的处理器数量。
	work.stwprocs, work.maxprocs = gomaxprocs, gomaxprocs
	if work.stwprocs > ncpu {
		work.stwprocs = ncpu
	}
	// 记录当前堆内存状态。
	work.heap0 = gcController.heapLive.Load()
	work.pauseNS = 0
	work.mode = mode

	// 停止“世界”。
	now := nanotime()
	work.tSweepTerm = now
	var stw worldStop
	systemstack(func() {
		stw = stopTheWorldWithSema(stwGCSweepTerm)
	})

	// 累积暂停时间统计。
	work.cpuStats.accumulateGCPauseTime(stw.stoppingCPUTime, 1)

	// 完成清扫工作
	systemstack(func() {
		finishsweep_m()
	})
	
	// 清除缓存池中的对象。
	clearpools()

	// 增加 GC 循环计数。
	work.cycles.Add(1)
	
	// 启动垃圾回收的一个新周期。
	gcController.startCycle(now, int(gomaxprocs), trigger)

	// 通知 CPU 限制器 GC 已经开始。
	gcCPULimiter.startGCTransition(true, now)
	
	// 禁止用户 Goroutine 调度（并发模式下）。
	if mode != gcBackgroundMode {
		schedEnableUser(false)
	}

	// GC 进入标记阶段
	setGCPhase(_GCmark)

	// 准备后台并发标记
	gcBgMarkPrepare()
	// 准备标记根对象
	gcMarkRootPrepare()
	
	// 标记 Tiny 对象
	gcMarkTinyAllocs()

	// 允许对象被标记为“黑色”
	atomic.Store(&gcBlackenEnabled, 1)
	
	mp = acquirem()
	
	// 更新 CPU 统计数据中的暂停时间。
	work.cpuStats.accumulateGCPauseTime(nanotime()-stw.finishedStopping, work.maxprocs)

	// 开始并发标记阶段。
	systemstack(func() {
        // start world
		now = startTheWorldWithSema(0, stw)
		work.pauseNS += now - stw.startedStopping
		work.tMark = now

		// 标记并发标记阶段完成，释放 CPU 限制器。
		gcCPULimiter.finishGCTransition(now)
	})
	
	// 释放 STW 权限
	semrelease(&worldsema)

	releasem(mp)
	
	// 非并发模式，让出执行权，让其他 Goroutine 有机会运行
	if mode != gcBackgroundMode {
		Gosched()
	}

	// 标记阶段结束
	semrelease(&work.startSema)
}
```

##### 开启，停止”世界“

停止“世界”

```go
func stopTheWorld(reason stwReason) worldStop {
	semacquire(&worldsema)
	gp := getg()
    // 设置不可抢占
	gp.m.preemptoff = reason.String()
	systemstack(func() {
		// 切换当前 G 状态
		casGToWaitingForGC(gp, _Grunning, waitReasonStoppingTheWorld)
		// 停止
		stopTheWorldContext = stopTheWorldWithSema(reason)
		// 恢复 G 状态
		casgstatus(gp, _Gwaiting, _Grunning)
	})
	return stopTheWorldContext
}

func stopTheWorldWithSema(reason stwReason) worldStop {
	gp := getg()
	
	// 确保当前线程没有持有锁
	if gp.m.locks > 0 {
		throw("stopTheWorld: holding locks")
	}

	// 初始化调度器状态
	lock(&sched.lock)
	start := nanotime()
	// 设置全局 GC 状态
	sched.stopwait = gomaxprocs
	// 标记调度器为 gcwaiting 状态，表示正在等待 GC
	sched.gcwaiting.Store(true)
	// 抢占 Goroutines
	preemptall()
	// 暂停当前 P
	gp.m.p.ptr().status = _Pgcstop
	gp.m.p.ptr().gcStopTime = start
	// 更新 sched.stopwait，表示当前处理器已完成暂停
	sched.stopwait--
	// 尝试暂停其他处于 _Psyscall（即当前处于系统调用中的 P） 状态的处理器
	for _, pp := range allp {
		s := pp.status
		if s == _Psyscall && atomic.Cas(&pp.status, s, _Pgcstop) {
			pp.syscalltick++
			pp.gcStopTime = nanotime()
			sched.stopwait--
		}
	}

	// 停止空闲的处理器
	now := nanotime()
	for {
		// 从空闲列表获取处理器
		pp, _ := pidleget(now)
		if pp == nil {
			break
		}
		pp.status = _Pgcstop
		pp.gcStopTime = nanotime()
		sched.stopwait--
	}
	// 判断是否需要等待其他处理器
	wait := sched.stopwait > 0
	unlock(&sched.lock)

	// 等待其余处理器自愿停止
	if wait {
		for {
			// 等待剩余的 P 停止
			if notetsleep(&sched.stopnote, 100*1000) {
				// 如果条件变量被唤醒（所有 P 已停止），则退出循环
				noteclear(&sched.stopnote)
				break
			}
			// 如果超时（100 微秒内未完成），则调用 preemptall，再次触发抢占逻辑，确保没有 P 持续运行。
			preemptall()
		}
	}

	// 记录时间消耗
	finish := nanotime()
	startTime := finish - start
	if reason.isGC() {
		sched.stwStoppingTimeGC.record(startTime)
	} else {
		sched.stwStoppingTimeOther.record(startTime)
	}
	
	// 确保所有处理器 (P) 确实停止
	stoppingCPUTime := int64(0)
	bad := ""
	if sched.stopwait != 0 {
		bad = "stopTheWorld: not stopped (stopwait != 0)"
	} else {
		// 检查所有的 P 是否符合期望
		for _, pp := range allp {
			if pp.status != _Pgcstop {
				bad = "stopTheWorld: not stopped (status != _Pgcstop)"
			}
			if pp.gcStopTime == 0 && bad == "" {
				bad = "stopTheWorld: broken CPU time accounting"
			}
			stoppingCPUTime += finish - pp.gcStopTime
			pp.gcStopTime = 0
		}
	}
	// 处理极端情况
	if freezing.Load() {
		// 如果某个线程因信号处理器中的异常正在引发 panic，可能导致状态检查失败。
		// 通过死锁模拟 (lock(&deadlock) 两次调用) 强制线程停止。
		lock(&deadlock)
		lock(&deadlock)
	}
	if bad != "" {
		throw(bad)
	}

	// 已停止
	worldStopped()

	return worldStop{
		reason:           reason,
		startedStopping:  start,
		finishedStopping: finish,
		stoppingCPUTime:  stoppingCPUTime,
	}
}
```

开启“世界”

```go
func startTheWorldWithSema(now int64, w worldStop) int64 {
	assertWorldStopped()

	mp := acquirem() // disable preemption because it can be holding p in a local var
	// 网络轮询和任务注入
	if netpollinited() {
		list, delta := netpoll(0) // non-blocking
		injectglist(&list)
		netpollAdjustWaiters(delta)
	}
	lock(&sched.lock)

	// 调整处理器数量
	procs := gomaxprocs
	if newprocs != 0 {
		procs = newprocs
		newprocs = 0
	}
	// 分配或回收逻辑处理器
	p1 := procresize(procs)
	// 恢复调度状态
	sched.gcwaiting.Store(false)
	if sched.sysmonwait.Load() {
		sched.sysmonwait.Store(false)
		notewakeup(&sched.sysmonnote)
	}
	unlock(&sched.lock)

	// 已启动
	worldStarted()

	// 遍历所有待激活的 P
	for p1 != nil {
		p := p1
		p1 = p1.link.ptr()
		if p.m != 0 {
			// 如果 P 已绑定一个 M（线程），则唤醒对应的线程。
			mp := p.m.ptr()
			p.m = 0
			if mp.nextp != 0 {
				throw("startTheWorld: inconsistent mp->nextp")
			}
			mp.nextp.set(p)
			notewakeup(&mp.park)
		} else {
			// 如果未绑定 M，则创建一个新的线程（newm）以运行 P。
			newm(nil, p, -1)
		}
	}

	// 记录时间和状态
	if now == 0 {
		now = nanotime()
	}
	totalTime := now - w.startedStopping
	if w.reason.isGC() {
		sched.stwTotalTimeGC.record(totalTime)
	} else {
		sched.stwTotalTimeOther.record(totalTime)
	}
	
	// 唤醒额外的处理器
	wakep()

	releasem(mp)

	return now
}
```


##### 辅助方法

判断垃圾回收触发条件是否满足

```go
func (t gcTrigger) test() bool {
	// memstats.enablegc 如果垃圾回收被禁用，则直接返回 false。
	// panicking.Load() != 0 如果当前程序正在 panic 状态，GC 触发被禁用，返回 false。
	// gcphase != _GCoff 如果当前 GC 不在 _GCoff 阶段（即非垃圾回收空闲阶段），返回 false。
	if !memstats.enablegc || panicking.Load() != 0 || gcphase != _GCoff {
		return false
	}
	switch t.kind {
	case gcTriggerHeap:
		// 比较当前的堆内存使用量 heapLive 是否大于等于触发阈值。
		trigger, _ := gcController.trigger()
		return gcController.heapLive.Load() >= trigger
	case gcTriggerTime:
		// 比较时间间隔是否超过强制回收周期 forcegcperiod。
		if gcController.gcPercent.Load() < 0 {
			return false
		}
		lastgc := int64(atomic.Load64(&memstats.last_gc_nanotime))
		return lastgc != 0 && t.now-lastgc > forcegcperiod
	case gcTriggerCycle:
		// 判断目标周期 t.n 是否在当前周期之后。
		return int32(t.n-work.cycles.Load()) > 0
	}
	return true
}
```

准备 GC 标记 Goroutine

```go
func gcBgMarkStartWorkers() {
    // 检查现有 Goroutine 数量
	if gcBgMarkWorkerCount >= gomaxprocs {
		return
	}
	
	mp := acquirem()
	ready := make(chan struct{}, 1)
	releasem(mp)

	for gcBgMarkWorkerCount < gomaxprocs {
		mp := acquirem()
		// 异步启动 Goroutine
		go gcBgMarkWorker(ready)
		releasem(mp)
		
		// 等待启动完成
		<-ready

		gcBgMarkWorkerCount++
	}
}
```

重置标记状态

```go
func gcResetMarkState() {
	// 重置 Goroutine
	forEachG(func(gp *g) {
		gp.gcscandone = false
		gp.gcAssistBytes = 0
	})

    // 重置 heapArena
	lock(&mheap_.lock)
	arenas := mheap_.allArenas
	unlock(&mheap_.lock)
	for _, ai := range arenas {
		ha := mheap_.arenas[ai.l1()][ai.l2()]
		clear(ha.pageMarks[:])
	}

	// 初始化全局标记数据
	work.bytesMarked = 0
	work.initialHeapLive = gcController.heapLive.Load()
}
```
