### 介绍

mspan 是 Go 运行时内存管理中的核心数据结构，用于表示堆内存中的一段连续区域（span）。

它在分配器和垃圾回收器中扮演了桥梁角色，用于管理小对象的分配和回收。

### 主要属性

- ##### state mSpanStateBox

  当前 span 状态

  - mSpanInUse: 该 span 正在使用中（没有被回收）。
  - mSpanDead: 内存块不再使用或已经被释放。
  - mSpanManual: 该 span 手动管理，不属于垃圾回收堆管理的一部分。

- ##### spanclass spanClass

  当前 span 所属类别（根据对象大小）

- ##### elemsize uintptr

  span 中对象大小

- ##### startAddr uintptr

  span 的起始地址

- ##### npages uintptr

  包含的页面数量

- ##### freeindex uint16

  空闲对象 index，指向 span 中的下一个空闲对象

- ##### nelems uint16

  span 中的对象数量(已分配+未分配)

- ##### allocCount uint16

  已分配对象数量

- ##### allocBits *gcBits

  当前 mspan 中每个字对象的分配状态

```go
// 分配/标记位图（bitmap），用于记录内存中每个对象块的状态（分配或标记）。
// 只存储了起点，通过偏移访问更多的数据。
type gcBits struct {
    x uint8
}
```

- ##### sweepgen uint32

  当前清扫版本，每次GC自增2

- ##### allocBits *gcBits

  当前 mspan 中每个字对象的分配状态

### 关联结构体

##### type spanAllocType uint8

span的类型

- spanAllocHeap 堆分配
- spanAllocStack 栈分配
- spanAllocPtrScalarBits 指针或标量位

  用于垃圾回收时标记对象的指针或标量字段，帮助 GC 跟踪对象图。

- spanAllocWorkBuf 工作缓冲区

```go
// heap里的span是自动维护（GC）
func (s spanAllocType) manual() bool {
	return s != spanAllocHeap
}
```

##### type spanClass uint8

组合值，包含了内存块的大小类别（sizeclass）和是否不需要扫描（noscan）的信息

```go
const (
    // 总共有多少种 span 类别，136
    numSpanClasses = _NumSizeClasses << 1
    // <16字节
    tinySpanClass = spanClass(tinySizeClass<<1 | 1)
)
```

```go
// 创建 spanClass
func makeSpanClass(sizeclass uint8, noscan bool) spanClass {
    // 将 sizeclass 左移 1 位，为 noscan 标志腾出空间
    // 将 noscan 转换为整数（0 或 1），并与左移后的 sizeclass 进行按位或操作 
    return spanClass(sizeclass<<1) | spanClass(bool2int(noscan))
}

// sizeclass 信息
func (sc spanClass) sizeclass() int8 {
	return int8(sc >> 1)
}

// noscan 信息
func (sc spanClass) noscan() bool {
	return sc&1 != 0
}
```

### 主要方法

##### init 初始化

```go
func (span *mspan) init(base uintptr, npages uintptr) {
    span.startAddr = base
    span.npages = npages
    span.state.set(mSpanDead)
}
```

初始化会设置起始地址，分配的页面数量

### 总结

mcentral相当于全局的span缓存中心，mcache会从这里申请和释放span。

partial和full的设计，用来高效管理和分类span的分配和回收状态。

使用两个列表来配合GC的工作，支持并行GC和分配。