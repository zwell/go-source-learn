### 介绍

#### 互斥锁的两种模式

- 正常模式（Normal Mode）

  - 等待的 Goroutine 按 FIFO（先进先出）顺序排队
    
    备注：

    - 被唤醒的 Goroutine 并不直接拥有锁，而是和新到达的 Goroutine 竞争。

    - 新来的 Goroutine 通常有优势，因为它们已经在 CPU 上运行，不需要从休眠状态恢复，因此更容易获得锁。

    - 如果被唤醒的 Goroutine 竞争失败，它会被放到等待队列的前面（LIFO 方式，保证下次优先获取锁）。

  - 如果 Goroutine 超过 1 毫秒都无法获取锁，它会触发“饥饿模式”。

- 饥饿模式（Starvation Mode）
  
    在这个模式下，锁的所有权是直接从解锁的 Goroutine 交给队列前面的 Goroutine：

    新来的 Goroutine 不能直接获取锁，即使它看起来是空闲的，它们也不会竞争，而是自动排到等待队列的末尾。

### 结构体

```go
type Mutex struct {
    // 用于存储 Mutex 的状态信息，通过位运算来管理不同的标志位。
    state int32 
    // 用于管理 Goroutine 的等待队列，实际上是一个信号量
    sema uint32
}
```

#### 重要常量
```go
// 锁是否被持有
mutexLocked = 1 << iota
// 是否有 Goroutine 被唤醒
mutexWoken
// 是否处于饥饿模式
mutexStarving
// 等待者计数的起始位，也就是从第4个bit开始
mutexWaiterShift = iota
```

### 常用方法

##### lock

如果无法获取锁，说明存在竞争

1、如果当前没有唤醒者且不处于饥饿模式，将当前的 Goroutine 设为唤醒者，优先获取锁。尝试自旋等待

2、如果等到超过1毫秒，会设置饥饿标记，并在下一次获取失败时（被唤醒，并且又获取失败）进入饥饿模式

```go
func (m *Mutex) Lock() {
    // 快速路径，当前state为空（没有被其他Goroutine持有，没有等待者）
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return
    }
    // 慢速路径
    m.lockSlow()
}

func (m *Mutex) lockSlow() {
    // 当前 goroutine 等待开始的时间
    var waitStartTime int64
    // 是否进入饥饿模式
    starving := false
    // 该 goroutine 是否被唤醒
    awoke := false
    // 自旋次数
    iter := 0
    // 当前锁的状态
    old := m.state
    for {
        // 如果锁已被占用，未处于饥饿模式，并且允许自旋，则进行自旋
        // mutexLocked == 1 表示锁已经被占用。
        // mutexStarving == 0 表示未进入饥饿模式。
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // 如果当前 Goroutine 没有被唤醒，并且没有其他 Goroutine 被唤醒，并且存在等待者
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                // 设置 mutexWoken 标志位，通知 Unlock 不再唤醒其他 Goroutine，避免不必要的调度开销。
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            // 执行自旋操作
            runtime_doSpin()
            // 递增自旋次数，避免无限自旋。
            iter++
            // 重新读取 Mutex 的状态
            old = m.state
            continue
        }
        // 自旋结束，或者锁已经释放，或者处于饥饿模式

        new := old
        // 如果锁未进入饥饿模式，尝试获取锁
        if old&mutexStarving == 0 {
            // 添加 mutexLocked 标志
            new |= mutexLocked
        }
        // 如果锁已经被占用或者进入饥饿模式，需要将等待者数量 +1（因为当前 Goroutine 需要排队）。
        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift
        }
        // 如果当前 Goroutine 进入了饥饿模式，并且锁仍然被占用，则强制锁进入 mutexStarving 模式。
        // 在这个模式下，新来的 Goroutine 不再竞争锁，而是排队等待，防止长时间等待的 Goroutine 饿死。
        if starving && old&mutexLocked != 0 {
            // 添加 mutexStarving 标志
            new |= mutexStarving
        }
        // 清除唤醒标志
        if awoke {
            // 清除 mutexWoken 标志
            new &^= mutexWoken
        }
        // 更新 Mutex 状态
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            // 如果之前没有持有锁和没有处于饥饿模式，获取锁成功。
            if old&(mutexLocked|mutexStarving) == 0 {
                break
            }
            // 如果 waitStartTime 已经有值，说明之前已经等待过了，就采用 LIFO（后进先出）。
            queueLifo := waitStartTime != 0
            // 记录当前时间，用于计算是否进入饥饿模式。
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime()
            }
            // 阻塞当前 Goroutine。
            runtime_SemacquireMutex(&m.sema, queueLifo, 2)
            // 如果等待时间超过 starvationThresholdNs（默认 1ms），则进入饥饿模式。
            // 此时并没有进入饥饿模式，如果这一次还没有获取到锁，才会进入饥饿模式。
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            // 修正锁的状态
            old = m.state
            // 是否处于饥饿模式
            if old&mutexStarving != 0 {
                // mutexLocked 置 0（解锁）。饥饿模式下，unlock不会执行解锁，而是将锁的所有权直接交给等待队列中的第一个 Goroutine。
                // mutexWaiterShift 计数减 1。
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
                // 是否需要退出饥饿模式
                // !starving：如果当前 Goroutine 没有真正进入饥饿状态，说明饥饿模式不需要持续。
                // old>>mutexWaiterShift == 1：如果 当前是最后一个等待者（即只剩下当前 Goroutine 在等待），那么不需要继续保持饥饿模式。
                if !starving || old>>mutexWaiterShift == 1 {
                    // 清除 mutexStarving 标志
                    delta -= mutexStarving
                }
                // 状态更新
                atomic.AddInt32(&m.state, delta)
                break
            }
            // 重置
            awoke = true
            iter = 0
        } else {
            // 更新失败，重置状态
            old = m.state
        }
    }
}
```

##### Unlock

1、如果不处于饥饿模式，解锁，并尝试唤醒一个等待的 Goroutine

2、如果处于饥饿模式，直接唤醒优先级最高的 Goroutine（一般情况下是当前设置了饥饿标记的 Goroutine）

```go
func (m *Mutex) Unlock() {
    // 删除锁标记
    new := atomic.AddInt32(&m.state, -mutexLocked)
    // =0代表没有等待者和唤醒者，且不处于饥饿模式
    if new != 0 {
        m.unlockSlow(new)
    }
}

func (m *Mutex) unlockSlow(new int32) {
    if new&mutexStarving == 0 {
        // 不处于饥饿模式
        old := new
        for {
            // 如果没有等待的 Goroutine，或者已经有 Goroutine 被唤醒或获取了锁，则无需唤醒新的 Goroutine
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                return
            }
            // 减少等待者的数量
            // 设置 mutexWoken 标志，表示有一个等待的 Goroutine 被唤醒
            new = (old - 1<<mutexWaiterShift) | mutexWoken
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                // 唤醒一个等待的 Goroutine。
                runtime_Semrelease(&m.sema, false, 2)
                return
            }
            old = m.state
        }
    } else {
        // 在饥饿模式下，我们不直接解锁 mutex。相反，我们通过调用 runtime_Semrelease 将锁的所有权直接交给等待
        // 队列中的第一个 Goroutine（可能不是设置饥饿模式的Goroutine（如多个 Goroutine 同时触发饥饿模式、锁的状态更新延迟、调度顺序等））。
        // 第二个参数 true 表示传递锁的所有权给下一个等待的 Goroutine，而不是唤醒一个普通的 Goroutine。
        runtime_Semrelease(&m.sema, true, 2)
    }
}
```