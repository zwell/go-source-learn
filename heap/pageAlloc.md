### 介绍

页面分配器主要用于管理已映射的内存页，提供高效的页面分配和重用机制：

- 管理的单位：以 pageSize 为基本单位（注意，不是物理页大小 physPageSize）。

- 嵌入 mheap：页面分配器是 mheap 的一部分，用于支持 Go 的内存管理机制。

- 数据结构核心：基于位图（bitmap）与稀疏数组（sparse array）。

### 数据结构设计

页面分配器采用了一种 分层的位图管理机制 和 隐式的基数树（radix tree） 设计。

- ##### 位图（bitmap）
  
  位图存储每一页的状态：1 表示该页已被使用；0 表示该页是空闲的，可以分配。
  
  位图覆盖了整个进程的地址空间。

- ##### 基数树（[radix tree]([数据结构之Radix Tree | Ivanzz](https://ivanzz1001.github.io/records/post/data-structure/2018/11/18/ds-radix-tree))）
  
  位图的高效搜索通过 基数树（radix tree） 实现。
  
  - 分层结构：基数树按地址空间划分层次，每一层用一个连续数组表示。
  
  - 总结信息（summary）：每个基数树节点记录某个区域的总结信息，包括：区域内开头的连续空闲页数量。区域末尾的连续空闲页数量。区域内最大连续空闲页数量。

- ##### 分层粒度
  
  每一层的节点表示地址空间的不同粒度：
  
  - 根层（L0）：表示最大的地址空间块（64 位系统中为 16 GiB）。
  
  - 子层：逐层细分地址空间块，每层表示的粒度递减（如从 16 GiB -> 2 GiB -> 256 MiB 等）。
  
  - 叶层：直接对应位图的块（chunk）。

### 主要属性

- ##### summary [summaryLevels][]pallocSum
  
  Radix Tree（基数树）中的 summaries（摘要层级）的内存管理和使用方式
  
  summary 是一个二维切片数组，表示基数树的各个层级：
  
  - ###### summaryLevels：基数树的层级数，决定了 summary 有多少级摘要数据。
    
    ```go
    summaryLevels = 5
    summaryLevelBits = 3
    // 64位系统是14
    summaryL0Bits = heapAddrBits - logPallocChunkBytes - (summaryLevels-1)*summaryLevelBits
    logPallocChunkPages = 9
    ```
    
    ```go
    // 基数树每一层使用的位数 [14,3,3,3,3]
    var levelBits = [summaryLevels]uint{
        summaryL0Bits,
        summaryLevelBits,
        summaryLevelBits,
        summaryLevelBits,
        summaryLevelBits,
    }
    
    // 对地址进行右移操作后可以获得该层索引的位移量 [34,31,28,25,22]
    var levelShift = [summaryLevels]uint{
        heapAddrBits - summaryL0Bits,
        heapAddrBits - summaryL0Bits - 1*summaryLevelBits,
        heapAddrBits - summaryL0Bits - 2*summaryLevelBits,
        heapAddrBits - summaryL0Bits - 3*summaryLevelBits,
        heapAddrBits - summaryL0Bits - 4*summaryLevelBits,
    }
    
    // 每一层的节点所管理的最大页数的对数 [21,18,15,12,9]
    var levelLogPages = [summaryLevels]uint{
        logPallocChunkPages + 4*summaryLevelBits,
        logPallocChunkPages + 3*summaryLevelBits,
        logPallocChunkPages + 2*summaryLevelBits,
        logPallocChunkPages + 1*summaryLevelBits,
        logPallocChunkPages,
    }
    ```
  
  - ###### pallocSum：每个基数树节点存储的摘要信息，描述某个地址空间区域的属性。
    
    ```go
    type pallocSum uint64
    ```
    
    摘要信息是将三个值打包成一个 uint64，每一个位占21位，最高位用来标识是否完全为空。
    
    - start 从开始连续的可用页面数
    
    - end 从结束连续的可用页面数
    
    - max 最大的连续可用页面数。
  
  - ###### 每一级的 `[]pallocSum` 是一个切片，存储该层的所有节点。

- ##### chunks [1 << pallocChunksL1Bits]*[1 << pallocChunksL2Bits]pallocData
  
  堆内存由多个chunk组成，每个chunk内部地址是连续的
  
  chunks 是一个两级稀疏数组，主要用于管理内存分配的位图（bitmap）。每个 chunk 表示一个特定的地址空间区域，记录了该区域是否已被分配。
  
  **注：64位非windows系统只使用了L2**
  
  ```go
  type pallocData struct {
      // 内存页面分配的位图。
      pallocBits
      // 页面是否被清理的位图。
      scavenged pageBits
  }
  type pallocBits pageBits
  // pageBits 是一个位图，palloc 块中每页占一位。
  pallocChunkPages = 512
  type pageBits [pallocChunkPages / 64]uint64
  ```
  
  pallocData 是每个 chunk 的核心，它包含位图信息，记录了该 chunk 内部页面的使用情况。

- ##### searchAddr offAddr
  
  用来告诉分配器从哪里开始下一次的内存分配搜索。
  
  默认为内存的起始地址。成功分配内存后，会被更新为刚刚分配的内存块的尾地址（下一次可能的分配起始点）

- ##### start, end chunkIdx
  
  可使用的chunk index范围，内存增长时更新end

- ##### inUse addrRanges
  
  记录当前已被分配堆内存范围。
  
  ```go
  type addrRanges struct {
      // 一个按基地址排序的切片
      ranges []addrRange
  
      // 总地址空间的大小
      totalBytes uintptr
  }
  
  type addrRange struct {
      base, limit offAddr
  }
  ```
  
  - ###### 跟踪分配的堆地址范围
  
  在内存分配器中，堆地址的管理需要知道哪些地址范围已经“有效”或“处于使用中”。
  
  如果不跟踪这些信息，可能会导致分配器分配到未映射的地址，进而引发错误（如段错误）。
  
  - ###### 限制页面分配器的搜索范围
    
    通过 inUse 限制页面分配器的搜索范围，使其仅在已知有效的地址范围内查找空闲页面。
  
  - ###### 保证堆内存的连续性
  
  大多数情况下，堆内存希望尽可能分配在连续的地址空间中，以便减少碎片化，提高访问效率。
  
  - ###### 防止地址冲突
    
    确保每次分配新地址时，检查是否在 `inUse` 中，避免覆盖已分配的页面。

- ##### scav
  
  ```go
  scav struct {
      // 可用于清理的页面的块的有效索引。
      index scavengeIndex
  
      // 此清除周期在后台释放的内存量。
      releasedBg atomic.Uintptr
  
      // 这个 scavenge 周期 agered 释放的内存量。
      releasedEager atomic.Uintptr
  }
  ```

### 重要结构

##### chunk

一个 chunk 是运行时页面分配器用于跟踪和管理固定大小地址空间的单位。64位系统，每个chunk大小是 4M，包含512个page。

```go
type chunkIdx uint
// 获取某个指针的全局chunkIdx
// 地址/chunkSize
func chunkIndex(p uintptr) chunkIdx {
    return chunkIdx((p - arenaBaseOffset) / pallocChunkBytes)
}

// 指针p在其所在的chunk中的页面索引（相对偏移）。
// 它将一个全局内存地址转化为局部的chunk内页号。
func chunkPageIndex(p uintptr) uint {
    // p % pallocChunkBytes 操作可以将地址映射到当前chunk内
    return uint(p % pallocChunkBytes / pageSize)
}
```

通过地址可以获取到对应的chunkIdx，然后可以定位到对应的 L1,L2。

也可以获取到chunk内的页面索引，锁定到具体page。

```go
// 指针p在其所在的chunk中的页面索引（相对偏移）。
// 它将一个全局内存地址转化为局部的chunk内页号。
func chunkPageIndex(p uintptr) uint {
    // p % pallocChunkBytes 操作可以将地址映射到当前chunk内
    return uint(p % pallocChunkBytes / pageSize)
}
```

通过 chunkIdx 获取到对应的 chunk，然后可以通过chunk获取到page分配情况

```go
// 根据chunk index获取到对应的l1,l2；然后从chunks数组中获取到指定的chunk
func (p *pageAlloc) chunkOf(ci chunkIdx) *pallocData {
    return &p.chunks[ci.l1()][ci.l2()]
}
```

### 主要方法

##### init 初始化

```go
// mheap 初始化时调用
func (p *pageAlloc) init(mheapLock *mutex, sysStat *sysMemStat, test bool) {
    // 初始化正在使用的page，默认len=0,cap=16
    p.inUse.init(sysStat)

    // 初始化summary
    p.sysInit(test)

    // 没有空闲地址。防止未初始化完成就分配
    p.searchAddr = maxSearchAddr()

    // 初始化 scavengeIndex
    p.summaryMappedReady += p.scav.index.init(test, sysStat)
}
```

初始化 summary。summary 是隐式的基数树设计，是通过连续数组来隐式表达树结构，需要保证内存地址连续，所以先[预留内存](https://zhuanlan.zhihu.com/p/630335892)，需要的时候再实际申请。

```go
func (p *pageAlloc) sysInit(test bool) {
    // 为每个level预留内存
    for l, shift := range levelShift {
        // heapAddrBits 表示堆的地址空间位数。例如，在 64 位系统中可能是 48。
        // entries 表示当前层需要的摘要条目数
        entries := 1 << (heapAddrBits - shift)

        // 内存对齐。pallocSumBytes 是每个摘要条目占用的字节数
        b := alignUp(uintptr(entries)*pallocSumBytes, physPageSize)
        // 为当前层的摘要分配内存（虚拟地址预留，并不实际占用内存）
        r := sysReserve(nil, b)
        if r == nil {
            throw("failed to reserve page summary memory")
        }

        // 将分配的内存映射到切片
        sl := notInHeapSlice{(*notInHeap)(r), 0, entries}
        p.summary[l] = *(*[]pallocSum)(unsafe.Pointer(&sl))
    }
}
```

##### alloc 分配内存

优先从searchAddr对应的chunk中查找，如果找不到就从summary里遍历查找

```go
// 分配内存
func (p *pageAlloc) alloc(npages uintptr) (addr uintptr, scav uintptr) {
    // searchAddr 是否超出可使用的内存范围
    if chunkIndex(p.searchAddr.addr()) >= p.end {
        return 0, 0
    }

    // 直接检查 searchAddr 所在的 chunk 来判断是否能满足 npages 的需求。
    // 如果满足，就直接从该 chunk 中分配内存，避免不必要的遍历操作。
    searchAddr := minOffAddr
    // 如果剩余页数（pallocChunkPages - chunkPageIndex）大于等于 npages，就有可能直接在该 chunk 中分配。
    if pallocChunkPages-chunkPageIndex(p.searchAddr.addr()) >= uint(npages) {
        i := chunkIndex(p.searchAddr.addr())
        if max := p.summary[len(p.summary)-1][i].max(); max >= uint(npages) {
            j, searchIdx := p.chunkOf(i).find(npages, chunkPageIndex(p.searchAddr.addr()))
            addr = chunkBase(i) + uintptr(j)*pageSize
            searchAddr = offAddr{chunkBase(i) + uintptr(searchIdx)*pageSize}
            goto Found
        }
    }
    // 通过 summary 查找
    addr, searchAddr = p.find(npages)
    if addr == 0 {
        if npages == 1 {
            // 如果1个page的时候都无法找到，说明已经耗尽了内存。
            // 如果多于1个page没找到，可能是因为没有足够的连续page
            p.searchAddr = maxSearchAddr()
        }
        return 0, 0
    }
Found:
    // 将找到的地址范围标记为已分配。
    // scav 是否需要清零。如果某些页之前被释放回操作系统（以节省内存），这些页可能需要重新清零以确保安全。
    scav = p.allocRange(addr, npages)

    // 更新 searchAddr
    if p.searchAddr.lessThan(searchAddr) {
        p.searchAddr = searchAddr
    }
    return addr, scav
}
```

pallocBits 查找可用的pages，会根据不同数量使用的查找策略

```go
func (b *pallocBits) find(npages uintptr, searchIdx uint) (uint, uint) {
    if npages == 1 {
        // 分配1个（直接遍历查找）
        addr := b.find1(searchIdx)
        return addr, addr
    } else if npages <= 64 {
        // 分配<=64个
        return b.findSmallN(npages, searchIdx)
    }
    // 分配很多页
    return b.findLargeN(npages, searchIdx)
}

// 在 pallocBits 位图中查找第一个可用的页面（未分配的页面），并返回其索引。
func (b *pallocBits) find1(searchIdx uint) uint {
    _ = b[0] // lift nil check out of loop
    for i := searchIdx / 64; i < uint(len(b)); i++ {
        x := b[i]
        if ^x == 0 { // 当前块的所有页面都已分配
            continue
        }
        // 计算 ^x 的最低有效位（第一个未分配页面）的索引
        // i*64 为全局索引
        return i*64 + uint(sys.TrailingZeros64(^x))
    }
    return ^uint(0)
}

// 在一个 pallocBits 对象中查找 npages 个连续空闲页。
// 其主要目标是找到满足以下条件的起始位置：找到一段 npages 大小的连续空闲页面；返回该段连续页面的起始索引和搜索过程中的第一个空闲页面索引。
func (b *pallocBits) findSmallN(npages uintptr, searchIdx uint) (uint, uint) {
    end, newSearchIdx := uint(0), ^uint(0)
    for i := searchIdx / 64; i < uint(len(b)); i++ {
        bi := b[i]
        // 全部分配完
        if ^bi == 0 {
            end = 0
            continue
        }
        // 如果尚未找到第一个空闲位，获取当前块中第一个空闲位的索引，并更新 newSearchIdx。
        if newSearchIdx == ^uint(0) {
            newSearchIdx = i*64 + uint(sys.TrailingZeros64(^bi))
        }
        // 检查跨 chunk 的连续空闲位
        start := uint(sys.TrailingZeros64(bi))
        if end+start >= uint(npages) {
            return i*64 - end, newSearchIdx
        }
        // 检查 chunk 内部的连续空闲位
        j := findBitRange64(^bi, uint(npages))
        if j < 64 {
            return i*64 + j, newSearchIdx
        }
        // 更新尾部空闲位数（为了后面的跨 chunk 查找）
        end = uint(sys.LeadingZeros64(bi))
    }
    // 未找到满足条件的空闲段
    return ^uint(0), newSearchIdx
}

// 在 chunk 中查找连续的 n(>64) 个 page。
// 假定 npages > 64，连续空闲段会跨越至少一个对齐的 64 位边界。
func (b *pallocBits) findLargeN(npages uintptr, searchIdx uint) (uint, uint) {
    start, size, newSearchIdx := ^uint(0), uint(0), ^uint(0)
    for i := searchIdx / 64; i < uint(len(b)); i++ {
        x := b[i]
        // 该块已完全分配，跳过检查，并将 size 重置为 0。
        if x == ^uint64(0) {
            size = 0
            continue
        }
        // 如果尚未找到第一个空闲位，获取当前块中第一个空闲位的索引，并更新 newSearchIdx。
        if newSearchIdx == ^uint(0) {
            newSearchIdx = i*64 + uint(sys.TrailingZeros64(^x))
        }
        // 如果当前尚未找到连续段（size == 0）,找到当前块中连续空闲段的尾部长度，并初始化 size 和 start。
        if size == 0 {
            size = uint(sys.LeadingZeros64(x))
            start = i*64 + 64 - size
            continue
        }
        // 处理跨块的连续段
        // 如果当前块尾部的空闲位数 s 加上之前累积的空闲段长度 size 大于或等于 npages，说明已找到符合条件的连续段
        s := uint(sys.TrailingZeros64(x))
        if s+size >= uint(npages) {
            return start, newSearchIdx
        }
        // 如果块内部尾部有一些空闲位（s < 64），重新计算 size 和 start。
        if s < 64 {
            size = uint(sys.LeadingZeros64(x))
            start = i*64 + 64 - size
            continue
        }
        // 如果块内部全为空闲（s == 64），将整块加入累积段，更新 size。
        size += 64
    }
    // 如果遍历结束时，累积的空闲段长度 size 仍不足 npages，返回失败值 ^uint(0)。
    if size < uint(npages) {
        return ^uint(0), newSearchIdx
    }
    return start, newSearchIdx
}
```

summary里保存了所有页面的分配摘要，通过分层设计可以快速定位指定大小的空闲页面。

```go
// 完整的基数树搜索
func (p *pageAlloc) find(npages uintptr) (uintptr, offAddr) {
    i := 0
    firstFree := struct {
        base, bound offAddr
    }{
        base:  minOffAddr,
        bound: maxOffAddr,
    }
    // foundFree 采用给定的地址范围 [addr， addr+size） 并更新 firstFree（如果范围较窄）。
    // 输入范围必须完全包含在 firstFree 中，或者根本不与它重叠。
    // 这样，我们将记录在根级别上找到的任何免费页面的第一个摘要，如果我们进入该摘要，则会缩小范围。
    // 但是，一旦我们需要在某个关卡中迭代该摘要以找到足够大的范围，我们就会停止缩小范围。
    foundFree := func(addr offAddr, size uintptr) {
        if firstFree.base.lessEqual(addr) && addr.add(size-1).lessEqual(firstFree.bound) {
            firstFree.base = addr
            firstFree.bound = addr.add(size - 1)
        }
    }

    lastSum := packPallocSum(0, 0, 0)
    lastSumIdx := -1

nextLevel: // 标签 nextLevel 允许 continue nextLevel 跳过当前循环的剩余部分，并直接进入下一次外层 for 循环的开头，而不会重新执行整个循环。
    // 从最高层遍历所有的level
    for l := 0; l < len(p.summary); l++ {
        // 当前level包含的条目数量。root（第0层）代表所有的条目
        entriesPerBlock := 1 << levelBits[l]
        // 当前level包含page数的对数
        logMaxPages := levelLogPages[l]

        // 更新索引 i，使它指向新的层级开始的位置。
        i <<= levelBits[l]

        // 当前level包含的所有条目
        entries := p.summary[l][i : i+entriesPerBlock]

        // 通过 searchAddr 来优化迭代起始位置 j0，从而减少不必要的迭代操作。
        j0 := 0
        if searchIdx := offAddrToLevelIndex(l, p.searchAddr); searchIdx&^(entriesPerBlock-1) == i {
            j0 = searchIdx & (entriesPerBlock - 1)
        }

        // 遍历所有的条目
        var base, size uint
        for j := j0; j < len(entries); j++ {
            sum := entries[j]
            // 当前条目没有剩余空间
            if sum == 0 {
                size = 0
                continue
            }

            // 我们遇到了一个非零摘要，这意味着有空闲内存，所以更新 firstFree。
            foundFree(levelIndexToOffAddr(l, i+j), (uintptr(1)<<logMaxPages)*pageSize)

            // size加上当前数量足够分配，查找结束
            s := sum.start()
            if size+s >= uint(npages) {
                // 当前没有任何连续的空闲页，即尚未找到或开始追踪连续的空闲块。
                if size == 0 {
                    // 设置 base 为当前块中第一个页的起始地址
                    base = uint(j) << logMaxPages
                }
                // We hit npages; we're done!
                size += s
                break
            }
            // 当前条目内部包含连续的npages，从下一个level开始查找
            if sum.max() >= uint(npages) {
                // 如果是最后一层，会跳出循环，直接从 chunk 里查找
                i += j
                lastSumIdx = i
                lastSum = sum
                continue nextLevel
            }
            // 当前没有连续空闲页段（size == 0）；当前条目不是完全空闲，无法延续当前的连续空闲页段
            if size == 0 || s < 1<<logMaxPages {
                // 设置 size 为当前条目末尾的连续空闲页数
                size = sum.end()
                // 重新计算 base（当前连续空闲页段的起始页索引）
                base = uint(j+1)<<logMaxPages - size
                continue
            }
            // 当前条目为空(且不够npages)，继续查找
            size += 1 << logMaxPages
        }
        // 找到足够大的空闲页面，计算地址并返回它。
        if size >= uint(npages) {
            addr := levelIndexToOffAddr(l, i).add(uintptr(base) * pageSize).addr()
            return addr, p.findMappedAddr(firstFree.base)
        }
        // 没有找到，并且在root级别，标识内存耗尽
        if l == 0 {
            return 0, maxSearchAddr()
        }
    }

    // 无法从上层概要（summary）找到满足条件的连续空闲页段时，深入到内存块（chunk）中查找一个合适的空闲页段。
    // 表示在最后一层找到了连续的 npages 摘要信息（索引是i），现在从指定的 chunk 里查找
    ci := chunkIdx(i)
    // 块内搜索一个符合条件的连续空闲页段。
    j, searchIdx := p.chunkOf(ci).find(npages, 0)
    // 检查 j 是否为无效值（如果无效代表 summary 摘要信息有问题）
    if j == ^uint(0) {
        throw("bad summary data")
    }

    // 计算找到的空闲页段的起始物理地址
    addr := chunkBase(ci) + uintptr(j)*pageSize

    // 计算更精确的搜索地址 searchAddr
    searchAddr := chunkBase(ci) + uintptr(searchIdx)*pageSize
    foundFree(offAddr{searchAddr}, chunkBase(ci+1)-searchAddr)
    // 空闲页段的起始地址
    return addr, p.findMappedAddr(firstFree.base)
}
```

##### free 内存释放

这里的释放是将chunk里的对应设置为0（未使用），然后更新summary摘要信息

```go
func (p *pageAlloc) free(base, npages uintptr) {
    // 如果我们要释放 p.searchAddr 下的页面，更新 searchAddr。
    if b := (offAddr{base}); b.lessThan(p.searchAddr) {
        p.searchAddr = b
    }
    // 需要释放的结尾
    limit := base + npages*pageSize - 1
    // 释放 1 个 page
    if npages == 1 {
        // 直接定位 chunk 和 page 索引
        i := chunkIndex(base)
        pi := chunkPageIndex(base)
        // page 标识置为0
        p.chunkOf(i).free1(pi)
        p.scav.index.free(i, pi, 1)
    } else {
        sc, ec := chunkIndex(base), chunkIndex(limit)
        si, ei := chunkPageIndex(base), chunkPageIndex(limit)

        if sc == ec {
            // 同一个 chunk 内
            p.chunkOf(sc).free(si, ei+1-si)
            p.scav.index.free(sc, si, ei+1-si)
        } else {
            // 跨 chunk，遍历处理所有的 chunk
            p.chunkOf(sc).free(si, pallocChunkPages-si)
            p.scav.index.free(sc, si, pallocChunkPages-si)
            for c := sc + 1; c < ec; c++ {
                p.chunkOf(c).freeAll()
                p.scav.index.free(c, 0, pallocChunkPages)
            }
            p.chunkOf(ec).free(0, ei+1)
            p.scav.index.free(ec, 0, ei+1)
        }
    }
    // 更新摘要信息
    p.update(base, npages, true, false)
}
```

摘要信息对内存的分配查找非常关键，任何对内存的操作都需要更新摘要信息

更新摘要信息就是设置每一层对应节点的start,end和max信息

```go
// 如果 contig 为 true，则 update 会进行一些优化，假设 addr 和 addr+npages 之间存在连续的 allocation 或 free。
// alloc 指示执行的操作是 ALLOCATION 还是 FREE。
func (p *pageAlloc) update(base, npages uintptr, contig, alloc bool) {
    limit := base + npages*pageSize - 1
    // sc起始chunk索引，ec更新后的chunk索引
    sc, ec := chunkIndex(base), chunkIndex(limit)

    // 更新sc,ec以及他们之间的chunk的摘要信息
    if sc == ec {
        // Fast path：分配不会跨越多个 chunk
        x := p.summary[len(p.summary)-1][sc] // 存储的摘要信息
        y := p.chunkOf(sc).summarize()       // 重新计算摘要信息
        // 摘要信息没有变化，直接返回
        if x == y {
            return
        }
        // 更新当前chunk摘要信息
        p.summary[len(p.summary)-1][sc] = y
    } else if contig { // 需要做一些优化
        // 慢速连续路径：分配跨越多个块时，更新中间块的状态。
        summary := p.summary[len(p.summary)-1]

        // 更新当前chunk摘要信息
        summary[sc] = p.chunkOf(sc).summarize()

        // 更新sc(当前chunk索引)和ex(更新后的chunk索引)之间的chunk的状态
        // 如果是分配操作，将这些chunk置为不可用状态；如果是释放操作，将这些chunk置为空闲状态
        whole := p.summary[len(p.summary)-1][sc+1 : ec]
        if alloc {
            // clear 会将切片中的所有元素重置为零或默认值。(都变成0，代表start，max，end都是0，表示当前chunk没有可用的page)
            clear(whole)
        } else {
            // 置为完全空闲状态
            for i := range whole {
                whole[i] = freeChunkSum
            }
        }

        // 计算 pallocBits 中位图的摘要信息
        // 根据 bitmap 计算对应的 start,end和max
        summary[ec] = p.chunkOf(ec).summarize()
    } else {
        // 慢速通用路径：分配跨越多个块，循环更新这些块的摘要信息。
        summary := p.summary[len(p.summary)-1]
        for c := sc; c <= ec; c++ {
            summary[c] = p.chunkOf(c).summarize()
        }
    }

    // 从倒数第二层开始往上更新摘要信息（最底层的摘要上面已经更新过）
    changed := true
    for l := len(p.summary) - 2; l >= 0 && changed; l-- {
        // 本层摘要是否更新，本层更新了才需要继续往上更新
        changed = false

        logEntriesPerBlock := levelBits[l+1]
        logMaxPages := levelLogPages[l+1]

        // 需要更新的索引范围。每一层对应的范围不同，粒度越粗跨度越小
        lo, hi := addrsToSummaryRange(l, base, limit+1)

        // 迭代每个数据块，在较细的级别更新相应的摘要。
        for i := lo; i < hi; i++ {
            // children 包含了当前块的所有子块摘要
            // i<<logEntriesPerBlock 子块起始索引；(i+1)<<logEntriesPerBlock 子块结束索引
            children := p.summary[l+1][i<<logEntriesPerBlock : (i+1)<<logEntriesPerBlock]
            // 合并摘要信息
            sum := mergeSummaries(children, logMaxPages)
            old := p.summary[l][i]
            if old != sum {
                // 更新当前摘要信息，并标记需要更新更上层
                changed = true
                p.summary[l][i] = sum
            }
        }
    }
}
```

grow 内存增长

```go
// summary,scav 的地址init的时候就预留好了，现在需要将状态改成 Ready。
// chunks 的内存需要动态的分配。
func (p *pageAlloc) grow(base, size uintptr) {
    // 四舍五入到 chunks，因为我们无法处理小于 chunks 的增量。
    limit := alignUp(base+size, pallocChunkBytes)
    base = alignDown(base, pallocChunkBytes)

    // 映射和分配摘要信息，scav的内存（预留状态更新为 Ready）
    p.sysGrow(base, limit)

    // 更新后台清理器状态映射情况
    p.summaryMappedReady += p.scav.index.grow(base, limit, p.sysStat)

    // 更新可使用 chunk 范围
    firstGrowth := p.start == 0
    start, end := chunkIndex(base), chunkIndex(limit)
    if firstGrowth || start < p.start {
        p.start = start
    }
    if end > p.end {
        p.end = end
    }
    // 更新 inUse。将这段地址区间加入到 inUse
    p.inUse.add(makeAddrRange(base, limit))

    // 如果当前地址小于 searchAddr， 更新
    if b := (offAddr{base}); b.lessThan(p.searchAddr) {
        p.searchAddr = b
    }

    // 如果需要，将条目添加到 chunks 中，这是稀疏的。然后初始化位图。
    // 新增长的内存总是被认为是被清除的。将清理位图中的所有位设置为高位。
    for c := chunkIndex(base); c < chunkIndex(limit); c++ {
        // 当前所指向的 l1 为空，初始化 l1
        if p.chunks[c.l1()] == nil {
            // 为 l1 分配内存（整个l2的大小）
            const l2Size = unsafe.Sizeof(*p.chunks[0])
            r := sysAlloc(l2Size, p.sysStat)
            if r == nil {
                throw("pageAlloc: out of memory")
            }
            if !p.test {
                // 设置是否使用巨页
                // 巨页是一种内存优化技术，通过增加单个内存页的大小，减少页表项数量，从而提升性能。
                if p.chunkHugePages {
                    sysHugePage(r, l2Size)
                } else {
                    sysNoHugePage(r, l2Size)
                }
            }
            // 内存绑定到 chunks 上
            *(*uintptr)(unsafe.Pointer(&p.chunks[c.l1()])) = uintptr(r)
        }
        // 设置当前 chunk 的清理状态（整个 chunk 设置为已使用）
        p.chunkOf(c).scavenged.setRange(0, pallocChunkPages)
    }

    // 更新摘要信息
    p.update(base, size/pageSize, true, false)
}
```

##### sysGrow 系统内存增长

处理堆内存的增长，同时管理和映射与分配相关的摘要数据（summary）。

确保新增堆范围的 summary 内存已映射，并标记状态为可用（ready）。

```go
func (p *pageAlloc) sysGrow(base, limit uintptr) {
	// 将地址范围转换为指定摘要层级的索引范围。
	addrRangeToSummaryRange := func(level int, r addrRange) (int, int) {
		sumIdxBase, sumIdxLimit := addrsToSummaryRange(level, r.base.addr(), r.limit.addr())
		return blockAlignSummaryRange(level, sumIdxBase, sumIdxLimit)
	}

	// 将摘要索引范围转换为物理地址范围（用于内存映射）。
	summaryRangeToSumAddrRange := func(level, sumIdxBase, sumIdxLimit int) addrRange {
		baseOffset := alignDown(uintptr(sumIdxBase)*pallocSumBytes, physPageSize)
		limitOffset := alignUp(uintptr(sumIdxLimit)*pallocSumBytes, physPageSize)
		base := unsafe.Pointer(&p.summary[level][0])
		return addrRange{
			offAddr{uintptr(add(base, baseOffset))},
			offAddr{uintptr(add(base, limitOffset))},
		}
	}

	// 结合前两者，实现地址范围到摘要物理地址范围的转换。
	addrRangeToSumAddrRange := func(level int, r addrRange) addrRange {
		sumIdxBase, sumIdxLimit := addrRangeToSummaryRange(level, r)
		return summaryRangeToSumAddrRange(level, sumIdxBase, sumIdxLimit)
	}

	// 找到新增长范围的插入位置：
	inUseIndex := p.inUse.findSucc(base)

	// 按层级映射摘要
	for l := range p.summary {
		// 计算新地址范围需要的摘要索引范围。
		needIdxBase, needIdxLimit := addrRangeToSummaryRange(l, makeAddrRange(base, limit))
		
		// 更新摘要切片的长度。
		if needIdxLimit > len(p.summary[l]) {
			p.summary[l] = p.summary[l][:needIdxLimit]
		}

		// 计算 level l 的 summary 数组中所需的地址范围。
		need := summaryRangeToSumAddrRange(l, needIdxBase, needIdxLimit)
		
		// 新地址范围可能部分已映射，需修剪出尚未映射的部分。
		if inUseIndex > 0 {
			need = need.subtract(addrRangeToSumAddrRange(l, p.inUse.ranges[inUseIndex-1]))
		}
		if inUseIndex < len(p.inUse.ranges) {
			need = need.subtract(addrRangeToSumAddrRange(l, p.inUse.ranges[inUseIndex]))
		}
		// 在我们上面的修剪之后，可能没有什么新的东西可以映射。
		if need.size() == 0 {
			continue
		}

		// 内存区域转换为 Ready。
		sysMap(unsafe.Pointer(need.base.addr()), need.size(), p.sysStat)
		sysUsed(unsafe.Pointer(need.base.addr()), need.size(), need.size())
		// 更新已映射的摘要总量。
		p.summaryMappedReady += need.size()
	}

	// 更新后台清理器状态映射情况
	p.summaryMappedReady += p.scav.index.sysGrow(base, limit, p.sysStat)
}
```

### 总结

pageAlloc 用 chunks 和 summary 来实现高效的分配和释放，大量使用位运算优化性能。

使用 searchAddr 来加速查找，inUse和start,end来记录和限制使用的内存范围。

***注：这里没有对GC相关做说明，会在GC部分补充***
