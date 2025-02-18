### 介绍

mcache 是线程本地缓存（TLS）的核心组件之一，用于加速内存分配和回收操作。

它通过减少线程之间的竞争，优化了小对象分配的性能。

### 作用

- ##### 本地缓存

  每个操作系统线程（P，即 processor）都有一个独立的 mcache 实例，避免线程之间的锁竞争。
  
  mcache 从全局缓存中心（mcentral）中分配内存并将其缓存起来，以快速响应本地的内存分配请求。

- ##### 管理小对象分配

  mcache 主要用于分配小对象（≤32KB）。大于32KB的对象会直接从heap分配内存

  对于不同大小的对象，mcache 提供快速分配路径。

- #### 减少锁竞争

  线程直接从自己的 mcache 中分配内存，只有当 mcache 耗尽时，才需要访问全局 mheap，从而极大地减少了并发分配时的锁竞争。

### 主要属性

- ##### alloc [numSpanClasses]*mspan

  每个类别对应已分配的span

- ##### stackcache [_NumStackOrders]stackfreelist

  专门为 Goroutine 栈设计的缓存，用于快速分配和回收栈内存，避免频繁从全局堆请求内存。

```go
type stackfreelist struct {
	list gclinkptr // 链表头部的指针
	size uintptr   // 栈块的总大小，以字节为单位
}
```

- ##### flushGen atomic.Uint32

  表示上次刷新此 mcache 的 sweepgen。

  如果 flushGen!=mheap_.sweepgen，表示span已过时，需要释放掉(GC会重置mcache)。

- ##### tiny       uintptr
  ##### tinyoffset uintptr
  ##### tinyAllocs uintptr

  用于优化小对象（<16字节）的分配

  这些小对象通常会聚集在一个内存块中，以减少内存碎片和分配开销。

### 主要方法

##### allocmcache 分配mcache

P 初始化时调用，用来分配 P 对应的 mcache

```go
func allocmcache() *mcache {
    var c *mcache
    systemstack(func() {
        lock(&mheap_.lock)
        // 从全局堆中分配一个 mcache 结构
        c = (*mcache)(mheap_.cachealloc.alloc())
        c.flushGen.Store(mheap_.sweepgen)
        unlock(&mheap_.lock)
    })
    // 初始化 alloc。每个元素指向一个空 mspan（表示还未从堆中分配实际的 mspan）
    for i := range c.alloc {
        c.alloc[i] = &emptymspan
    }
    c.nextSample = nextSample()
    return c
}
```

##### freemcache 释放 mcache

P 销毁时调用

释放与 mcache 关联的资源，并将对象放入空闲列表。

```go
func freemcache(c *mcache) {
    systemstack(func() {
    // 释放 mcache 的所有资源
    c.releaseAll()
    // 清空 stackcache
    stackcache_clear(c)

    // 将 mcache 对象本身释放到全局堆
    lock(&mheap_.lock)
    mheap_.cachealloc.free(unsafe.Pointer(c))
    unlock(&mheap_.lock)
    })
}

// 释放当前 mcache 中的所有 span
func (c *mcache) releaseAll() {
	// 循环释放span
    for i := range c.alloc {
        s := c.alloc[i]
        if s != &emptymspan {
            // span 释放到 mcentral
            mheap_.central[i].mcentral.uncacheSpan(s)
            c.alloc[i] = &emptymspan
        }
    }

    // 重置tinyalloc相关的值
    c.tiny = 0
    c.tinyoffset = 0
    c.tinyAllocs = 0
}
```

##### refill 重新填充span

mcache 分配对象时，如果span所有对象都分配完了，会调用这个

```go
func (c *mcache) refill(spc spanClass) {
	// 当前 span
	s := c.alloc[spc]

	// 还有可用空间
	if s.allocCount != s.nelems {
		throw("refill of span with free space remaining")
	}
	if s != &emptymspan {
		// 确保span是缓存状态
		if s.sweepgen != mheap_.sweepgen+3 {
			throw("bad sweepgen in refill")
		}
		// 将使用完 span 释放到 mcentral
		mheap_.central[spc].mcentral.uncacheSpan(s)
	}

	// 从 mcentral 中获取新的 span。
	s = mheap_.central[spc].mcentral.cacheSpan()
	if s == nil {
		throw("out of memory")
	}

	if s.allocCount == s.nelems {
		throw("span has no free space")
	}
	
	// 指示此 span 已缓存，并防止在下一个扫描阶段进行异步扫描。
	s.sweepgen = mheap_.sweepgen + 3

	// 记录使用前已被分配的对象数量
	s.allocCountBeforeCache = s.allocCount

	c.alloc[spc] = s
}
```

##### allocLarge 分配大对象

对于大于 32KB 的对象，直接从heap中分配一个span

```go
func (c *mcache) allocLarge(size uintptr, noscan bool) *mspan {
	if size+_PageSize < size {
		throw("out of memory")
	}
	npages := size >> _PageShift
	if size&_PageMask != 0 {
		npages++
	}
	
	// 分配 span
	spc := makeSpanClass(0, noscan)
	s := mheap_.alloc(npages, spc)
	if s == nil {
		throw("out of memory")
	}
	
	// 将 span 放入 fullSwept
	mheap_.central[spc].mcentral.fullSwept(mheap_.sweepgen).push(s)
	// 初始化span
	s.limit = s.base() + size
	s.initHeapBits()
	return s
}
```