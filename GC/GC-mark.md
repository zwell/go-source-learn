### 介绍

标记阶段是垃圾回收的核心阶段之一，它的主要任务是遍历堆中的所有对象，标记出所有存活的对象。

标记阶段完成后，未被标记的对象将被视为垃圾，并在后续的清扫阶段中被回收。

#### 标记阶段的目标

1. 标记存活对象：
    - 从根对象（如全局变量、栈上的变量、寄存器等）出发，遍历所有可达的对象，并将其标记为存活。
    - 未被标记的对象将被视为垃圾。

2. 构建对象图：
    - 标记阶段会构建一个对象图，表示堆中所有对象的引用关系。
    - 这个对象图用于确定哪些对象是存活的，哪些是可以回收的。

3. 支持并发标记：
    - Go 的垃圾回收器是并发的，标记阶段可以与用户程序（Mutator）同时运行。
    - 通过并发标记，减少垃圾回收对用户程序的停顿时间。

#### 标记阶段的步骤

1. 初始化标记：
    在标记阶段开始时，垃圾回收器会初始化标记状态，例如：
    - 清空标记位图（mark bits）。
    - 设置 GC 阶段为 _GCmark。

2. 根对象扫描：
    - 暂停程序 STW
    - 从根对象（Roots）开始扫描，根对象包括：
        - 全局变量。
        - 每个 Goroutine 的栈上的变量。
        - 寄存器中的变量。
    - 将这些根对象加入标记队列（Mark Queue）。
    - 启动程序

3. 并发标记：
    - 标记工作由多个 Goroutine 并发执行，包括：
        - 后台标记 Goroutine：专门负责标记工作。
        - Mutator Assists：用户程序在分配内存时，如果发现 GC 需要帮助，会主动参与标记工作。
    - 标记 Goroutine 从标记队列中取出对象，遍历其引用的子对象，并将子对象加入标记队列。

4. 标记终止（Mark Termination）：
    - 当标记队列为空时，标记阶段进入终止阶段。
    - 暂停程序 STW
    - 在标记终止阶段，垃圾回收器会：
        - 重新扫描根对象，确保没有遗漏的对象。
        - 处理在并发标记期间新分配的对象（通过写屏障记录）。
    - 启动程序

5. 标记完成：
    - 标记阶段完成后，所有存活对象都被标记。
    - GC 阶段从 _GCmark 切换到 _GCmarktermination，然后进入清扫阶段。

#### 并发标记的实现

Go 的垃圾回收器通过以下机制实现并发标记：

1. 写屏障（Write Barrier）：
    - 在并发标记期间，用户程序可能会修改对象的引用关系。
    - 写屏障用于捕获这些修改，确保标记阶段的正确性。
    - 例如，当用户程序将一个指针写入某个对象时，写屏障会将该指针指向的对象加入标记队列。

2. 三色标记法：
    - Go 的垃圾回收器使用三色标记法（Tri-Color Marking）来实现并发标记。
    - 对象分为三种颜色：
        - 白色：未标记的对象（可能是垃圾）。
        - 灰色：已标记但子对象未完全扫描的对象。
        - 黑色：已标记且子对象已完全扫描的对象。
    - 标记阶段的目标是将所有存活对象标记为黑色。

3. 标记队列（Mark Queue）：
    - 标记队列用于存储待扫描的对象（灰色对象）。
    - 标记 Goroutine 从队列中取出对象，扫描其子对象，并将其标记为黑色。

4. Mutator Assists：
    - 用户程序在分配内存时，如果发现 GC 需要帮助，会主动参与标记工作。
    - 这是为了避免 GC 滞后，影响用户程序的性能。

#### 标记工作的模式

```go
type gcMarkWorkerMode int

const (
	// 当前没有工作需要执行。
	gcMarkWorkerNotWorker gcMarkWorkerMode = iota

	// 专用工作者模式，确保标记任务独占执行。
	gcMarkWorkerDedicatedMode
	
	// 分数工作者模式，调整工作量以便更精准地控制背景回收的利用率。
	gcMarkWorkerFractionalMode

	// 空闲工作者模式，当没有其他任务时运行标记工作。
	gcMarkWorkerIdleMode
)
```

### 主要方法

##### gcMarkRootPrepare 为根对象扫描做准备

根对象是垃圾回收的起点，包括全局变量、栈上的变量、寄存器中的变量以及一些特殊对象（如带有终结器的对象）。

将根对象分配到工作队列中，以便后续的并发标记阶段可以高效地扫描它们。

```go
func gcMarkRootPrepare() {
	// 确保当前处于 STW 状态
	assertWorldStopped()

	// 计算根块数量
	nBlocks := func(bytes uintptr) int {
		return int(divRoundUp(bytes, rootBlockBytes))
	}

	// 初始化根对象计数器
	work.nDataRoots = 0
	work.nBSSRoots = 0

	// 遍历所有活跃的模块，计算每个模块的 data 和 BSS 段的根块数量
	// 活跃模块（Active Modules） 指的是当前程序中所有加载的模块（Module），这些模块包含了程序的代码、全局变量、类型信息等。
    for _, datap := range activeModules() {
		nDataRoots := nBlocks(datap.edata - datap.data)
		if nDataRoots > work.nDataRoots {
			work.nDataRoots = nDataRoots
		}

		nBSSRoots := nBlocks(datap.ebss - datap.bss)
		if nBSSRoots > work.nBSSRoots {
			work.nBSSRoots = nBSSRoots
		}
	}

   // 准备扫描堆中的 span 根对象
   // 只有在 span 中包含特殊对象（如带有终结器的对象）时，才需要扫描这些 Span
	mheap_.markArenas = mheap_.allArenas[:len(mheap_.allArenas):len(mheap_.allArenas)]
	work.nSpanRoots = len(mheap_.markArenas) * (pagesPerArena / pagesPerSpanRoot)
	
	// 扫描栈
	work.stackRoots = allGsSnapshot() // allGsSnapshot 获取所有 Goroutine 的快照
	work.nStackRoots = len(work.stackRoots)

	// 初始化工作队列
	work.markrootNext = 0
	work.markrootJobs = uint32(fixedRootCount + work.nDataRoots + work.nBSSRoots + work.nSpanRoots + work.nStackRoots)

	// 计算每种根类型的基准索引
	work.baseData = uint32(fixedRootCount)
	work.baseBSS = work.baseData + uint32(work.nDataRoots)
	work.baseSpans = work.baseBSS + uint32(work.nBSSRoots)
	work.baseStacks = work.baseSpans + uint32(work.nSpanRoots)
	work.baseEnd = work.baseStacks + uint32(work.nStackRoots)
}
```

##### 后台标记

启动后台标记工作 Goroutine

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

管理后台标记工作 Goroutine（gcBgMarkWorker）

它是 gcBgMarkWorkerPool 中的一个条目，用于指向一个单独的后台标记工作 Goroutine。

```go
type gcBgMarkWorkerNode struct {
	// 一个无锁栈（lock-free stack）的节点。
	node lfnode

	// 关联的 G
	gp guintptr
	
	// 关联的 M
	m muintptr
}
```

***注：用于标记的 Goroutine（gcBgMarkWorker）本身也需要被标记***

```go
func gcBgMarkWorker(ready chan struct{}) {
	gp := getg()
	
	// 创建 gcBgMarkWorkerNode
	gp.m.preemptoff = "GC worker init"
	node := new(gcBgMarkWorkerNode)
	gp.m.preemptoff = ""

	// 设置 Goroutine 和线程信息
	node.gp.set(gp)
	node.m.set(acquirem())

	// 通知启动完成
	ready <- struct{}{}
	
	for {
		// 将当前 Goroutine 暂停，等待唤醒。
		gopark(func(g *g, nodep unsafe.Pointer) bool {
			node := (*gcBgMarkWorkerNode)(nodep)

			if mp := node.m.ptr(); mp != nil {
				// 检查当前 Goroutine 是否有绑定的线程（M）,如果存在，释放该线程资源。
				releasem(mp)
			}

			// 将当前 Goroutine 的节点（gcBgMarkWorkerNode）推入后台标记任务池（gcBgMarkWorkerPool）。
			gcBgMarkWorkerPool.push(&node.node)

			return true
		}, unsafe.Pointer(node), waitReasonGCWorkerIdle, traceBlockSystemGoroutine, 0)

		// 如果被唤醒，继续执行后面的。
		
		// 禁止抢占。
		node.m.set(acquirem())
		pp := gp.m.p.ptr()

		// 空闲工作模式下，定义限速机制
		startTime := nanotime()
		pp.gcMarkWorkerStartTime = startTime
		var trackLimiterEvent bool
		if pp.gcMarkWorkerMode == gcMarkWorkerIdleMode {
			trackLimiterEvent = pp.limiterEvent.start(limiterEventIdleMarkWork, startTime)
		}

        // 更新等待计数，该 Goroutine 准备开始工作
        decnwait := atomic.Xadd(&work.nwait, -1)

        // 使用系统栈执行标记工作
		systemstack(func() {
            // 标记当前 Goroutine 可被抢占，以便其栈可以被扫描（避免两个标记工作 Goroutine 相互扫描时发生死锁。）
			casGToWaitingForGC(gp, _Grunning, waitReasonGCWorkerActive)
			// 不同的标记工作模式处理
			switch pp.gcMarkWorkerMode {
			default:
				throw("gcBgMarkWorker: unexpected gcMarkWorkerMode")
			case gcMarkWorkerDedicatedMode:
				// 专用标记模式，工作线程全力执行标记任务。
				gcDrainMarkWorkerDedicated(&pp.gcw, true)
				if gp.preempt {
					// 如果被抢占，清空运行队列，方便调度其他任务。
					if drainQ, n := runqdrain(pp); n > 0 {
						lock(&sched.lock)
						globrunqputbatch(&drainQ, int32(n))
						unlock(&sched.lock)
					}
				}
				// 不允许抢占
				gcDrainMarkWorkerDedicated(&pp.gcw, false)
			case gcMarkWorkerFractionalMode:
				// 分时标记模式，工作线程根据特定比例执行标记任务。
				gcDrainMarkWorkerFractional(&pp.gcw)
			case gcMarkWorkerIdleMode:
				// 空闲标记模式，仅在系统空闲时执行。
				gcDrainMarkWorkerIdle(&pp.gcw)
			}
			// 标记完成后状态恢复。
			casgstatus(gp, _Gwaiting, _Grunning)
		})
		
		// 更新等待计数，该 Goroutine 完成标记工作
		incnwait := atomic.Xadd(&work.nwait, +1)
		
		// 清除工作模式
		pp.gcMarkWorkerMode = gcMarkWorkerNotWorker
		
		// 所有工作者都完成了工作，并且没有标记任务（最后一个工作者完成时触发）
		if incnwait == work.nproc && !gcMarkWorkAvailable(nil) {
            // 释放 M 并允许抢占，以便当前 Goroutine 可以像普通 Goroutine 一样被调度。
			releasem(node.m.ptr())
			node.m.set(nil)

			// 标记阶段结束
			gcMarkDone()
		}
	}
}
```

##### 标记

根据不同的工作模式，执行标记工作

```go
// 专用标记模式，不允许抢占
func gcDrainMarkWorkerDedicated(gcw *gcWork, untilPreempt bool) {
	flags := gcDrainFlushBgCredit
	if untilPreempt {
		flags |= gcDrainUntilPreempt
	}
	gcDrain(gcw, flags)
}

// 分时标记模式，工作线程根据特定比例执行标记任务。
func gcDrainMarkWorkerFractional(gcw *gcWork) {
	gcDrain(gcw, gcDrainFractional|gcDrainUntilPreempt|gcDrainFlushBgCredit)
}

// 空闲标记模式，仅在系统空闲时执行。
func gcDrainMarkWorkerIdle(gcw *gcWork) {
gcDrain(gcw, gcDrainIdle|gcDrainUntilPreempt|gcDrainFlushBgCredit)
}
```

从根集和工作缓冲区扫描灰色对象，并将它们变成黑色，直到没有更多工作可做为止。

```go
func gcDrain(gcw *gcWork, flags gcDrainFlags) {
	gp := getg().m.curg
	pp := gp.m.p.ptr()
	// 是否是可抢占的上下文。
	preemptible := flags&gcDrainUntilPreempt != 0
	// 是否需要刷新后台信用。
	flushBgCredit := flags&gcDrainFlushBgCredit != 0
	// 是否处于空闲状态。
	idle := flags&gcDrainIdle != 0

	// 初始化扫描工作
	initScanWork := gcw.heapScanWork
	
	// 检查工作量，并决定是否进行自我抢占检查
	checkWork := int64(1<<63 - 1)
	var check func() bool
	if flags&(gcDrainIdle|gcDrainFractional) != 0 {
		checkWork = initScanWork + drainCheckThreshold
		if idle {
			check = pollWork
		} else if flags&gcDrainFractional != 0 {
			check = pollFractionalWorkerExit
		}
	}

	// 根标记任务执行检查
	if work.markrootNext < work.markrootJobs {
		// 自我抢占和其他停止条件
		for !(gp.preempt && (preemptible || sched.gcwaiting.Load() || pp.runSafePointFn != 0)) {
			// 原子操作执行根标记任务
			job := atomic.Xadd(&work.markrootNext, +1) - 1
			if job >= work.markrootJobs {
				break
			}
			markroot(gcw, job, flushBgCredit)
			// 是否需要提前退出
			if check != nil && check() {
				goto done
			}
		}
	}
	
	// 抢占检查
	// gp.preempt：检查当前 goroutine 是否标记为可抢占。
	// preemptible：是否可以进行抢占。
	// sched.gcwaiting.Load()：是否处于 GC 等待状态。
	// pp.runSafePointFn != 0：是否有正在运行的安全点函数。
	for !(gp.preempt && (preemptible || sched.gcwaiting.Load() || pp.runSafePointFn != 0)) {
		// 保持全局队列中的工作项
		if work.full == 0 {
			gcw.balance()
		}

		// 获取工作项
		b := gcw.tryGetFast()
		if b == 0 {
			b = gcw.tryGet()
			if b == 0 {
				// 如果仍然没有工作项，调用 wbBufFlush() 刷新写屏障缓冲区，可能会创建新的工作项。
				wbBufFlush()
				b = gcw.tryGet()
			}
		}
		// 无法获取到工作项
		if b == 0 {
			break
		}
		// 扫描对象
		scanobject(b, gcw)
		
		// 将局部扫描工作积分到全局
		// 如果本地扫描工作（gcw.heapScanWork）达到或超过 gcCreditSlack，将本地的工作积分到全局。
		if gcw.heapScanWork >= gcCreditSlack {
			// 将本地的扫描工作累加到全局工作进度中。
			gcController.heapScanWork.Add(gcw.heapScanWork)
			if flushBgCredit {
				gcFlushBgCredit(gcw.heapScanWork - initScanWork)
				initScanWork = 0
			}
			checkWork -= gcw.heapScanWork
			gcw.heapScanWork = 0

			// 检查是否需要继续工作
			if checkWork <= 0 {
				checkWork += drainCheckThreshold
				if check != nil && check() {
					break
				}
			}
		}
	}

done:
	// 垃圾回收工作后处理并刷新剩余的扫描工作积分。
	if gcw.heapScanWork > 0 {
		// 将剩余的扫描工作积分加入全局工作进度
		gcController.heapScanWork.Add(gcw.heapScanWork)
		// 刷新背景工作积分
		if flushBgCredit {
			gcFlushBgCredit(gcw.heapScanWork - initScanWork)
		}
		// 重置本地扫描工作积分
		gcw.heapScanWork = 0
	}
}
```

### GC标记结束 gcMarkDone

```go
func gcMarkDone() {
	semacquire(&work.markDoneSema)

top:
	// 检查过渡条件:
	// 当前必须处于 GCmark 阶段。
	// work.nwait == work.nproc：所有处理器（P）都处于等待状态，说明没有线程在处理标记任务。
	// !gcMarkWorkAvailable(nil)：全局工作队列中没有剩余的灰对象需要处理。
	if !(gcphase == _GCmark && work.nwait == work.nproc && !gcMarkWorkAvailable(nil)) {
		semrelease(&work.markDoneSema)
		return
	}
	
	// 获取 worldsema 信号量，用于接下来的全局操作。
	semacquire(&worldsema)
	
	// 阻止弱引用的转换
	work.strongFromWeak.block = true

	// 初始化并开始遍历所有处理器
	gcMarkDoneFlushed = 0
	forEachP(waitReasonGCMarkTermination, func(pp *p) {
		// 清空写屏障缓冲区
		wbBufFlush1(pp)
		
		// 清空本地标记工作缓冲区
		pp.gcw.dispose()
		// 检查 flushedWork 标志
		if pp.gcw.flushedWork {
			atomic.Xadd(&gcMarkDoneFlushed, 1)
			pp.gcw.flushedWork = false
		}
	})

	// 决定是否重新检查过渡条件。
	// 如果 gcMarkDoneFlushed 不为 0，表示有新的工作需要处理
	if gcMarkDoneFlushed != 0 {
		semrelease(&worldsema)
		goto top
	}
	
	now := nanotime()
	work.tMarkTerm = now
	// 停止整个世界
	getg().m.preemptoff = "gcing"
	var stw worldStop
	systemstack(func() {
		stw = stopTheWorldWithSema(stwGCMarkTerm)
	})

	// 记录停止时间
	work.cpuStats.accumulateGCPauseTime(stw.stoppingCPUTime, 1)
	
	// 检查并恢复可能遗留的工作
	restart := false
	systemstack(func() {
		for _, p := range allp {
			wbBufFlush1(p)
			// 如果发现某个处理器的本地标记工作队列 gcw 仍然有未处理的灰色对象，需要重新启动标记结束阶段。
			if !p.gcw.empty() {
				restart = true
				break
			}
		}
	})
	// 如果有遗留工作，重新启动标记阶段
	if restart {
		gcDebugMarkDone.restartedDueTo27993 = true

		getg().m.preemptoff = ""
		systemstack(func() {
			// Accumulate the time we were stopped before we had to start again.
			work.cpuStats.accumulateGCPauseTime(nanotime()-stw.finishedStopping, work.maxprocs)

			// 重新启动世界
			now := startTheWorldWithSema(0, stw)
			work.pauseNS += now - stw.startedStopping
		})
		semrelease(&worldsema)
		goto top
	}

	// 重新计算起始栈大小
	gcComputeStartingStackSize()
	
	// 停止所有 GC 辅助标记（assists）和后台标记工作。
	atomic.Store(&gcBlackenEnabled, 0)

	// 通知 GC CPU 限制器，辅助标记（assists）将停止。
	gcCPULimiter.startGCTransition(false, now)
	
	// 唤醒所有因等待辅助标记而阻塞的 Goroutine。
	gcWakeAllAssists()
	
	// 唤醒弱引用的强引用转换
	work.strongFromWeak.block = false
	gcWakeAllStrongFromWeak()
	
	// 释放标记完成的信号量
	semrelease(&work.markDoneSema)
	
	// 重新启用用户 Goroutine
	schedEnableUser(true)
	
	// 结束当前 GC 周期
	gcController.endCycle(now, int(gomaxprocs), work.userForced)

	// 进入终结阶段
	gcMarkTermination(stw)
}
```

### 辅助GC

用户程序在分配内存（mallocgc）时，如果发现 GC 需要帮助，会主动参与标记工作

#### 核心思想

如果程序分配内存过快，GC 线程（后台标记/清扫线程）来不及处理，那么让正在分配内存的 Goroutine 自己承担一部分 GC 工作，以维持回收进度。

在垃圾回收的设计上，Go 的 辅助 GC（Assist GC） 机制本质上是一种 "work-stealing"（分担工作负载） 的策略，用于平衡 GC 线程与应用 Goroutine 之间的负担。

#### 流程

##### 内存分配，扣减助力额度
分配内存时，当 GC 处于标记阶段，更新助力额度

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
   ...
   if gcBlackenEnabled != 0 {
	   deductAssistCredit(size)
   }
   ...
}
```

减少当前 Goroutine (G) 的助力额度 (assist credit)，并根据需要协助垃圾回收 (GC) 处理。

```go
func deductAssistCredit(size uintptr) {
	// 当前运行的 G
	assistG := getg()
	if assistG.m.curg != nil {
		assistG = assistG.m.curg
	}
	
	// 减少助力额度
	assistG.gcAssistBytes -= int64(size)

	// 如果当前 Goroutine 的助力额度变为负数（即超出了其分配的额度），则需要协助 GC
	if assistG.gcAssistBytes < 0 {
		// 当前 Goroutine 协助垃圾回收，以便修正额度不足的情况。
		gcAssistAlloc(assistG)
	}
}
```

##### 辅助标记

用户 Goroutine 的垃圾回收（GC）辅助工作，目的是清偿当前 Goroutine 的助力债务（assist debt），确保垃圾回收工作能够顺利进行。

```go
func gcAssistAlloc(gp *g) {
	// 在系统 Goroutine 的上下文中，不执行 GC 辅助工作。
	if getg() == gp.m.g0 {
		return
	}
	// 不可抢占的情况
	if mp := getg().m; mp.locks > 0 || mp.preemptoff != "" {
		return
	}

	// 解除与同步组的关联
	if gp := getg(); gp.syncGroup != nil {
		sg := gp.syncGroup
		gp.syncGroup = nil
		defer func() {
			gp.syncGroup = sg
		}()
	}
	
	// 跟踪进入标记辅助的状态
	enteredMarkAssistForTracing := false
retry:
	// 是否启用了 GC CPU 限制
	if gcCPULimiter.limiting() {
		// 如果启用了 CPU 限制，跳过辅助工作。
		// 减少垃圾回收在 CPU 时间上的占用。
		return
	}
	// 计算需要执行的 GC 扫描工作量
	assistWorkPerByte := gcController.assistWorkPerByte.Load()
	assistBytesPerWork := gcController.assistBytesPerWork.Load()
	debtBytes := -gp.gcAssistBytes
	// 计算扫描工作量
	scanWork := int64(assistWorkPerByte * float64(debtBytes))
	if scanWork < gcOverAssistWork {
		scanWork = gcOverAssistWork
		debtBytes = int64(assistBytesPerWork * float64(scanWork))
	}
	
	// 背景 GC 的信用表示系统已完成的扫描工作
	bgScanCredit := gcController.bgScanCredit.Load()
	// 当前从背景 GC 信用中窃取的工作量。
	stolen := int64(0)
	if bgScanCredit > 0 {
		if bgScanCredit < scanWork {
			stolen = bgScanCredit
			gp.gcAssistBytes += 1 + int64(assistBytesPerWork*float64(stolen))
		} else {
			stolen = scanWork
			gp.gcAssistBytes += debtBytes
		}
		// 减少背景扫描信用余额
		gcController.bgScanCredit.Add(-stolen)

		// 更新剩余工作量
		scanWork -= stolen

		// 是否还需要工作
		if scanWork == 0 {
			return
		}
	}
	// 如果背景信用不足以完全覆盖所需工作量，进入标记辅助工作状态

	// 执行扫描任务
	systemstack(func() {
		gcAssistAlloc1(gp, scanWork)
	})

	// 检查 GC 标记是否完成
	completed := gp.param != nil
	gp.param = nil
	if completed {
		gcMarkDone()
	}

	// 检查助力债务清偿状态。
	// 如果小于 0，说明未能通过窃取背景信用或完成足够工作量来清偿债务。
	if gp.gcAssistBytes < 0 {
		// 如果被抢占了，则重试
		if gp.preempt {
			Gosched()
			goto retry
		}
		
		// 停止当前 Goroutine 并加入助力队列
		if !gcParkAssist() {
			goto retry
		}
	}
}
```

将当前 Goroutine 放入助力队列中并挂起，等待后台垃圾回收（GC）标记进程完成部分工作或为助力分配新的信用。

```go
func gcParkAssist() bool {
	lock(&work.assistQueue.lock)
	// 标记阶段是否结束
	if atomic.Load(&gcBlackenEnabled) == 0 {
		unlock(&work.assistQueue.lock)
		return true
	}

	// 将 Goroutine 放入助力队列
	gp := getg()
	oldList := work.assistQueue.q
	work.assistQueue.q.pushBack(gp)
	
	// 再次检查背景信用
	if gcController.bgScanCredit.Load() > 0 {
		work.assistQueue.q = oldList
		if oldList.tail != 0 {
			oldList.tail.ptr().schedlink.set(nil)
		}
		unlock(&work.assistQueue.lock)
		return false
	}
	// 挂起当前 Goroutine
	goparkunlock(&work.assistQueue.lock, waitReasonGCAssistWait, traceBlockGCMarkAssist, 2)
	return true
}
```

执行垃圾回收的标记工作，同时确保 Goroutine 助力分配的安全性与效率。

```go
func gcAssistAlloc1(gp *g, scanWork int64) {
	// 清理完成标志
	gp.param = nil

	// 检查垃圾回收是否处于标记状态
	if atomic.Load(&gcBlackenEnabled) == 0 {
		gp.gcAssistBytes = 0
		return
	}

	// 跟踪助力工作的耗时
	startTime := nanotime()
	trackLimiterEvent := gp.m.p.ptr().limiterEvent.start(limiterEventMarkAssist, startTime)

    // 更新等待计数
    decnwait := atomic.Xadd(&work.nwait, -1)

	// 切换 Goroutine 状态到等待标记
	casGToWaitingForGC(gp, _Grunning, waitReasonGCAssistMarking)
	
	// 优先处理本地缓存中的工作
	gcw := &getg().m.p.ptr().gcw
	workDone := gcDrainN(gcw, scanWork)

	// 恢复 Goroutine 的运行状态
	casgstatus(gp, _Gwaiting, _Grunning)
	
	// 更新助力信用与标记工作量
	assistBytesPerWork := gcController.assistBytesPerWork.Load()
	gp.gcAssistBytes += 1 + int64(assistBytesPerWork*float64(workDone))
	
	// 更新等待计数
	incnwait := atomic.Xadd(&work.nwait, +1)
	
	// 检查是否到达标记完成点
	if incnwait == work.nproc && !gcMarkWorkAvailable(nil) {
		gp.param = unsafe.Pointer(gp)
	}
	
	// 记录助力工作的耗时
	now := nanotime()
	duration := now - startTime
	pp := gp.m.p.ptr()
	pp.gcAssistTime += duration
	if trackLimiterEvent {
		pp.limiterEvent.stop(limiterEventMarkAssist, now)
	}
	// 更新 CPU 限制器和全局助力时间
	if pp.gcAssistTime > gcAssistTimeSlack {
		gcController.assistTime.Add(pp.gcAssistTime)
		gcCPULimiter.update(now)
		pp.gcAssistTime = 0
	}
}
```

### 辅助方法

##### 不同工作模式的处理

```go
// 专用标记模式，不允许抢占
func gcDrainMarkWorkerDedicated(gcw *gcWork, untilPreempt bool) {
	flags := gcDrainFlushBgCredit
	if untilPreempt {
		flags |= gcDrainUntilPreempt
	}
	gcDrain(gcw, flags)
}

// 分时标记模式，工作线程根据特定比例执行标记任务。
func gcDrainMarkWorkerFractional(gcw *gcWork) {
	gcDrain(gcw, gcDrainFractional|gcDrainUntilPreempt|gcDrainFlushBgCredit)
}

// 空闲标记模式，仅在系统空闲时执行。
func gcDrainMarkWorkerIdle(gcw *gcWork) {
    gcDrain(gcw, gcDrainIdle|gcDrainUntilPreempt|gcDrainFlushBgCredit)
}
```

##### 是否还有标记任务可执行

```go
func gcMarkWorkAvailable(p *p) bool {
	// 检查本地工作队列
	if p != nil && !p.gcw.empty() {
		return true
	}
	// 检查全局任务队列
	if !work.full.empty() {
		return true
	}
	// 检查根扫描任务
	if work.markrootNext < work.markrootJobs {
		return true
	}
	return false
}
```

### 标记

##### gcDrain

从根集和工作缓冲区扫描灰色对象，并将它们变成黑色（标记已访问），直到没有更多工作可做为止。

```go
func gcDrain(gcw *gcWork, flags gcDrainFlags) {
	if !writeBarrier.enabled {
		throw("gcDrain phase incorrect")
	}
	
	gp := getg().m.curg
	pp := gp.m.p.ptr()
	// 是否是可抢占的上下文。
	preemptible := flags&gcDrainUntilPreempt != 0
	// 是否需要刷新后台信用。
	flushBgCredit := flags&gcDrainFlushBgCredit != 0
	// 是否处于空闲状态。
	idle := flags&gcDrainIdle != 0

	// 初始化扫描工作
	initScanWork := gcw.heapScanWork
	
	// 检查工作量，并决定是否进行自我抢占检查
	checkWork := int64(1<<63 - 1)
	var check func() bool
	if flags&(gcDrainIdle|gcDrainFractional) != 0 {
		checkWork = initScanWork + drainCheckThreshold
		if idle {
			check = pollWork
		} else if flags&gcDrainFractional != 0 {
			check = pollFractionalWorkerExit
		}
	}

	// 根标记任务执行检查
	if work.markrootNext < work.markrootJobs {
		// 自我抢占和其他停止条件
		for !(gp.preempt && (preemptible || sched.gcwaiting.Load() || pp.runSafePointFn != 0)) {
			// 原子操作执行根标记任务
			job := atomic.Xadd(&work.markrootNext, +1) - 1
			if job >= work.markrootJobs {
				break
			}
			markroot(gcw, job, flushBgCredit)
			// 是否需要提前退出
			if check != nil && check() {
				goto done
			}
		}
	}

	// 抢占检查
	// gp.preempt：检查当前 goroutine 是否标记为可抢占。
	// preemptible：是否可以进行抢占。
	// sched.gcwaiting.Load()：是否处于 GC 等待状态。
	// pp.runSafePointFn != 0：是否有正在运行的安全点函数。
	for !(gp.preempt && (preemptible || sched.gcwaiting.Load() || pp.runSafePointFn != 0)) {
		// 保持全局队列中的工作项
		if work.full == 0 {
			gcw.balance()
		}

		// 获取工作项
		b := gcw.tryGetFast()
		if b == 0 {
			b = gcw.tryGet()
			if b == 0 {
				// 如果仍然没有工作项，调用 wbBufFlush() 刷新写屏障缓冲区，可能会创建新的工作项。
				wbBufFlush()
				b = gcw.tryGet()
			}
		}
		// 无法获取到工作项（b == 0），退出循环。
		if b == 0 {
			break
		}
		// 扫描对象
		scanobject(b, gcw)
		
		// 将局部扫描工作积分到全局
		// 如果本地扫描工作（gcw.heapScanWork）达到或超过 gcCreditSlack，将本地的工作积分到全局。
		if gcw.heapScanWork >= gcCreditSlack {
			// 将本地的扫描工作累加到全局工作进度中。
			gcController.heapScanWork.Add(gcw.heapScanWork)
			if flushBgCredit {
				gcFlushBgCredit(gcw.heapScanWork - initScanWork)
				initScanWork = 0
			}
			checkWork -= gcw.heapScanWork
			gcw.heapScanWork = 0

			// 检查是否需要继续工作
			if checkWork <= 0 {
				checkWork += drainCheckThreshold
				if check != nil && check() {
					break
				}
			}
		}
	}

done:
	// 垃圾回收工作后处理并刷新剩余的扫描工作积分。
	if gcw.heapScanWork > 0 {
		// 将剩余的扫描工作积分加入全局工作进度
		gcController.heapScanWork.Add(gcw.heapScanWork)
		// 刷新背景工作积分
		if flushBgCredit {
			gcFlushBgCredit(gcw.heapScanWork - initScanWork)
		}
		// 重置本地扫描工作积分
		gcw.heapScanWork = 0
	}
}
```

### 扫描对象

##### scanblock

扫描一个内存块并根据指针位图（ptrmask）扫描其中的指针。

```go
func scanblock(b0, n0 uintptr, ptrmask *uint8, gcw *gcWork, stk *stackScanState) {
	b := b0
	n := n0

	// 遍历整个内存块
	for i := uintptr(0); i < n; {
		// 从位图中获取当前内存位置 i 对应的位信息，addb 函数用于根据偏移量计算位图中的正确位置。
		bits := uint32(*addb(ptrmask, i/(goarch.PtrSize*8)))
		// 如果当前没有指针位（即没有指针需要扫描），则跳过当前字节
		if bits == 0 {
			i += goarch.PtrSize * 8
			continue
		}
		// 扫描每个指针(对于每个字节，检查其每个比特位（每个比特位表示一个指针）。)
		for j := 0; j < 8 && i < n; j++ {
			// 检查当前比特位是否为 1，如果是，表示当前位置是一个有效的指针。
			if bits&1 != 0 {
				// 根据当前字节位置，提取出该位置的指针值 p。
				p := *(*uintptr)(unsafe.Pointer(b + i))
				// 如果 p 是有效的非零指针
				if p != 0 {
					// 检查 p 所指向的内存是否属于一个已知的对象。
					// 如果是，则执行 greyobject，将该对象标记为待处理（灰色）。
					if obj, span, objIndex := findObject(p, b, i); obj != 0 {
						greyobject(obj, b, i, span, gcw, objIndex)
					} else if stk != nil && p >= stk.stack.lo && p < stk.stack.hi {
						// 如果 p 是一个栈指针，并且位于栈的有效范围内，则调用 stk.putPtr(p, false) 记录该指针。
						stk.putPtr(p, false)
					}
				}
			}
			// 位移与循环
			bits >>= 1
			i += goarch.PtrSize
		}
	}
}
```

##### scanobject

遍历一个堆对象，查找其中的指针，并将这些指针加入垃圾回收的工作队列

```go
func scanobject(b uintptr, gcw *gcWork) {
	// 预取该对象，优化内存访问
	sys.Prefetch(b)
	
	// 获取对象大小和内存块信息
	s := spanOfUnchecked(b)
	n := s.elemsize
	if n == 0 {
		throw("scanobject n == 0")
	}
	// 检查是否是可扫描对象
	if s.spanclass.noscan() {
		throw("scanobject of a noscan object")
	}

	var tp typePointers
	// 处理大对象
	if n > maxObletBytes {
		// 分割大型对象
		if b == s.base() {
			// 遍历对象的其他部分
			for oblet := b + maxObletBytes; oblet < s.base()+s.elemsize; oblet += maxObletBytes {
				// 将 oblet 添加到工作队列中
				if !gcw.putFast(oblet) {
					gcw.put(oblet)
				}
			}
		}
		
		// 计算 oblet 的大小
		n = s.base() + s.elemsize - b
		n = min(n, maxObletBytes)
		// 获取指针类型并推进指针位置
		tp = s.typePointersOfUnchecked(s.base())
		tp = tp.fastForward(b-tp.addr, b+n)
	} else {
		// 类型指针的处理
		tp = s.typePointersOfUnchecked(b)
	}

	var scanSize uintptr
	// 循环扫描对象内部的指针
	for {
		var addr uintptr
		// 首先尝试快速获取下一个指针地址，如果没有找到（addr == 0），则继续尝试通过 tp.next() 获取指针。
		if tp, addr = tp.nextFast(); addr == 0 {
			// 如果两个方法都没有找到有效的指针（addr == 0），则退出循环。
			if tp, addr = tp.next(b + n); addr == 0 {
				break
			}
		}
		
		// 追踪扫描的最大指针地址
		scanSize = addr - b + goarch.PtrSize
		
		// 提取指针并进行过滤
		obj := *(*uintptr)(unsafe.Pointer(addr))
		
		// 确保 obj 指向的地址在对象的有效范围内
		if obj != 0 && obj-b >= n {
			// 查找指针 obj 指向的对象是否在 Go 堆内
			if obj, span, objIndex := findObject(obj, b, addr-b); obj != 0 {
				// 标记该对象
				greyobject(obj, b, addr-b, span, gcw, objIndex)
			}
		}
	}
	// 更新 GC 工作进度
	gcw.bytesMarked += uint64(n)
	gcw.heapScanWork += int64(scanSize)
}
```

##### greyobject

标记一个对象并将其加入到垃圾回收（GC）工作队列中，供后续的扫描处理。

```go
func greyobject(obj, base, off uintptr, span *mspan, gcw *gcWork, objIndex uintptr) {
	// 获取对象的标记位
	mbits := span.markBitsForIndex(objIndex)

     // 是否已标记
     if mbits.isMarked() {
         return
     }
     // 标记对象
     mbits.setMarked()

     // 如果对象所在的内存页还未标记，则更新该内存页的标记状态，确保其在垃圾回收过程中被考虑。
     arena, pageIdx, pageMask := pageIndexOf(span.base())
     if arena.pageMarks[pageIdx]&pageMask == 0 {
         atomic.Or8(&arena.pageMarks[pageIdx], pageMask)
     }
	 
     // 如果该对象属于 noscan 类（不需要扫描的对象），则直接将其标记为黑色，而跳过灰色扫描。
     if span.spanclass.noscan() {
         gcw.bytesMarked += uint64(span.elemsize)
         return
     }
	 
	// 预取并将对象加入队列
	sys.Prefetch(obj)
	if !gcw.putFast(obj) {
		gcw.put(obj)
	}
}
```

##### scanstack

扫描一个 goroutine 的栈，以标记栈上所有指向堆对象的指针。

```go
func scanstack(gp *g, gcw *gcWork) int64 {
	// 防止自我扫描
	if gp == getg() {
		throw("can't scan our own stack")
	}
	
	// 计算需要扫描的栈部分的大小
	var sp uintptr
	if gp.syscallsp != 0 {
		// 如果在系统调用中，使用 syscallsp 作为栈指针。
		sp = gp.syscallsp
	} else {
		// 否则使用调度保存的栈指针。
		sp = gp.sched.sp
	}
	scannedSize := gp.stack.hi - sp
	
	// 更新统计数据
	p := getg().m.p.ptr()
	p.scannedStackSize += uint64(scannedSize)
	p.scannedStacks++

	// 判断是否可以安全地收缩栈。
	if isShrinkStackSafe(gp) {
		// 收缩栈。
		shrinkstack(gp)
	} else {
		// 在下一个安全点进行栈收缩。
		gp.preemptShrink = true
	}

	// 栈扫描状态初始化
	var state stackScanState
	state.stack = gp.stack
	
	// 扫描保存的上下文指针
	// 这相当于扫描一个“栈外”保存的指针值。
	if gp.sched.ctxt != nil {
		scanblock(uintptr(unsafe.Pointer(&gp.sched.ctxt)), goarch.PtrSize, &oneptrmask[0], gcw, &state)
	}

	// 扫描栈帧
	var u unwinder
	for u.init(gp, 0); u.valid(); u.next() {
		scanframeworker(&u.frame, &state, gcw)
	}

	// 扫描 defer 链表
	for d := gp._defer; d != nil; d = d.link {
		// 扫描 d.fn：defer 可能包含闭包（closure），闭包可能存储在栈上，需要扫描其指针。
		if d.fn != nil {
			scanblock(uintptr(unsafe.Pointer(&d.fn)), goarch.PtrSize, &oneptrmask[0], gcw, &state)
		}
		// 扫描 d.link：这是 defer 的链表指针，可能指向堆分配的 defer，需要标记。
		if d.link != nil {
			scanblock(uintptr(unsafe.Pointer(&d.link)), goarch.PtrSize, &oneptrmask[0], gcw, &state)
		}
		// 扫描堆分配的 defer 记录：如果 defer 本身位于堆上（d.heap 为 true），需要扫描它的指针。
		if d.heap {
			scanblock(uintptr(unsafe.Pointer(&d)), goarch.PtrSize, &oneptrmask[0], gcw, &state)
		}
	}
	// 扫描 panic 记录
	if gp._panic != nil {
		state.putPtr(uintptr(unsafe.Pointer(gp._panic)), false)
	}
	
	// 构建栈对象索引
	state.buildIndex()
	// 扫描栈对象
	for {
		// 获取待扫描的指针
		p, conservative := state.getPtr()
		if p == 0 {
			break
		}
		// 查找指针对应的栈对象
		obj := state.findObject(p)
		if obj == nil {
			continue
		}
		// 判断对象是否已扫描
		r := obj.r
		if r == nil {
			continue
		}
		// 标记对象为“已扫描”。
		obj.setRecord(nil)
		// 获取对象的扫描信息
		ptrBytes, gcData := r.gcdata()
		// 对象在栈上的基地址
		b := state.stack.lo + uintptr(obj.off)
		// 扫描对象的内存区域
		if conservative {
			// 保守扫描
			scanConservative(b, ptrBytes, gcData, gcw, &state)
		} else {
			// 精确扫描
			scanblock(b, ptrBytes, gcData, gcw, &state)
		}
	}
	
	// 释放栈扫描缓冲区
	// 此循环将遍历所有的缓冲区，释放每一个已分配的缓冲区。
	for state.head != nil {
		x := state.head
		state.head = x.next
		x.nobj = 0
		// 释放缓冲区
		putempty((*workbuf)(unsafe.Pointer(x)))
	}
	return int64(scannedSize)
}
```

##### scanframeworker 扫描堆栈帧

标记堆栈中的所有活动对象，以便 GC 可以正确地收回不再使用的内存。

```go
func scanframeworker(frame *stkframe, state *stackScanState, gcw *gcWork) {
	isAsyncPreempt := frame.fn.valid() && frame.fn.funcID == abi.FuncID_asyncPreempt
	isDebugCall := frame.fn.valid() && frame.fn.funcID == abi.FuncID_debugCallV2
	// 判断是否使用保守扫描
	// 保守扫描确保即使当前函数尚未完全执行（如异步中断时），GC 也能正确地标记和保留仍然活动的对象。
	if state.conservative || isAsyncPreempt || isDebugCall {
		if frame.varp != 0 {
			size := frame.varp - frame.sp
			if size > 0 {
				// 保守扫描
				scanConservative(frame.sp, size, nil, gcw, state)
			}
		}

		// Scan arguments to this frame.
		if n := frame.argBytes(); n != 0 {
			// TODO: We could pass the entry argument map
			// to narrow this down further.
			scanConservative(frame.argp, n, nil, gcw, state)
		}

		if isAsyncPreempt || isDebugCall {
			// This function's frame contained the
			// registers for the asynchronously stopped
			// parent frame. Scan the parent
			// conservatively.
			state.conservative = true
		} else {
			// We only wanted to scan those two frames
			// conservatively. Clear the flag for future
			// frames.
			state.conservative = false
		}
		return
	}

	// 获取堆栈映射
	locals, args, objs := frame.getStackMap(false)

	// 执行精确扫描
	if locals.n > 0 {
		size := uintptr(locals.n) * goarch.PtrSize
		scanblock(frame.varp-size, size, locals.bytedata, gcw, state)
	}

	// 扫描函数参数
	if args.n > 0 {
		scanblock(frame.argp, uintptr(args.n)*goarch.PtrSize, args.bytedata, gcw, state)
	}

	// 扫描堆栈对象
	if frame.varp != 0 {
		// 遍历堆栈中的所有对象，检查每个对象的偏移量（obj.off）和基地址（frame.varp 或 frame.argp）来计算对象的实际地址。
		for i := range objs {
			obj := &objs[i]
			off := obj.off
			base := frame.varp
			if off >= 0 {
				base = frame.argp
			}
			ptr := base + uintptr(off)
			if ptr < frame.sp {
				continue
			}
			if stackTraceDebug {
				println("stkobj at", hex(ptr), "of size", obj.size)
			}
			// 将对象添加到 GC 状态中，以便后续的垃圾回收过程能够正确地识别和保留这些对象。
			state.addObject(ptr, obj)
		}
	}
}
```