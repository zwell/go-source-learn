### 结构体

##### note

```go
type note struct {
	key uintptr
}
```

key 字段在不同的底层实现（futex 或信号量）中有不同的用途：

- futex 实现：key 作为 uint32 类型的 key，直接参与 futex 操作（Linux）。
- 信号量 实现：key 作为 M* 类型（指向等待线程的 waitm 指针），用于信号量等待（Windows）。

note 结构是 不可重入的，即：

- 必须先调用 noteclear 进行初始化。
- notesleep 和 notewakeup 只能各有一个线程调用一次。

### 主要方法

##### notesleep

让当前的 M 进入休眠状态，直到被其他线程通过 notewakeup 唤醒。

semasleep 根据不同操作系统实现

```go
func notesleep(n *note) {
	gp := getg()
	if gp != gp.m.g0 {
		throw("notesleep not on g0")
	}
	// 创建信号量
	semacreate(gp.m)
	// 尝试获取锁
	if !atomic.Casuintptr(&n.key, 0, uintptr(unsafe.Pointer(gp.m))) {
		if n.key != locked {
			throw("notesleep - waitm out of sync")
		}
		return
	}
	// 进入休眠状态
	gp.m.blocked = true
	if *cgo_yield == nil {
		// 无限休眠
		semasleep(-1)
	} else {
		// cgo 场景（需要定期轮询以避免阻塞 cgo 的拦截器）
		const ns = 10e6
		for atomic.Loaduintptr(&n.key) == 0 {
			semasleep(ns)
			asmcgocall(*cgo_yield, nil)
		}
	}
	// 唤醒后恢复状态
	gp.m.blocked = false
}
```

##### notewakeup

唤醒一个等待信号的 Goroutine

```go
func notewakeup(n *note) {
	var v uintptr
	for {
		// 加载和设置 n.key
		v = atomic.Loaduintptr(&n.key)
		if atomic.Casuintptr(&n.key, v, locked) {
			break
		}
	}
	
	switch {
	case v == 0:
		// Nothing was waiting. Done.
	case v == locked:
		throw("notewakeup - double wakeup")
	default:
		// 唤醒线程
		semawakeup((*m)(unsafe.Pointer(v)))
	}
}
```

##### noteclear

重置 note，使其可以被 notesleep 正确等待。

```go
func noteclear(n *note) {
	n.key = 0
}
```
