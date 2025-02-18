

##### 工作者

```go
// 工作者
var work workType

// 包含了与 Go 垃圾回收过程中的并发标记、清扫、状态转移、内存管理和性能监控等多个方面相关的字段。
type workType struct {
	// 已满的工作块。
	full lfstack          // lock-free list of full blocks workbuf
	_    cpu.CacheLinePad // prevents false-sharing between full and empty
	// 空闲的工作块。
	empty lfstack          // lock-free list of empty blocks workbuf
	_     cpu.CacheLinePad // prevents false-sharing between empty and nproc/nwait

	// 用于工作块的内存页。
	wbufSpans struct {
		lock mutex
		// free is a list of spans dedicated to workbufs, but
		// that don't currently contain any workbufs.
		// 存储空闲的内存页（spans），这些内存页用于分配工作块，但当前没有工作块。
		free mSpanList
		// busy is a list of all spans containing workbufs on
		// one of the workbuf lists.
		// 包含工作块的内存页。
		busy mSpanList
	}

	// Restore 64-bit alignment on 32-bit.
	_ uint32

	// bytesMarked is the number of bytes marked this cycle. This
	// includes bytes blackened in scanned objects, noscan objects
	// that go straight to black, objects allocated as black during
	// the cycle, and permagrey objects scanned by markroot during
	// the concurrent scan phase.
	//
	// This is updated atomically during the cycle. Updates may be batched
	// arbitrarily, since the value is only read at the end of the cycle.
	//
	// Because of benign races during marking, this number may not
	// be the exact number of marked bytes, but it should be very
	// close.
	//
	// Put this field here because it needs 64-bit atomic access
	// (and thus 8-byte alignment even on 32-bit architectures).
	// 当前垃圾回收周期内标记的字节数。
	bytesMarked uint64

	// 下一个标记根任务的索引。
	markrootNext uint32 // next markroot job
	// 待处理的标记根任务数。
	markrootJobs uint32 // number of markroot jobs

	// 当前处理的工作线程数。
	nproc uint32
	// GC 周期开始的时间戳。
	tstart int64
	// 等待的工作线程数。
	nwait uint32

	// Number of roots of various root types. Set by gcMarkRootPrepare.
	//
	// nStackRoots == len(stackRoots), but we have nStackRoots for
	// consistency.
	// 不同类型的根对象的数量。
	// 根对象是垃圾回收中用来追踪活动对象的基础对象。
	nDataRoots, nBSSRoots, nSpanRoots, nStackRoots int

	// Base indexes of each root type. Set by gcMarkRootPrepare.
	baseData, baseBSS, baseSpans, baseStacks, baseEnd uint32

	// stackRoots is a snapshot of all of the Gs that existed
	// before the beginning of concurrent marking. The backing
	// store of this must not be modified because it might be
	// shared with allgs.
	// 在并发标记开始之前的所有 goroutine 的根对象快照。
	stackRoots []*g

	// Each type of GC state transition is protected by a lock.
	// Since multiple threads can simultaneously detect the state
	// transition condition, any thread that detects a transition
	// condition must acquire the appropriate transition lock,
	// re-check the transition condition and return if it no
	// longer holds or perform the transition if it does.
	// Likewise, any transition must invalidate the transition
	// condition before releasing the lock. This ensures that each
	// transition is performed by exactly one thread and threads
	// that need the transition to happen block until it has
	// happened.
	//
	// startSema protects the transition from "off" to mark or
	// mark termination.
	// 保护 GC 状态转换
	startSema uint32
	// markDoneSema protects transitions from mark to mark termination.
	// 标记阶段完成到标记终止阶段的转换。
	markDoneSema uint32

	// 标记后台标记阶段完成的标志。
	bgMarkDone uint32 // cas to 1 when at a background mark completion point
	// Background mark completion signaling

	// mode is the concurrency mode of the current GC cycle.
	// 当前垃圾回收周期的并发模式。这
	mode gcMode

	// userForced indicates the current GC cycle was forced by an
	// explicit user call.
	// 表示当前的垃圾回收周期是否是由用户显式调用的（例如调用 runtime.GC()）。
	userForced bool

	// initialHeapLive is the value of gcController.heapLive at the
	// beginning of this GC cycle.
	// 当前垃圾回收周期开始时堆的活动内存大小，用于与垃圾回收后期的内存大小进行比较。
	initialHeapLive uint64

	// assistQueue is a queue of assists that are blocked because
	// there was neither enough credit to steal or enough work to
	// do.
	// 一个阻塞队列，存储那些因为没有足够的信用或任务无法执行的辅助操作（assist）。
	assistQueue struct {
		lock mutex
		q    gQueue
	}

	// sweepWaiters is a list of blocked goroutines to wake when
	// we transition from mark termination to sweep.
	// 在垃圾回收从标记阶段终止到清扫阶段开始之间被阻塞的 goroutines。
	// 垃圾回收状态转换时，清扫阶段开始时会唤醒这些 goroutines。
	sweepWaiters struct {
		lock mutex
		list gList
	}

	// strongFromWeak controls how the GC interacts with weak->strong
	// pointer conversions.
	// 控制 GC 与弱指针转换为强指针的交互。
	strongFromWeak struct {
		// block is a flag set during mark termination that prevents
		// new weak->strong conversions from executing by blocking the
		// goroutine and enqueuing it onto q.
		//
		// Mutated only by one goroutine at a time in gcMarkDone,
		// with globally-synchronizing events like forEachP and
		// stopTheWorld.
		// 在标记终止期间，用于阻止新的弱指针转换为强指针。
		block bool

		// q is a queue of goroutines that attempted to perform a
		// weak->strong conversion during mark termination.
		//
		// Protected by lock.
		lock mutex
		// 尝试在标记终止期间进行弱->强转换的 goroutine 队列。
		q gQueue
	}

	// cycles is the number of completed GC cycles, where a GC
	// cycle is sweep termination, mark, mark termination, and
	// sweep. This differs from memstats.numgc, which is
	// incremented at mark termination.
	// 已完成的 GC 周期数。
	cycles atomic.Uint32

	// Timing/utilization stats for this cycle.
	// stwprocs 在垃圾回收期间参与 STW（Stop-The-World）的核心数。
	// maxprocs 最大可用的核心数。
	stwprocs, maxprocs int32
	// 记录了垃圾回收过程中不同阶段的时间戳，用于衡量各个阶段的持续时间（以纳秒为单位）。
	tSweepTerm, tMark, tMarkTerm, tEnd int64 // nanotime() of phase start

	// pauseNS is the total STW time this cycle, measured as the time between
	// when stopping began (just before trying to stop Ps) and just after the
	// world started again.
	// 垃圾回收周期内的总停顿时间，即从尝试停止 P（处理器）开始，到世界恢复正常执行的时间。
	pauseNS int64

	// debug.gctrace heap sizes for this cycle.
	// 垃圾回收周期中用于调试跟踪的堆大小数据（gctrace）。
	heap0, heap1, heap2 uint64

	// Cumulative estimated CPU usage.
	// 当前 GC 周期的累计 CPU 使用情况。
	cpuStats
}
```


### 主要方法

##### 获取标记任务

快速获取

```go
func (w *gcWork) tryGetFast() uintptr {
	wbuf := w.wbuf1
	// 为空时直接返回
	if wbuf == nil || wbuf.nobj == 0 {
		return 0
	}

	wbuf.nobj--
	return wbuf.obj[wbuf.nobj]
}
```

获取任务

```go
func (w *gcWork) tryGet() uintptr {
	wbuf := w.wbuf1
	// 检查主缓冲区（wbuf1）是否为空
	if wbuf == nil {
		// 初始化
		w.init()
		wbuf = w.wbuf1
	}
	// 检查当前主缓冲区是否有对象可用
	if wbuf.nobj == 0 {
		// 如果 wbuf1.nobj == 0（当前主缓冲区没有工作项），交换 wbuf1 和 wbuf2。
		w.wbuf1, w.wbuf2 = w.wbuf2, w.wbuf1
		// 将次缓冲区（wbuf2）切换为新的主缓冲区。
		wbuf = w.wbuf1
		// 如果切换后的主缓冲区仍然为空，调用 trygetfull() 从全局队列中获取一个满缓冲区。
		if wbuf.nobj == 0 {
			owbuf := wbuf
			// 从全局队列中获取一个满缓冲区。
			wbuf = trygetfull()
			if wbuf == nil {
				return 0
			}
			// 将之前的空缓冲区（owbuf）放回空缓冲区池。
			putempty(owbuf)
			// 更新 wbuf1 为新的主缓冲区。
			w.wbuf1 = wbuf
		}
	}

	// 弹出对象指针
	wbuf.nobj--
	return wbuf.obj[wbuf.nobj]
}
```

##### balance 平衡工作任务

将部分缓存的工作（灰色对象）从本地缓冲区返还到全局队列，以便其他 GC 工作线程能够获取这些工作。

```go
func (w *gcWork) balance() {
	// 如果主缓冲区 wbuf1 未初始化（nil），直接返回
	if w.wbuf1 == nil {
		return
	}
	// 如果次缓冲区有对象，移动到全局队列
	if wbuf := w.wbuf2; wbuf.nobj != 0 {
		putfull(wbuf)
		w.flushedWork = true
		w.wbuf2 = getempty()
	} else if wbuf := w.wbuf1; wbuf.nobj > 4 {
		// 如果主缓冲区有超过 4 个对象，移动部分到全局队列
		w.wbuf1 = handoff(wbuf)
		w.flushedWork = true
	} else {
		return
	}
	// 唤醒 GC 工作线程
	if gcphase == _GCmark {
		gcController.enlistWorker()
	}
}
```