### 介绍

Go 的每个 Goroutine 都有一个独立的栈，用于存储函数调用的局部变量、参数和返回地址。为了防止栈溢出，Go 运行时会在栈的底部保留一部分空间（称为 栈保护区域），并在函数调用时检查栈指针是否越界。

##### stackObject 栈对象

```go
type stackObject struct {
	_ sys.NotInHeap
	// 相对于堆栈底部（stack.lo）的偏移量。
	off uint32
	// 变量的大小（以字节为单位）。
	size uint32
	// 变量的详细信息，包括指针和非指针的分布信息。
	r *stackObjectRecord
	// 偏移地址更低的堆栈对象。
	left *stackObject
	// 偏移地址更高的堆栈对象。
	right *stackObject
}
```

##### stackObjectRecord 栈对象详细信息

```go
type stackObjectRecord struct {
	// 栈对象的偏移量
	// 如果是负值，表示从 varp（局部变量指针）开始的偏移。
	// 如果是非负值，表示从 argp（参数指针）开始的偏移。
	off int32
	// 栈对象的大小（以字节为单位）
	size int32
	// 栈对象中的指针占用的字节数
	ptrBytes int32
	// GC 数据的偏移量，用于在模块数据的只读数据段（moduledata.rodata）中找到与栈对象相关的垃圾回收数据。
	gcdataoff uint32
}
```

### 主要方法

##### 初始化

初始化小栈缓冲池和大栈缓冲池

```go
func stackinit() {
	if _StackCacheSize&_PageMask != 0 {
		throw("cache size must be a multiple of page size")
	}
	// 初始化小栈池
	for i := range stackpool {
		stackpool[i].item.span.init()
		lockInit(&stackpool[i].item.mu, lockRankStackpool)
	}
	// 初始化大栈池
	for i := range stackLarge.free {
		stackLarge.free[i].init()
		lockInit(&stackLarge.lock, lockRankStackLarge)
	}
}
```

小栈

```go
var stackpool [_NumStackOrders]struct {
	item stackpoolItem
}
type stackpoolItem struct {
    mu mutex
    span mSpanList
}
type mSpanList struct {
    first *mspan
    last  *mspan
}
```

大栈

```go
var stackLarge struct {
	lock mutex
	free [heapAddrBits - pageShift]mSpanList
}
```

### 分配内存

根据内存大小使用小栈或者大栈的分配逻辑

- 小栈分配。优先从 P 自带的 stackcache 中获取，没有缓存就从 stackpool 中获取

- 大栈分配。先从大栈缓存中获取，如果分配失败，直接从 heap 中分配

```go
func stackalloc(n uint32) stack {
	thisg := getg()
	
	var v unsafe.Pointer
	// 小栈分配
	if n < fixedStack<<_NumStackOrders && n < _StackCacheSize {
		order := uint8(0)
		n2 := n
		for n2 > fixedStack {
			order++
			n2 >>= 1
		}
		var x gclinkptr
		if stackNoCache != 0 || thisg.m.p == 0 || thisg.m.preemptoff != "" {
			// 无缓存分配
			lock(&stackpool[order].item.mu)
			x = stackpoolalloc(order)
			unlock(&stackpool[order].item.mu)
		} else {
			// 缓存分配
			c := thisg.m.p.ptr().mcache
			x = c.stackcache[order].list
			// 如果缓存为空，从全局池中补充缓存。
			if x.ptr() == nil {
				stackcacherefill(c, order)
				x = c.stackcache[order].list
			}
			c.stackcache[order].list = x.ptr().next
			c.stackcache[order].size -= uintptr(n)
		}
		v = unsafe.Pointer(x)
	} else {
		// 大栈分配
		var s *mspan
		npage := uintptr(n) >> _PageShift
		log2npage := stacklog2(npage)

		lock(&stackLarge.lock)
		// 尝试从大栈缓存中获取
		if !stackLarge.free[log2npage].isEmpty() {
			s = stackLarge.free[log2npage].first
			stackLarge.free[log2npage].remove(s)
		}
		unlock(&stackLarge.lock)

		lockWithRankMayAcquire(&mheap_.lock, lockRankMheap)

		// 如果缓存中没有合适的栈段，则从堆中分配。
		if s == nil {
			// 从堆中分配栈段
			s = mheap_.allocManual(npage, spanAllocStack)
			if s == nil {
				throw("out of memory")
			}
			// 操作系统分配实际内存
			osStackAlloc(s)
			s.elemsize = uintptr(n)
		}
		v = unsafe.Pointer(s.base())
	}

	return stack{uintptr(v), uintptr(v) + uintptr(n)}
}
```

##### 释放内存

栈的释放逻辑和分配逻辑类似

```go
func stackfree(stk stack) {
	gp := getg()
	// 栈的起始地址和大小。
	v := unsafe.Pointer(stk.lo)
	n := stk.hi - stk.lo
	
	if n < fixedStack<<_NumStackOrders && n < _StackCacheSize {
		// 小栈释放
		order := uint8(0)
		n2 := n
		for n2 > fixedStack {
			order++
			n2 >>= 1
		}
		x := gclinkptr(v)
		if stackNoCache != 0 || gp.m.p == 0 || gp.m.preemptoff != "" {
			// 无缓存释放
			lock(&stackpool[order].item.mu)
			stackpoolfree(x, order)
			unlock(&stackpool[order].item.mu)
		} else {
			// 缓存释放
			c := gp.m.p.ptr().mcache
			if c.stackcache[order].size >= _StackCacheSize {
				stackcacherelease(c, order)
			}
			x.ptr().next = c.stackcache[order].list
			c.stackcache[order].list = x
			c.stackcache[order].size += n
		}
	} else {
		// 大栈释放
		s := spanOfUnchecked(uintptr(v))
		if s.state.get() != mSpanManual {
			println(hex(s.base()), v)
			throw("bad span state")
		}
		if gcphase == _GCoff {
			// 如果垃圾回收器未运行，直接释放栈：
			osStackFree(s)
			mheap_.freeManual(s, spanAllocStack)
		} else {
			// 将栈添加到全局的大栈缓存（stackLarge.free），避免与垃圾回收器竞争。
			log2npage := stacklog2(s.npages)
			lock(&stackLarge.lock)
			stackLarge.free[log2npage].insert(s)
			unlock(&stackLarge.lock)
		}
	}
}
```

##### 栈扩展

在 Goroutine 的栈空间不足时被调用。它的主要职责是分配更大的栈并将 Goroutine 的执行状态迁移到新栈中。

```go
func newstack() {
	// 获取当前 Goroutine
	thisg := getg()
	
	// 获取当前 M（OS 线程）正在执行的 Goroutine
	gp := thisg.m.curg

	// 清理 morebuf（切换到更大栈时的上下文信息）
	morebuf := thisg.m.morebuf
	thisg.m.morebuf.pc = 0
	thisg.m.morebuf.lr = 0
	thisg.m.morebuf.sp = 0
	thisg.m.morebuf.g = 0
	
	// 加载 stackguard0（栈保护边界）
	stackguard0 := atomic.Loaduintptr(&gp.stackguard0)

	// 是否需要抢占
	preempt := stackguard0 == stackPreempt
	if preempt {
		// 检查当前 M 是否可以安全地抢占。
		if !canPreemptM(thisg.m) {
			// 如果不能抢占（例如，M 正在持有锁或进行内存分配），会重置 stackguard0 并允许 Goroutine 继续运行。
			gp.stackguard0 = gp.stack.lo + stackGuard
			gogo(&gp.sched)
		}
	}
	
	// 更新栈指针
	sp := gp.sched.sp

	// 是否需要抢占
	if preempt {
		// 栈收缩
		if gp.preemptShrink {
			gp.preemptShrink = false
			shrinkstack(gp)
		}

		// 停止抢占
		if gp.preemptStop {
			preemptPark(gp) // never returns
		}

		// 将控制权交还给调度器
		gopreempt_m(gp)
	}

	// 栈增长计算，2倍增长
	oldsize := gp.stack.hi - gp.stack.lo
	newsize := oldsize * 2
	
	// 动态调整栈大小
	if f := findfunc(gp.sched.pc); f.valid() {
		max := uintptr(funcMaxSPDelta(f))
		needed := max + stackGuard
		used := gp.stack.hi - gp.sched.sp
		for newsize-used < needed {
			newsize *= 2
		}
	}

	if stackguard0 == stackForceMove {
		newsize = oldsize
	}
	
	// 栈状态切换
	casgstatus(gp, _Grunning, _Gcopystack)
	
	// 复制栈到新段
	copystack(gp, newsize)
	if stackDebug >= 1 {
		print("stack grow done\n")
	}
	// 恢复 Goroutine 状态
	casgstatus(gp, _Gcopystack, _Grunning)
	// 恢复 Goroutine 的执行，从调度点继续运行。
	gogo(&gp.sched)
}
```

##### 栈收缩

```go
func shrinkstack(gp *g) {
	// // 检查栈是否有效
	if gp.stack.lo == 0 {
		throw("missing stack in shrinkstack")
	}

	// 如果配置了 debug.gcshrinkstackoff，则不进行栈收缩
	if debug.gcshrinkstackoff > 0 {
		return
	}
	// 检查是否处于特殊的函数（如 GC 工作线程）
	f := findfunc(gp.startpc)
	if f.valid() && f.funcID == abi.FuncID_gcBgMarkWorker {
		return
	}

	// 计算新的栈大小
	oldsize := gp.stack.hi - gp.stack.lo
	newsize := oldsize / 2
	// 不允许栈小于最小值
	if newsize < fixedStack {
		return
	}
	// 判断栈的当前使用量，决定是否进行缩小
	avail := gp.stack.hi - gp.stack.lo
	if used := gp.stack.hi - gp.sched.sp + stackNosplit; used >= avail/4 {
		return
	}

	// 执行栈收缩
	copystack(gp, newsize)
}
```

### 复制栈

创建新栈和栈收缩时，需要复制栈

##### copystack 复制栈

```go
func copystack(gp *g, newsize uintptr) {
	if gp.syscallsp != 0 {
		throw("stack growth not allowed in system call")
	}
	// 获取旧栈的使用情况
	old := gp.stack
	if old.lo == 0 {
		throw("nil stackbase")
	}
	// 栈中已使用的部分大小，表示栈中从栈顶到栈指针之间的空间。
	used := old.hi - gp.sched.sp
	// 将新的栈大小变更通知到垃圾回收器（GC）。
	gcController.addScannableStack(getg().m.p.ptr(), int64(newsize)-int64(old.hi-old.lo))

	// 分配新栈
	new := stackalloc(uint32(newsize))
	if stackPoisonCopy != 0 {
		fillstack(new, 0xfd)
	}
	if stackDebug >= 1 {
		print("copystack gp=", gp, " [", hex(old.lo), " ", hex(old.hi-used), " ", hex(old.hi), "]", " -> [", hex(new.lo), " ", hex(new.hi-used), " ", hex(new.hi), "]/", newsize, "\n")
	}

	// 计算栈的调整信息
	var adjinfo adjustinfo
	adjinfo.old = old
	adjinfo.delta = new.hi - old.hi

	// 调整 sudogs
	ncopy := used
	// 判断当前 goroutine 是否有与栈相关的 channel 操作。
	if !gp.activeStackChans {
		if newsize < old.hi-old.lo && gp.parkingOnChan.Load() {
			throw("racy sudog adjustment due to parking on channel")
		}
		adjustsudogs(gp, &adjinfo)
	} else {
		adjinfo.sghi = findsghi(gp, old)
		
		// 同步调整与 channel 操作相关的 sudogs。
		ncopy -= syncadjustsudogs(gp, used, &adjinfo)
	}

	// 复制栈
	memmove(unsafe.Pointer(new.hi-ncopy), unsafe.Pointer(old.hi-ncopy), ncopy)
	
	// 调整栈中的其他指针
	adjustctxt(gp, &adjinfo)
	adjustdefers(gp, &adjinfo)
	adjustpanics(gp, &adjinfo)
	if adjinfo.sghi != 0 {
		adjinfo.sghi += adjinfo.delta
	}

	// 更新栈指针
	gp.stack = new
	// 更新为新栈的保护区位置。
	gp.stackguard0 = new.lo + stackGuard
	gp.sched.sp = new.hi - used
	gp.stktopsp += adjinfo.delta

	// 调整栈帧
	var u unwinder // unwinder 是一个栈展开器，遍历栈帧并调整其中的指针。
	for u.init(gp, 0); u.valid(); u.next() {
		adjustframe(&u.frame, &adjinfo)
	}

	// 释放旧栈
	if stackPoisonCopy != 0 {
		fillstack(old, 0xfc)
	}
	stackfree(old)
}
```

##### adjustframe 调整栈帧指针

处理栈帧中的局部变量、参数、返回值，以及保存的帧指针，确保在栈重新分配（如扩展）后，所有指针指向新栈中的正确地址。

复制栈的时候需要调整栈帧指针

```go
func adjustframe(frame *stkframe, adjinfo *adjustinfo) {
	f := frame.fn

	// 在 AMD64 和 ARM64 上，可能需要调整保存的帧指针。
	if (goarch.ArchFamily == goarch.AMD64 || goarch.ArchFamily == goarch.ARM64) && frame.argp-frame.varp == 2*goarch.PtrSize {
		// 更新帧指针地址。
		adjustpointer(adjinfo, unsafe.Pointer(frame.varp))
	}

	// 获取局部变量、参数和栈对象的位图。
	locals, args, objs := frame.getStackMap(true)

	// 如果栈帧分配了局部变量，根据位图调用 adjustpointers 逐个更新指针。
	if locals.n > 0 {
		size := uintptr(locals.n) * goarch.PtrSize
		adjustpointers(unsafe.Pointer(frame.varp-size), &locals, adjinfo, f)
	}

	// 如果位图中存在参数，调用 adjustpointers 更新这些指针。
	if args.n > 0 {
		adjustpointers(unsafe.Pointer(frame.argp), &args, adjinfo, funcInfo{})
	}
	
	// 栈对象调整
	if frame.varp != 0 {
		// 迭代栈对象
		for i := range objs {
			obj := &objs[i]
			// 栈对象可能是局部变量或函数参数，基于 off 确定其基地址。
			off := obj.off
			base := frame.varp
			// 如果对象尚未分配，则跳过。
			if off >= 0 {
				base = frame.argp
			}
			p := base + uintptr(off)
			if p < frame.sp {
				continue
			}
			// 解析对象的指针位图 (gcData)，逐一调整指针地址。
			ptrBytes, gcData := obj.gcdata()
			for i := uintptr(0); i < ptrBytes; i += goarch.PtrSize {
				if *addb(gcData, i/(8*goarch.PtrSize))>>(i/goarch.PtrSize&7)&1 != 0 {
					adjustpointer(adjinfo, unsafe.Pointer(p+i))
				}
			}
		}
	}
}
```

##### adjustctxt 调整上下文信息

该函数确保 gp.sched 的相关指针指向新的栈位置。

```go
func adjustctxt(gp *g, adjinfo *adjustinfo) {
	// 调整调度上下文指针
	adjustpointer(adjinfo, unsafe.Pointer(&gp.sched.ctxt))
	// 如果当前环境未启用帧指针，直接返回
	if !framepointer_enabled {
		return
	}
	// 调整帧指针
	oldfp := gp.sched.bp
	adjustpointer(adjinfo, unsafe.Pointer(&gp.sched.bp))
	// 在 ARM64 架构上，帧指针 (bp) 保存的位置特殊
	if GOARCH == "arm64" {
		if oldfp == gp.sched.sp-goarch.PtrSize {
			memmove(unsafe.Pointer(gp.sched.bp), unsafe.Pointer(oldfp), goarch.PtrSize)
			adjustpointer(adjinfo, unsafe.Pointer(gp.sched.bp))
		}
	}
}
```

##### adjustdefers 调整 defer

```go
func adjustdefers(gp *g, adjinfo *adjustinfo) {
	// 调整 Goroutine 的 _defer 链表头指针
	adjustpointer(adjinfo, unsafe.Pointer(&gp._defer))
	// 遍历并调整链表中的指针
	for d := gp._defer; d != nil; d = d.link {
		adjustpointer(adjinfo, unsafe.Pointer(&d.fn))
		adjustpointer(adjinfo, unsafe.Pointer(&d.sp))
		adjustpointer(adjinfo, unsafe.Pointer(&d.link))
	}
}
```

##### adjustpanics 调整 panic

```go
func adjustpanics(gp *g, adjinfo *adjustinfo) {
    adjustpointer(adjinfo, unsafe.Pointer(&gp._panic))
}
```

##### 调整 sudog

同步调整一个 Goroutine 的 sudog 结构体，同时同步与通道操作相关的栈数据。

```go
func syncadjustsudogs(gp *g, used uintptr, adjinfo *adjustinfo) uintptr {
	if gp.waiting == nil {
		return 0
	}

	var lastc *hchan
	// 遍历等待队列，对每个 sudog 的通道 (hchan) 加锁。
	for sg := gp.waiting; sg != nil; sg = sg.waitlink {
		if sg.c != lastc {
			lockWithRank(&sg.c.lock, lockRankHchanLeaf)
		}
		lastc = sg.c
	}

	// 调整 sudog
	adjustsudogs(gp, adjinfo)
	
	// 复制栈数据
	var sgsize uintptr
	if adjinfo.sghi != 0 {
		// 旧栈的底部，等于栈顶部减去已使用字节。
		oldBot := adjinfo.old.hi - used
		// 新栈的底部，等于 oldBot 加上栈增长量 (adjinfo.delta)。
		newBot := oldBot + adjinfo.delta
		// 栈中 sudog 数据所占字节数。
		sgsize = adjinfo.sghi - oldBot
		// 将旧栈中的数据复制到新栈。
		memmove(unsafe.Pointer(newBot), unsafe.Pointer(oldBot), sgsize)
	}

	// 解锁通道
	lastc = nil
	for sg := gp.waiting; sg != nil; sg = sg.waitlink {
		if sg.c != lastc {
			unlock(&sg.c.lock)
		}
		lastc = sg.c
	}

	return sgsize
}
```

调整 sudog

```go
func adjustsudogs(gp *g, adjinfo *adjustinfo) {
	// 遍历等待链表，调整指针
	for s := gp.waiting; s != nil; s = s.waitlink {
		adjustpointer(adjinfo, unsafe.Pointer(&s.elem))
	}
}
```

##### adjustpointer 调整指针

调整指针在栈迁移过程中的指向

```go
func adjustpointer(adjinfo *adjustinfo, vpp unsafe.Pointer) {
	// 将传入的指针视为指针的指针
	pp := (*uintptr)(vpp)
	// 解引用，获取实际指针的值
	p := *pp
	// 判断指针是否在旧栈范围内
	if adjinfo.old.lo <= p && p < adjinfo.old.hi {
		// 调整指针的值，使其指向新栈的对应位置
		*pp = p + adjinfo.delta
	}
}
```