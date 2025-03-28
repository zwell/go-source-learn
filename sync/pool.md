### 介绍

重复利用临时对象，减少 GC 负担，提高性能。

适用于短生命周期、频繁创建和销毁的对象。

### 结构体

```go
type Pool struct {
    // 不可复制
    noCopy noCopy
    
    // 指向固定大小的每个 P（即每个 goroutine）的池的指针。
    // 实际上，这是一个指向每个 P 的池的指针数组。Go 运行时使用每个 goroutine 的局部池来存储它自己的对象，这样可以避免并发访问时的竞争条件，减少锁的开销。
    local unsafe.Pointer
    // 本地池数组的大小，即每个 P 的池的元素数量。
    localSize uintptr
    
    // 前一个周期中的受害者池的指针。
    // 受害者池用于存储从当前池中转移出的对象。在 Go 的运行时中，池的实现支持周期性的清理操作，其中一个周期的对象会被移动到另一个池（受害者池）中进行管理。
    victim unsafe.Pointer
    // 受害者池数组的大小。
    victimSize uintptr

    // 池中没有可用对象时生成新的对象。
    New func() any
}
```

### 常用方法

##### Put

优先写入到当前 P 独享的变量

```go
func (p *Pool) Put(x any) {
	// 绑定 Goroutine 到 P
	l, _ := p.pin()
	if l.private == nil {
		// private 是当前 P 独享的变量，访问开销最低
		l.private = x
	} else {
		// 如果 private 已有值，就放入 shared
		l.shared.pushHead(x)
	}
	// 解除 Goroutine 绑定
	runtime_procUnpin()
}
```

##### Get

```go
func (p *Pool) Get() any {
	// 绑定 Goroutine 到 P
	l, pid := p.pin()
	// 尝试从 private 获取对象
	x := l.private
	l.private = nil
	// 如果 private 为空，则尝试从 shared 获取
	if x == nil {
		x, _ = l.shared.popHead()
		// 如果 shared 也为空，进入 getSlow
		if x == nil {
			x = p.getSlow(pid)
		}
	}
	// 解除 P 绑定
	runtime_procUnpin()
	// 如果池子里没对象，调用 New 创建新对象
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}

// 当前 P 绑定的 poolLocal 为空时，尝试从其他 P 借对象
func (p *Pool) getSlow(pid int) any {
	// poolLocal 数组的大小（等于 GOMAXPROCS）。
	size := runtime_LoadAcquintptr(&p.localSize)
	locals := p.local                          
	// 尝试从其他 P 里偷取对象
	for i := 0; i < int(size); i++ {
		// 尝试从 shared 的尾部 (popTail()) 偷取对象，而 Get() 中是 popHead()：
		l := indexLocal(locals, (pid+i+1)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}
	
	// 尝试从 victim cache 获取对象
	// victim cache 是 sync.Pool 在 GC 之后可能存放对象的地方。
	size = atomic.LoadUintptr(&p.victimSize)
	if uintptr(pid) >= size {
		return nil
	}
	locals = p.victim
	l := indexLocal(locals, pid)
	// 从 victim cache 的 private 获取
	if x := l.private; x != nil {
		l.private = nil
		return x
	}
	// 尝试从 victim cache 的 shared 获取
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// victim cache 取不到对象，则清空 victimSize，让后续 Get() 直接跳过 victim cache，避免不必要的检查。
	atomic.StoreUintptr(&p.victimSize, 0)

	return nil
}
```

##### pin

将当前的 Goroutine 固定到某个 P，并禁用抢占

```go
func (p *Pool) pin() (*poolLocal, int) {
	// 固定当前 goroutine 到 P
	pid := runtime_procPin()
	// 获取本地池的大小
	s := runtime_LoadAcquintptr(&p.localSize)
	// 获取本地池
	l := p.local
	// 当前 P 已经在有效范围内
	if uintptr(pid) < s {
		// 获取到该 P 关联的 poolLocal
		return indexLocal(l, pid), pid
	}
	// 慢速路径
	return p.pinSlow()
}

func (p *Pool) pinSlow() (*poolLocal, int) {
	// 解除 Goroutine 与处理器 P 的绑定，使其能够获取锁。
	runtime_procUnpin()
	// 获取全局锁
	allPoolsMu.Lock()
	defer allPoolsMu.Unlock()
	// 重新绑定当前 Goroutine 到 P
	pid := runtime_procPin()
	s := p.localSize
	l := p.local
	// 如果 pid 小于 p.localSize，说明 p.local 仍然有效
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	// 如果 p.local 为空，将 p 加入 allPools
	if p.local == nil {
		allPools = append(allPools, p)
	}
	// 重新分配 p.local
	size := runtime.GOMAXPROCS(0)
	local := make([]poolLocal, size)
	// 存储新的 p.local 和 p.localSize
	atomic.StorePointer(&p.local, unsafe.Pointer(&local[0]))
	runtime_StoreReluintptr(&p.localSize, uintptr(size)) 
	return &local[pid], pid
}
```

### 使用案例

- [gin](https://github.com/gin-gonic/gin/blob/8763f33c65f7df8be5b9fe7504ab7fcf20abb41d/gin.go#L625)

