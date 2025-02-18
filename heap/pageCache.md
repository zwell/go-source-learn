### 介绍

pageCache 是运行时的页分配器中用于缓存未分配页面的结构

pageCache 从 pageAlloc 中分配内存，清空时将内存释放到 pageAlloc

### 主要工作

- 减少对全局数据结构（如位图）的频繁访问。
- 提供线程本地的快速分配，提高分配效率。
- 在需要时，可以将未使用的页面归还到全局分配器。

### 主要属性

- ##### base uintptr

起始地址

- ##### cache uint64

空闲页面位图，固定缓存64个页面

- ##### scav uint64

### 主要方法

##### allocToCache

从 pageAlloc 中分配

```go
// 获取一个与 pageCachePages 对齐的空闲页面块，该块可能不是连续的，并返回拥有该块的 pageCache 结构
//
//go:systemstack
func (p *pageAlloc) allocToCache() pageCache {
	// 内存不足，不缓存
	if chunkIndex(p.searchAddr.addr()) >= p.end {
		return pageCache{}
	}
	c := pageCache{}
	// 下一个搜索地址指向的chunk
	ci := chunkIndex(p.searchAddr.addr())
	var chunk *pallocData
	if p.summary[len(p.summary)-1][ci] != 0 {
		// 快速路径：searchAddr 地址处或附近有空闲页面。
		chunk = p.chunkOf(ci)
		// 从chunk里查找第一个可用的page
		j, _ := chunk.find(1, chunkPageIndex(p.searchAddr.addr()))
		if j == ^uint(0) {
			throw("bad summary data")
		}
		// 初始化 pageCache
		c = pageCache{
			base: chunkBase(ci) + alignDown(uintptr(j), 64)*pageSize,
			// 初始化使用位图
			cache: ^chunk.pages64(j),
			// 初始化清理位图
			scav: chunk.scavenged.block64(j),
		}
	} else {
		// 慢速路径：searchAddr 地址那里什么都没有，所以慢速找到第一个空闲页面。
		addr, _ := p.find(1)
		if addr == 0 {
			p.searchAddr = maxSearchAddr()
			return pageCache{}
		}
		ci = chunkIndex(addr)
		chunk = p.chunkOf(ci)
		c = pageCache{
			base:  alignDown(addr, 64*pageSize),
			cache: ^chunk.pages64(chunkPageIndex(addr)),
			scav:  chunk.scavenged.block64(chunkPageIndex(addr)),
		}
	}
	
	// 将页面位设置为已分配并清除已清理的位，但请注意仅设置和清除相关位。
	cpi := chunkPageIndex(c.base)
	chunk.allocPages64(cpi, c.cache)
	chunk.scavenged.clearBlock64(cpi, c.cache&c.scav /* free and scavenged */) // 按位清除page

	// 更新摘要数据（summary）
	p.update(c.base, pageCachePages, false, true)

	// Update density statistics.
	p.scav.index.alloc(ci, uint(sys.OnesCount64(c.cache)))
	
	// 更新search地址，指向刚刚分配的内存块的尾地址
	p.searchAddr = offAddr{c.base + pageSize*(pageCachePages-1)}
	return c
}
```

##### alloc 分配内存

根据位图从pageCache中获取连续的页面

```go
func (c *pageCache) alloc(npages uintptr) (uintptr, uintptr) {
	// 缓存中没有可用的内存块
	if c.cache == 0 {
		return 0, 0
	}
	if npages == 1 {
		// c.cache 中最低有效位（即最右边的 1 位）的位置。
		i := uintptr(sys.TrailingZeros64(c.cache))
		scav := (c.scav >> i) & 1
		// 将索引为 i 的页面标记为“已分配”。
		c.cache &^= 1 << i
		// 将对应的清理标志位置为“未清理”。
		c.scav &^= 1 << i
		return c.base + i*pageSize, uintptr(scav) * pageSize
	}
	return c.allocN(npages)
}

func (c *pageCache) allocN(npages uintptr) (uintptr, uintptr) {
	// 从 c.cache 中查找连续的 npages 个空闲页面。
	i := findBitRange64(c.cache, uint(npages))
	// 没有找到
	if i >= 64 {
		return 0, 0
	}
	// 标记从索引 i 开始的连续 npages 个页面。这个掩码会将这些页面标记为已分配。
	mask := ((uint64(1) << npages) - 1) << i
	// 每个页面的清理状态。
	scav := sys.OnesCount64(c.scav & mask)
	// 将找到的连续空闲页面标记为“已分配”。
	c.cache &^= mask
	// 将对应的清理标志位置为“未清理”。
	c.scav &^= mask
	return c.base + uintptr(i*pageSize), uintptr(scav) * pageSize
}
```

#####　flush 清空并释放

```go
func (c *pageCache) flush(p *pageAlloc) {
    // 确定当前缓存的基地址和偏移
	ci := chunkIndex(c.base)
	pi := chunkPageIndex(c.base)

	// 遍历位图，处理每个页面
	for i := uint(0); i < 64; i++ {
		if c.cache&(1<<i) != 0 {
			p.chunkOf(ci).free1(pi + i)
			p.scav.index.free(ci, pi+i, 1)
		}
		if c.scav&(1<<i) != 0 {
			p.chunkOf(ci).scavenged.setRange(pi+i, 1)
		}
	}

	// 更新 searchAddr
	if b := (offAddr{c.base}); b.lessThan(p.searchAddr) {
		p.searchAddr = b
	}
	// 更新摘要信息
	p.update(c.base, pageCachePages, false, false)
	*c = pageCache{}
}
```