问题：大的对象如何sweep(不在mcentral里的)


#### sweepGen

- 概念
    sweepGen 是 GC 里的一个重要字段，用于控制 span 的清扫状态，确保每个 span 在 GC 过程中 只会被 sweep 一次，防止多个 P 线程重复 sweep 同一个 span。

- 作用
    
    - 防止 span 被多次 sweep

      - 通过 atomic.Cas 确保 span.sweepGen 只能被一个 P 线程更新，避免并发 sweep。
      - sweepone() 里检查 sweepGen，只有 span.sweepGen == h.sweepgen-2 才需要 sweep。
  
    - 控制 GC 轮次
      - h.sweepgen += 2 控制 GC 轮次，每次 GC 代数增加 2，让 sweep 过程有明确的同步机制。
    - 提升 sweep 效率
      - 只 sweep 需要清扫的 span，避免重复工作。
      - 并发 sweep 时不同 P 线程不会竞争同一个 span，提升性能。

- 位置

    heap 里有全局 sweepGen，每个 span 也有自己的 sweepGen。

    mheap.sweepgen 是整个堆的 sweep 进度，而 mspan.sweepgen 是某个 span 的 sweep 状态，二者对比可以判断 span 是否需要 sweep。

- 取值变化

    sweepGen 的值是一个 单调递增的偶数，每次 GC 周期都会增加 2，在 mark termination 阶段更新

    heap 里的 sweepGen 的取值:
    - N （偶数）：旧的 sweepGen（上次 GC 生成的）
    - N+1 （奇数）：GC 运行时，表示 sweep 进行中
    - N+2 （偶数）：新 GC 轮次，等待 sweep

    span.sweepGen 可能的情况:
    - < h.sweepgen-1：还是上次 GC 的，应该 sweep
    - == h.sweepgen-1：已经 sweep 过了
    - == h.sweepgen：当前 GC 生成的 span，不需要 sweep

### 结构体

##### sweepdata

```go
var sweep sweepdata

type sweepdata struct {
	lock mutex
	g *g
	// 当前清扫 Goroutine 是否处于暂停状态。
	parked bool
	
	// 当前活跃的清扫器数量以及清扫的终止条件。
	active activeSweep
	
	// 当前未清扫的 span 类别。
	// 类别从小到大遍历处理，标记阶段终止时会重置
	centralIndex sweepClass
}
```

活跃的清扫器

```go
type activeSweep struct {
	// 由两部分组成
	// 1. **最高位**（由 `sweepDrainedMask` 掩码标识）：一个布尔值，表示是否所有清扫任务已完成。
	// - 如果设置了最高位，说明清扫工作队列已经被完全清空。
	// 2. **剩余位**（低31位）：一个计数器，表示当前有多少个并发清扫器正在执行任务。
	state atomic.Uint32
}
```

清扫任务类型

```go
// 清扫任务类型
type sweepClass uint32

const (
	// 清扫类别的总数。
	numSweepClasses = numSpanClasses * 2
	// 所有清扫任务已完成。
	sweepClassDone sweepClass = sweepClass(^uint32(0))
)
```

sweepClass 由两部分组成：spanClass 和 full或partial 标记

在 GC 过程中，sweepClass 单调递增

### 主要方法

##### bgsweep 启动后台清理器

启动 GC 时调用

```go
func bgsweep(c chan int) {
	// 将当前运行的 goroutine 记录为清扫器 goroutine。
	sweep.g = getg()

	lockInit(&sweep.lock, lockRankSweep)
	lock(&sweep.lock)
	// 标记已挂起
	sweep.parked = true
	// 通知调用者清扫器已完成初始化。
	c <- 1
	// 挂起 goroutine 并释放锁，等待触发条件唤醒。
	goparkunlock(&sweep.lock, waitReasonGCSweepWait, traceBlockGCSweep, 1)

	// 清扫器的主循环，重复尝试清扫 span。
	for {
		const sweepBatchSize = 10
		nSwept := 0
		// 开始清扫
		for sweepone() != ^uintptr(0) {
			nSwept++
			// 清扫一次数量后，判断是否需要让出 CPU 时间片
			if nSwept%sweepBatchSize == 0 {
				goschedIfBusy()
			}
		}
		// 释放工作缓冲区，循环执行直到无缓冲区可释放。
		for freeSomeWbufs(true) {
			// 释放缓冲区的同时让出时间片以降低优先级。
			goschedIfBusy()
		}
		lock(&sweep.lock)
		// 是否清扫完成
		if !isSweepDone() {
			unlock(&sweep.lock)
			continue
		}
		// 标记已挂起
		sweep.parked = true
		// 挂起 goroutine 并释放锁，等待下一次清扫任务。
		goparkunlock(&sweep.lock, waitReasonGCSweepWait, traceBlockGCSweep, 1)
	}
}
```

##### sweepone 清扫

从 mcentral 里获取一个需要清扫的 span，并尝试 sweep

```go
func sweepone() uintptr {
	gp := getg()
	
	// 保证清扫过程不被抢占
	gp.m.locks++

    // 注册清扫器
	sl := sweep.active.begin()
	if !sl.valid {
		gp.m.locks--
		return ^uintptr(0)
	}

	// 查找可清扫的 span，并执行清扫
	npages := ^uintptr(0)
	var noMoreWork bool
	for {
		// 尝试找到一个需要清扫的 span
		s := mheap_.nextSpanForSweep()
		if s == nil {
			// 如果没有找到，标记清扫工作完成。
			noMoreWork = sweep.active.markDrained()
			break
		}
		// 尝试获取 span 的清扫锁
		if s, ok := sl.tryAcquire(s); ok {
			npages = s.npages
			// 执行 sweep（返回 true 表示内存已释放）
			if s.sweep(false) {
				// 如果 span 已被释放，将其页数加到 reclaimCredit（回收信用，需要回收内存时可优先使用）
				mheap_.reclaimCredit.Add(npages)
			} else {
				// span 仍然在使用中，没有释放页面。
				npages = 0
			}
			break
		}
	}
	// 清扫结束
	sweep.active.end(sl)

	// 清扫工作是否完成
	if noMoreWork {
		// 唤醒清理器
		scavenger.ready()
	}

	gp.m.locks--
	return npages
}
```

从 mcentral 里获取一个需要清扫的 span

```go
func (h *mheap) nextSpanForSweep() *mspan {
	// 当前清扫代
	sg := h.sweepgen
	// 遍历 sweepClass
	for sc := sweep.centralIndex.load(); sc < numSweepClasses; sc++ {
		// 获取 spanClass 和列表类型
		spc, full := sc.split()
		// 获取 mcentral 和对应的 span
		c := &h.central[spc].mcentral
		var s *mspan
		if full {
			s = c.fullUnswept(sg).pop()
		} else {
			s = c.partialUnswept(sg).pop()
		}
		if s != nil {
            // 更新 sweepClass
			sweep.centralIndex.update(sc)
			return s
		}
	}
	// 标记清扫结束
	sweep.centralIndex.update(sweepClassDone)
	return nil
}
```

执行 sweep 工作

清理在标记阶段没有被标记的内存块，并根据条件将未使用的内存返回给堆或留给调用者处理

```go
func (sl *sweepLocked) sweep(preserve bool) bool {
	gp := getg()

	s := sl.mspan
	
	sweepgen := mheap_.sweepgen

	// 累加清扫page数量
	mheap_.pagesSwept.Add(int64(s.npages))

	// span 的类别和对象数量
	spc := s.spanclass
	size := s.elemsize
	
	hadSpecials := s.specials != nil
	siter := newSpecialsIter(s)
	// 遍历特殊记录（特殊记录是与对象关联的元数据，例如终结器（finalizers）和弱引用（weak handles））
	for siter.valid() {
		// 逐个遍历 mspan 中的特殊记录，并根据标记位（mark bits）判断每个对象的状态。
		objIndex := uintptr(siter.s.offset) / size
		p := s.base() + objIndex*size
		mbits := s.markBitsForIndex(objIndex)
		// 对象未被标记
		if !mbits.isMarked() {
			// 第一遍：检查是否有终结器
			hasFinAndRevived := false
			endOffset := p - s.base() + size
			for tmp := siter.s; tmp != nil && uintptr(tmp.offset) < endOffset; tmp = tmp.next {
				if tmp.kind == _KindSpecialFinalizer {
					// 则将对象标记为存活
					mbits.setMarkedNonAtomic()
					hasFinAndRevived = true
					break
				}
			}
			// 第二遍：处理终结器和弱引用
			if hasFinAndRevived {
				// 将所有终结器和弱引用的特殊记录从链表中移除，并调用 freeSpecial 释放它们。
				for siter.valid() && uintptr(siter.s.offset) < endOffset {
					special := siter.s
					p := s.base() + uintptr(special.offset)
					if special.kind == _KindSpecialFinalizer || special.kind == _KindSpecialWeakHandle {
						siter.unlinkAndNext()
						freeSpecial(special, unsafe.Pointer(p), size)
					} else {
						siter.next()
					}
				}
			} else {
				// 释放所有特殊记录
				for siter.valid() && uintptr(siter.s.offset) < endOffset {
					special := siter.s
					p := s.base() + uintptr(special.offset)
					siter.unlinkAndNext()
					freeSpecial(special, unsafe.Pointer(p), size)
				}
			}
		} else {
			// 如果对象被标记为存活
			if siter.s.kind == _KindSpecialReachable {
				// 标记为可达（reachable = true），并释放该特殊记录。
				special := siter.unlinkAndNext()
				(*specialReachable)(unsafe.Pointer(special)).reachable = true
				freeSpecial(special, unsafe.Pointer(p), size)
			} else {
				siter.next()
			}
		}
	}
	// specials 是否已经被清空。
	if hadSpecials && s.specials == nil {
		// 将 span 标记为在 arena 位图中没有 specials
		spanHasNoSpecials(s)
	}

	// 计算当前分配和释放的对象数量
	nalloc := uint16(s.countAlloc())
	nfreed := s.allocCount - nalloc

	// 更新分配数量和索引
	s.allocCount = nalloc
	s.freeindex = 0
	s.freeIndexForScan = 0

	// 切换和重置标记位
	s.allocBits = s.gcmarkBits
	s.gcmarkBits = newMarkBits(uintptr(s.nelems))

    // 重置 allocCache
	s.refillAllocCache(0)
	
	// 更新 sweepgen
	atomic.Store(&s.sweepgen, sweepgen)

	if spc.sizeclass() != 0 { // 处理小对象
		if nfreed > 0 {
			// 标记 span 分配前需要清零
			s.needzero = 1
		}
		if !preserve {
			if nalloc == 0 {
				// 释放 span
				mheap_.freeSpan(s)
				return true
			}
			// 放回 mcentral
			if nalloc == s.nelems {
				mheap_.central[spc].mcentral.fullSwept(sweepgen).push(s)
			} else {
				mheap_.central[spc].mcentral.partialSwept(sweepgen).push(s)
			}
		}
	} else if !preserve { // 处理大对象（大对象独占一个 span）
		if nfreed != 0 {
            // 释放 span
            mheap_.freeSpan(s)
			return true
		}
		// 放回 mcentral
		mheap_.central[spc].mcentral.fullSwept(sweepgen).push(s)
	}
	return false
}
```

获取 span 的清扫所有权

通过 s.sweepgen 与当前 GC 的 sweepgen 来判断是否需要清扫，原子更新防止重复清扫

```go
func (l *sweepLocker) tryAcquire(s *mspan) (sweepLocked, bool) {
	// 检查 span 是否属于当前清扫代
	if atomic.Load(&s.sweepgen) != l.sweepGen-2 {
		return sweepLocked{}, false
	}
	// 更新 s.sweepgen，表示该内存块正在被当前清扫器“锁定”并开始清扫。
	if !atomic.Cas(&s.sweepgen, l.sweepGen-2, l.sweepGen-1) {
		return sweepLocked{}, false
	}
	return sweepLocked{s}, true
}
```