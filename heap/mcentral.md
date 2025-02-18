### 介绍

固定大小的span管理中心

### 主要属性

- ##### spanclass spanClass
  
  ```go
  type spanClass uint8
  ```

- ##### partial [2]spanSet
  
  有空闲span列表

- ##### full [2]spanSet
  
  已满span列表
  
  partial和full都是两个列表，会根据sweepgen（GC清扫版本）来计算哪个列表是已清扫（可直接使用），哪个未清扫

### 主要方法

##### init 初始化

```go
func (c *mcentral) init(spc spanClass) {
    c.spanclass = spc
    lockInit(&c.partial[0].spineLock, lockRankSpanSetSpine)
    lockInit(&c.partial[1].spineLock, lockRankSpanSetSpine)
    lockInit(&c.full[0].spineLock, lockRankSpanSetSpine)
    lockInit(&c.full[1].spineLock, lockRankSpanSetSpine)
}
```

spc是span的size类别，获取span的时候也是从对应的spc列表中获取

##### cacheSpan 分配span给mcache使用

```go
func (c *mcentral) cacheSpan() *mspan {
    spanBytes := uintptr(class_to_allocnpages[c.spanclass.sizeclass()]) * _PageSize
    spanBudget := 100

    var s *mspan
    var sl sweepLocker

    // 从已清扫过的 span 空闲列表中获取一个可用的 span
    sg := mheap_.sweepgen
    if s = c.partialSwept(sg).pop(); s != nil {
        goto havespan
    }

    // 注册一个新的清扫器，清扫那些已经部分清扫的span
    sl = sweep.active.begin()
    if sl.valid {
        // 清扫有空闲列表
        for ; spanBudget >= 0; spanBudget-- {
            s = c.partialUnswept(sg).pop()
            if s == nil {
                break
            }
            // 获取所有权，并执行清扫
            if s, ok := sl.tryAcquire(s); ok {
                s.sweep(true)
                sweep.active.end(sl)
                goto havespan
            }
        }
        // 清扫已满列表
        for ; spanBudget >= 0; spanBudget-- {
            s = c.fullUnswept(sg).pop()
            if s == nil {
                break
            }
            // 获取所有权，并执行清扫
            if s, ok := sl.tryAcquire(s); ok {
                s.sweep(true)
                // 检测是否有空闲空间
                freeIndex := s.nextFreeIndex()
                if freeIndex != s.nelems {
                    s.freeindex = freeIndex
                    sweep.active.end(sl)
                    goto havespan
                }
                // 加入已清扫列表
                c.fullSwept(sg).push(s.mspan)
            }
        }
        sweep.active.end(sl)
    }

    // mcentral 中没有可用的span，从heap中获取一个
    s = c.grow()
    if s == nil {
        return nil
    }

    // 初始化获取到的 span
havespan:
    if !traceDone {
        trace := traceAcquire()
        if trace.ok() {
            trace.GCSweepDone()
            traceRelease(trace)
        }
    }
    // 确保当前span有空闲的对象可以分配
    n := int(s.nelems) - int(s.allocCoeindex == s.nelems || s.allocCount == s.nelems {
        throw("span has no free objects")
    }
    // 初始化分配位图缓存
    freeByteBase := &^ (64 - 1)
    whichByte := freeByteBase / 8
    s.refillAllocCache(whichByte)

    // 调整 allocCache 以匹配当前分配状态
    // 根据 freeindex 的位置调整 allocCache，确保低位对齐到下一个可分配的对象
    s.allocCache >>= s.freeindex % 64

    return s
}
```
##### uncacheSpan 从 mcache 释放 span
如果mcache被释放或者span不够用了，会将span释放给mcentral
```go
func (c *mcentral) uncacheSpan(s *mspan) {
	// 当前GC清扫版本
	sg := mheap_.sweepgen
	// 对比span和heap的清扫版本，判断是否需要清扫
	stale := s.sweepgen == sg+1

	if stale {
		// 标记为待清扫
		atomic.Store(&s.sweepgen, sg-1)
	} else {
		// Span 已经被清扫或者无需清扫，直接标记为已清扫。
		atomic.Store(&s.sweepgen, sg)
	}

	if stale {
		// 如果 span 是陈旧的，直接开始 sweep
		ss := sweepLocked{s}
		ss.sweep(false)
	} else {
		if int(s.nelems)-int(s.allocCount) > 0 {
			// 放回空闲列表（已清扫）
			c.partialSwept(sg).push(s)
		} else {
			// 放回已满列表（已清扫）
			c.fullSwept(sg).push(s)
		}
	}
}
```
##### grow 从堆中分配一个新的span
```go
func (c *mcentral) grow() *mspan {
	// size对应的page数
	npages := uintptr(class_to_allocnpages[c.spanclass.sizeclass()])
	// class对应的size
	size := uintptr(class_to_size[c.spanclass.sizeclass()])

	// 申请内存(指定的size)
	s := mheap_.alloc(npages, c.spanclass)
	if s == nil {
		return nil
	}

	// 计算对象数量 
	n := s.divideByElemSize(npages << _PageShift)
	// 初始化span
	s.limit = s.base() + size*n
	s.initHeapBits()
	return s
}
```

### 总结

mcentral相当于全局的span缓存中心，mcache会从这里申请和释放span。

partial和full的设计，用来高效管理和分类span的分配和回收状态。

使用两个列表来配合GC的工作，支持并行GC和分配。