### 介绍

负责管理动态分配的内存

支持动态内存分配和自动垃圾回收。通过分级分配、本地缓存和并发垃圾回收等机制，在性能和内存利用率之间取得了良好的平衡。

### 主要作用

- 动态内存分配

  堆用于分配那些在编译时无法确定大小的内存块
  
- 全局共享

  所有 Goroutine 共享堆内存

- 垃圾回收

  堆内存由垃圾回收器自动回收

### 结构

- ##### Arena

  Arena 是堆的基本管理单元，每个 Arena 的大小为 64MB（heapArenaBytes）。

  堆由多个 Arena 组成，Arena 的数量取决于程序的内存需求。

  每个 Arena 管理自己的内存分配和元数据。

- ##### Span

  Span 是堆内存分配的基本单位，表示一段连续的内存页

- ##### Size Class

  Size Class 是用于管理小对象分配的分类机制。

  Go 定义了多个 Size Class，每个 Size Class 对应一种固定大小的内存块。

- ##### mspan

  mspan 是 Span 的运行时表示，包含了 Span 的元数据和状态信息。

  每个 mspan 属于一个特定的 Size Class，用于分配固定大小的内存块。

### 主要属性

- ##### pages pageAlloc
  
  页面管理器，管理实际的页面分配和释放

- ##### sweepgen uint32
  
  GC清扫阶段的标识

- ##### allspans []*mspan
  
  所有mspan的全局数组，mspan的内存块的最小粒度

- ##### arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena
  
  每个 arena 是堆内存中的一个大块（通常是 64MB）。
  
  arenas 的主要作用是存储每个 arena 的元数据。

  ```go
  type arenaIdx uint
  
  func arenaIndex(p uintptr) arenaIdx {
    return arenaIdx((p - arenaBaseOffset) / heapArenaBytes)
  }
  ```
  通过指针可以计算出 arenaIdx，然后计算出L1,L2，然后定位到所在的 arena

- ##### arenaHints *arenaHint

  在内存分配过程中，尝试分配新 heapArena 时的地址提示（起始地址）。
  ```go
  type arenaHint struct {
      // 堆区域增长的起始地址
      addr uintptr
      // 标记增长方向：true 表示向下扩展，false 表示向上扩展
      down bool
      next *arenaHint
  }
  ```

- ##### allArenas []arenaIdx
  
  ##### sweepArenas []arenaIdx
  
  ##### markArenas []arenaIdx
  
  arena的全局切片，后面两个用在GC的清扫和标记阶段，是allArenas的快照信息

- ##### central
  
  ```go
  central [numSpanClasses]struct {
      mcentral mcentral
      pad      [(cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize) % cpu.CacheLinePadSize]byte
  }
  ```
  
  central 是用于管理不同大小对象所对应span的中央链表

- ##### pagesInUse atomic.Uintptr

  当前已分配的内存页数

- ##### reclaimIndex atomic.Uint64

  回收进度

### 主要方法

##### init 初始化
```go
func (h *mheap) init() {
    // 初始化锁
    lockInit(&h.lock, lockRankMheap)
    lockInit(&h.speciallock, lockRankMheapSpecial)
    
    // 初始化内存分配器
    h.spanalloc.init(unsafe.Sizeof(mspan{}), recordspan, unsafe.Pointer(h), &memstats.mspan_sys)
    h.cachealloc.init(unsafe.Sizeof(mcache{}), nil, nil, &memstats.mcache_sys)
    h.specialfinalizeralloc.init(unsafe.Sizeof(specialfinalizer{}), nil, nil, &memstats.other_sys)
    h.specialCleanupAlloc.init(unsafe.Sizeof(specialCleanup{}), nil, nil, &memstats.other_sys)
    h.specialprofilealloc.init(unsafe.Sizeof(specialprofile{}), nil, nil, &memstats.other_sys)
    h.specialReachableAlloc.init(unsafe.Sizeof(specialReachable{}), nil, nil, &memstats.other_sys)
    h.specialPinCounterAlloc.init(unsafe.Sizeof(specialPinCounter{}), nil, nil, &memstats.other_sys)
    h.specialWeakHandleAlloc.init(unsafe.Sizeof(specialWeakHandle{}), nil, nil, &memstats.gcMiscSys)
    h.arenaHintAlloc.init(unsafe.Sizeof(arenaHint{}), nil, nil, &memstats.other_sys)
    
    // 不对 mspan 结构体进行清零处理。
    h.spanalloc.zero = false
    
    // 初始化不同size对应的 mcentral
    for i := range h.central {
        h.central[i].mcentral.init(spanClass(i))
    }
    
    // pageAlloc 初始化
    h.pages.init(&h.lock, &memstats.gcMiscSys, false)
}
```

##### 分配 span

分为 heap 管理的 span 和手动管理的 span 的分配

heap 管理的 span 由 GC 自动释放，手动管理的 span 需要手动释放

- heap 管理的 span

```go
func (h *mheap) alloc(npages uintptr, spanclass spanClass) *mspan {
	var s *mspan
    // 避免在 G 栈上执行堆锁操作，防止触发栈增长
	systemstack(func() {
        // 如果尚未完成清扫，回收至少 npages 页内存
		if !isSweepDone() {
			h.reclaim(npages)
		}
		// 分配 span
		s = h.allocSpan(npages, spanAllocHeap, spanclass)
	})
	return s
}
```
<p hidden="hidden">
graph TD
    A[开始] --> B[切换到系统栈执行]
    B --> C{是否已完成清扫?}
    C -->|否| D[回收至少 npages 页内存]
    C -->|是| E[分配内存跨度]
    D --> E
    E --> F[返回分配的 mspan]
    F --> G[结束]
</p>

- 手动管理的 span

```go
func (h *mheap) allocManual(npages uintptr, typ spanAllocType) *mspan {
	if !typ.manual() {
		throw("manual span allocation called with non-manually-managed type")
	}
	return h.allocSpan(npages, typ, 0)
}
```

分配 span

```go
func (h *mheap) allocSpan(npages uintptr, typ spanAllocType, spanclass spanClass) (s *mspan) {
	// 当前 g
	gp := getg()
	base, scav := uintptr(0), uintptr(0)
	// 本次增长的内存
	growth := uintptr(0)
	
	// 是否需要对栈分配进行物理页对齐。
	needPhysPageAlign := physPageAlignedStacks && typ == spanAllocStack && pageSize < physPageSize
	
	// g 对应的处理器 p
	pp := gp.m.p.ptr()
	// 如果需要的 npages<16,从 cache 中获取
	if !needPhysPageAlign && pp != nil && npages < pageCachePages/4 {
		// P 对应的cache
		c := &pp.pcache

		// 如果 cache 为空，重新填充
		if c.empty() {
			lock(&h.lock)
			*c = h.pages.allocToCache()
			unlock(&h.lock)
		}

		// 从 cache 中分配内存
		base, scav = c.alloc(npages)
		if base != 0 {
			// 从 P 的缓存中分配 span
			s = h.tryAllocMSpan()
			if s != nil {
				goto HaveSpan
			}
		}
	}
	
	lock(&h.lock)
	// 需要内存对齐时，先查找足够大的区域，然后手动分配
	if needPhysPageAlign {
		// 额外需要的页面数。
		extraPages := physPageSize / pageSize
		
		// pageAlloc 查找连续的 pages
		base, _ = h.pages.find(npages + extraPages)
		// 如果没找到，heap 需要新增内存
		if base == 0 {
			var ok bool
			growth, ok = h.grow(npages + extraPages)
			if !ok {
				unlock(&h.lock)
				return nil
			}
			// 重新查找
			base, _ = h.pages.find(npages + extraPages)
			if base == 0 {
				throw("grew heap, but no adequate free space found")
			}
		}
		// 地址对齐
		base = alignUp(base, physPageSize)
		// 找到的 page 标记为已分配
		scav = h.pages.allocRange(base, npages)
	}

	// pageAlloc 分配
	if base == 0 {
		base, scav = h.pages.alloc(npages)
		if base == 0 {
			var ok bool
			growth, ok = h.grow(npages)
			if !ok {
				unlock(&h.lock)
				return nil
			}
			base, scav = h.pages.alloc(npages)
			if base == 0 {
				throw("grew heap, but no adequate free space found")
			}
		}
	}
	// 没有从 mspancache 获取到 span，从 heap 分配一个
	if s == nil {
		s = h.allocMSpanLocked()
	}
	unlock(&h.lock)

// 如果找到了span，需要初始化span，更新GC相关的信息和更新统计数据	
HaveSpan:
	// 用于跟踪需要回收的字节数和是否强制进行回收。
	bytesToScavenge := uintptr(0)
	forceScavenge := false
	// 检查当前是否因 CPU 限制暂停垃圾回收操作。如果启用了 CPU 限制（limiting 为 true），则跳过回收检查。
	if limit := gcController.memoryLimit.Load(); !gcCPULimiter.limiting() {
		inuse := gcController.mappedReady.Load()
		// 检查是否超过内存限制。已使用的+这次分配的
		if uint64(scav)+inuse > uint64(limit) {
			bytesToScavenge = uintptr(uint64(scav) + inuse - uint64(limit))
			forceScavenge = true
		}
	}
	//  如果堆增长超过了某个目标值（gcPercentGoal），则可能需要进行内存回收。
	if goal := scavenge.gcPercentGoal.Load(); goal != ^uint64(0) && growth > 0 {
		// heapRetained() 当前堆内存的保留量（已分配但未被释放的内存）
		// 如果 当前保留的内存 + 本次增长的内存 > 目标堆大小（goal），说明堆内存增长超出了预期。
		if retained := heapRetained(); retained+uint64(growth) > goal {
			// 计算需要回收的内存量
			todo := growth
			// 计算堆增长超出的部分：retained + growth - goal。
			if overage := uintptr(retained + uint64(growth) - goal); todo > overage {
				todo = overage
			}
			// 更新最大回收字节数
			if todo > bytesToScavenge {
				bytesToScavenge = todo
			}
		}
	}
	// 如果需要回收内存并且当前有有效的 pp（即物理页），那么执行回收操作
	var now int64
	if pp != nil && bytesToScavenge > 0 {
		start := nanotime()
		track := pp.limiterEvent.start(limiterEventScavengeAssist, start)

		// 内存回收逻辑，目标是释放指定的字节数（bytesToScavenge）
		released := h.pages.scavenge(bytesToScavenge, func() bool {
			return gcCPULimiter.limiting()
		}, forceScavenge)

		// 更新释放的内存统计
		mheap_.pages.scav.releasedEager.Add(released)

		now = nanotime()
		if track {
			pp.limiterEvent.stop(limiterEventScavengeAssist, now)
		}
		scavenge.assistTime.Add(now - start)
	}

	// 初始化span
	h.initSpan(s, typ, spanclass, base, npages)

	// 提交和统计内存
	nbytes := npages * pageSize
	// 是否需要回收
	if scav != 0 {
		// 标记已回收页为可用
		sysUsed(unsafe.Pointer(base), nbytes, scav)
		// 更新回收内存统计
		gcController.heapReleased.add(-int64(scav))
	}
	gcController.heapFree.add(-int64(nbytes - scav))
	if typ == spanAllocHeap {
		gcController.heapInUse.add(int64(nbytes))
	}
	// 更新一致性统计
	stats := memstats.heapStats.acquire()
	atomic.Xaddint64(&stats.committed, int64(scav))
	atomic.Xaddint64(&stats.released, -int64(scav))
	switch typ {
	case spanAllocHeap:
		atomic.Xaddint64(&stats.inHeap, int64(nbytes))
	case spanAllocStack:
		atomic.Xaddint64(&stats.inStacks, int64(nbytes))
	case spanAllocPtrScalarBits:
		atomic.Xaddint64(&stats.inPtrScalarBits, int64(nbytes))
	case spanAllocWorkBuf:
		atomic.Xaddint64(&stats.inWorkBufs, int64(nbytes))
	}
	memstats.heapStats.release()

	// 跟踪内存分配
	if traceAllocFreeEnabled() {
		trace := traceAcquire()
		if trace.ok() {
			trace.SpanAlloc(s)
			traceRelease(trace)
		}
	}
	return s
}
```

从 P 本地缓存分配 mspan 对象

```go
func (h *mheap) tryAllocMSpan() *mspan {
	pp := getg().m.p.ptr()
	// P 为空，或者没有空闲的
	if pp == nil || pp.mspancache.len == 0 {
		return nil
	}
	// 获取 cache 里的最后一个 span
	s := pp.mspancache.buf[pp.mspancache.len-1]
	pp.mspancache.len--
	return s
}
```

初始化 span

```go
func (h *mheap) initSpan(s *mspan, typ spanAllocType, spanclass spanClass, base, npages uintptr) {
	// 初始化
	s.init(base, npages)
	// 清理空间
	if h.allocNeedsZero(base, npages) {
		s.needzero = 1
	}
	nbytes := npages * pageSize
    // 是否手动管理
	if typ.manual() {          
		s.manualFreeList = 0
		s.nelems = 0
		s.limit = s.base() + s.npages*pageSize
		s.state.set(mSpanManual)
	} else {
		s.spanclass = spanclass
		// 大对象 (sizeclass == 0)
		if sizeclass := spanclass.sizeclass(); sizeclass == 0 {
            // span 里只有一个对象
			s.elemsize = nbytes
			s.nelems = 1        
			s.divMul = 0
		} else {
			s.elemsize = uintptr(class_to_size[sizeclass])
			// 是否需要指针/扫描位图
			if !s.spanclass.noscan() && heapBitsInSpan(s.elemsize) {
				s.nelems = uint16((nbytes - (nbytes / goarch.PtrSize / 8)) / s.elemsize)
			} else {
				s.nelems = uint16(nbytes / s.elemsize)
			}
			s.divMul = class_to_divmagic[sizeclass]
		}

		// 初始化标记和分配结构
		s.freeindex = 0
		s.freeIndexForScan = 0
		// 所有page都是空闲
		s.allocCache = ^uint64(0)
		s.gcmarkBits = newMarkBits(uintptr(s.nelems))
		s.allocBits = newAllocBits(uintptr(s.nelems))
		
		// 设置 sweepgen
		atomic.Store(&s.sweepgen, h.sweepgen)
		
		// 设置 span 状态。使用中
		s.state.set(mSpanInUse)
	}
	
	// page 关联 span（反向关联）
	h.setSpans(s.base(), npages, s)

	// 在 arena 页面位图中标记正在使用的span。
	if !typ.manual() {
		// 标记指定 arena.pageInUse[pageIdx] 对应的位图为已使用。
		// pageInUse 只用第一个页做标记，所以只需要设置 base 的
		arena, pageIdx, pageMask := pageIndexOf(s.base())
		atomic.Or8(&arena.pageInUse[pageIdx], pageMask)

		// 更新已分配 pages
		h.pagesInUse.Add(npages)
	}
	
	// 插入内存屏障
	publicationBarrier()
}

func (span *mspan) init(base uintptr, npages uintptr) {
    span.startAddr = base
    span.npages = npages
    span.state.set(mSpanDead)
}
```

##### 释放 span

span的释放分为GC清扫阶段的释放和手动维护span的释放

区别是，GC 清扫阶段的释放是使用系统栈来执行，手动维护span在释放时需要将内存清零

- GC 清扫阶段释放

```go
func (h *mheap) freeSpan(s *mspan) {
	systemstack(func() {
		lock(&h.lock)
		// 释放 span
		h.freeSpanLocked(s, spanAllocHeap)
		unlock(&h.lock)
	})
}
```

- 手动维护span释放

```go
func (h *mheap) freeManual(s *mspan, typ spanAllocType) {
	s.needzero = 1
	lock(&h.lock)
	// 释放 span
	h.freeSpanLocked(s, typ)
	unlock(&h.lock)
}
```

释放 span

```go
func (h *mheap) freeSpanLocked(s *mspan, typ spanAllocType) {
	assertLockHeld(&h.lock)

	switch s.state.get() {
	case mSpanManual:
		if s.allocCount != 0 {
			throw("mheap.freeSpanLocked - invalid stack free")
		}
	case mSpanInUse:
		if s.isUserArenaChunk {
			throw("mheap.freeSpanLocked - invalid free of user arena chunk")
		}
		if s.allocCount != 0 || s.sweepgen != h.sweepgen {
			print("mheap.freeSpanLocked - span ", s, " ptr ", hex(s.base()), " allocCount ", s.allocCount, " sweepgen ", s.sweepgen, "/", h.sweepgen, "\n")
			throw("mheap.freeSpanLocked - invalid free")
		}
		// 更新heap已分配页面数
		h.pagesInUse.Add(-s.npages)

        // 清除 arena 里的 page 已使用标记
		arena, pageIdx, pageMask := pageIndexOf(s.base())
		atomic.And8(&arena.pageInUse[pageIdx], ^pageMask)
	default:
		throw("mheap.freeSpanLocked - invalid span state")
	}
	
	// 更新统计信息
	nbytes := s.npages * pageSize
	gcController.heapFree.add(int64(nbytes))
	if typ == spanAllocHeap {
		gcController.heapInUse.add(-int64(nbytes))
	}
	stats := memstats.heapStats.acquire()
	switch typ {
	case spanAllocHeap:
		atomic.Xaddint64(&stats.inHeap, -int64(nbytes))
	case spanAllocStack:
		atomic.Xaddint64(&stats.inStacks, -int64(nbytes))
	case spanAllocPtrScalarBits:
		atomic.Xaddint64(&stats.inPtrScalarBits, -int64(nbytes))
	case spanAllocWorkBuf:
		atomic.Xaddint64(&stats.inWorkBufs, -int64(nbytes))
	}
	memstats.heapStats.release()

	// 释放 pageAlloc 里的空间（标记为空闲状态）
	h.pages.free(s.base(), s.npages)

	// 设置 span 状态为不再使用
	s.state.set(mSpanDead)
	// 释放 mspan
	h.freeMSpanLocked(s)
}
```

释放 mspan

```go
func (h *mheap) freeMSpanLocked(s *mspan) {
	assertLockHeld(&h.lock)

	pp := getg().m.p.ptr()
	// spanCache 有空间，放回缓存
	if pp != nil && pp.mspancache.len < len(pp.mspancache.buf) {
		pp.mspancache.buf[pp.mspancache.len] = s
		pp.mspancache.len++
		return
	}
	// 直接释放
	h.spanalloc.free(unsafe.Pointer(s))
}
```

##### grow 内存增长

分配内存（allocSpan）时找不到空间内存时触发

```go
func (h *mheap) grow(npage uintptr) (uintptr, bool) {
    // 对齐后的内存大小
    ask := alignUp(npage, pallocChunkPages) * pageSize
    
    // 增长量
    totalGrowth := uintptr(0)
    // 分配请求的结束地址 end，以及对齐后的 nBase 地址。
    end := h.curArena.base + ask
    nBase := alignUp(end, physPageSize)
    // 检查是否有足够的空间
    if nBase > h.curArena.end || /* overflow */ end < h.curArena.base {
        // 分配更多的 arena 空间
        av, asize := h.sysAlloc(ask, &h.arenaHints, true)
        if av == nil {
            inUse := gcController.heapFree.load() + gcController.heapReleased.load() + gcController.heapInUse.load()
            print("runtime: out of memory: cannot allocate ", ask, "-byte block (", inUse, " in use)\n")
            return 0, false
        }
    
        if uintptr(av) == h.curArena.end {
            // 新分配的内存空间与当前 arena 的末尾相邻，直接扩展当前的 arena。
            h.curArena.end = uintptr(av) + asize
        } else {
            // 新分配的空间与当前空间不相邻，就需要处理旧 arena 的残留部分并将它标记为已释放。
            // 这种情况应该很少见。
            
            // 计算当前 arena 中剩余的空间大小，即从 h.curArena.base 到 h.curArena.end 之间的字节数。size 表示当前 arena 中未被使用的内存空间。
            // 如果该空间大小为零，说明当前 arena 中没有可用空间。
            if size := h.curArena.end - h.curArena.base; size != 0 {
                // 转换空间状态并标记为已释放
                sysMap(unsafe.Pointer(h.curArena.base), size, &gcController.heapReleased)
                // page内存增长
                h.pages.grow(h.curArena.base, size)
                totalGrowth += size
            }
            // 切换 curArena
            h.curArena.base = uintptr(av)
            h.curArena.end = uintptr(av) + asize
        }
		
        // 重新计算对齐后的 nBase 地址。
        nBase = alignUp(h.curArena.base+ask, physPageSize)
    }
    
    // 更新 base 地址
    v := h.curArena.base
    h.curArena.base = nBase
	
    // 将空间从 "Reserved" 状态过渡到 "Prepared" 状态
    sysMap(unsafe.Pointer(v), nBase-v, &gcController.heapReleased)
	
    // page内存增长
    h.pages.grow(v, nBase-v)
    totalGrowth += nBase - v
    return totalGrowth, true
}
```

##### reclaim 回收内存

```go
func (h *mheap) reclaim(npage uintptr) {
	// 如果回收的工作已经完成（即 reclaimIndex 已达到某个上限值），则提前退出。
	if h.reclaimIndex.Load() >= 1<<63 {
		return
	}
	
	mp := acquirem()
	arenas := h.sweepArenas
	locked := false
	for npage > 0 {
		// reclaim 会首先检查是否有已经积累的回收信用（reclaimCredit）。
		// 回收信用是指之前回收的内存页数量，可以用于直接释放内存，而无需再次扫描内存区域。
		// 如果没有足够的信用，reclaim 会开始扫描并回收指定数量的内存页：
		if credit := h.reclaimCredit.Load(); credit > 0 {
			take := credit
			if take > npage {
				take = npage
			}
			if h.reclaimCredit.CompareAndSwap(credit, credit-take) {
				npage -= take
			}
			continue
		}

		// 获取一块需要回收的内存区域（“工作块”），并检查是否已经完成了所有需要回收的内存区域。
		idx := uintptr(h.reclaimIndex.Add(pagesPerReclaimerChunk) - pagesPerReclaimerChunk)
		// 检查是否已经完成了所有需要回收的内存区域。
		if idx/pagesPerArena >= uintptr(len(arenas)) {
			// 记录回收已经完成。
			h.reclaimIndex.Store(1 << 63)
			break
		}

		// 开始回收，加锁
		if !locked {
			lock(&h.lock)
			locked = true
		}

		// 扫描并回收一个内存区域（chunk）
		nfound := h.reclaimChunk(arenas, idx, pagesPerReclaimerChunk)
		// 如果回收的数量小于期望值，继续回收；否者将多余的数量加入到回收信用
		if nfound <= npage {
			npage -= nfound
		} else {
			// 将多回收的放入回收信用
			h.reclaimCredit.Add(nfound - npage)
			npage = 0
		}
	}
	if locked {
		unlock(&h.lock)
	}
	
	releasem(mp)
}
```

回收一个 chunk

```go
func (h *mheap) reclaimChunk(arenas []arenaIdx, pageIdx, n uintptr) uintptr {
	assertLockHeld(&h.lock)

	n0 := n
	var nFreed uintptr
	sl := sweep.active.begin()
	if !sl.valid {
		return 0
	}
	for n > 0 {
		// pageIdx 对应的 heapArena
		ai := arenas[pageIdx/pagesPerArena]
		ha := h.arenas[ai.l1()][ai.l2()]

		// 当前页面在特定区域（arena）中的位置
		arenaPage := uint(pageIdx % pagesPerArena)
		// 从 arenaPage 对应的字节位置开始到数组末尾的切片。
		inUse := ha.pageInUse[arenaPage/8:]
		marked := ha.pageMarks[arenaPage/8:]
		// 确保处理的 inUse 和 marked 数组不会超过指定的页面数量 n（只处理需要的字节数）。
		if uintptr(len(inUse)) > n/8 {
			inUse = inUse[:n/8]
			marked = marked[:n/8]
		}

		// 遍历 inUse 位图的每个字节，每个字节代表 8 个页面的使用状态。
		for i := range inUse {
			// &^ 是 Go 的“按位清除”操作。它的效果是将 inUse[i] 中对应 marked[i] 为 1 的位清零，留下那些“未被标记”的页面位。
			inUseUnmarked := atomic.Load8(&inUse[i]) &^ marked[i]
			// 如果当前字节中没有符合条件的页面（即全部位为 0），直接跳过，继续检查下一个字节。
			if inUseUnmarked == 0 {
				continue
			}

			// 逐位检查页面
			for j := uint(0); j < 8; j++ {
				// 通过 1<<j（左移操作），检查当前页面的状态。如果该位为 1，说明这个页面是“正在使用且未被标记”的。
				if inUseUnmarked&(1<<j) != 0 {
					// 从 spans 数组中获取对应的内存 span。
					s := ha.spans[arenaPage+uint(i)*8+j]
					// 尝试获取 span 的所有权。如果成功，说明可以安全地对该 span 进行操作。
					if s, ok := sl.tryAcquire(s); ok {
						// 获取 span 中页面的数量，用于后续统计已释放的页面。
						npages := s.npages
						// 在调用 s.sweep 之前，释放全局锁 h.lock。这避免了长时间持有锁导致的阻塞。
						unlock(&h.lock)
						// 清理内存。如果返回 true，说明 span 被成功回收。
						if s.sweep(false) {
							// 更新实际清理的page数量
							nFreed += npages
						}
						// 完成 s.sweep 操作后，重新获取锁。
						lock(&h.lock)
						// 重新加载位图
						// 在释放锁期间，其他线程可能释放了一些页面，导致 inUse 和 marked 状态发生变化。
						inUseUnmarked = atomic.Load8(&inUse[i]) &^ marked[i]
					}
				}
			}
		}

		// 更新当前处理的页面索引 pageIdx 和剩余页面数 n
		pageIdx += uintptr(len(inUse) * 8)
		n -= uintptr(len(inUse) * 8)
	}
	sweep.active.end(sl)

	assertLockHeld(&h.lock) // Must be locked on return.
	return nFreed
}
```