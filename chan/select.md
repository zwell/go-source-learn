### 介绍

在多个 channel 操作中进行多路复用：即在多个 send、receive 和 default 分支中选择一个可以立即执行的操作。

#### 阻塞 vs 非阻塞

- 阻塞的 select

如果所有 case 都不能立即执行，那么这个 select 会让当前 goroutine 阻塞，直到某个 case 能执行。

```go
select {
case msg := <-ch1:
    fmt.Println("received", msg)
}
```

- 非阻塞的 select

如果其他 case 都没法立刻执行（比如通道没数据）， 就立刻走 default 分支，不阻塞。

```go
select {
case msg := <-ch1:
    fmt.Println("received", msg)
default:
    fmt.Println("no channel ready, skip")
}
```

### 主要方法

select 关键字的实现

返回选中的哪一个，以及是否成功接收（如果是 recv）

```go
func reflect_rselect(cases []runtimeSelect) (int, bool) {
	if len(cases) == 0 {
		block()
	}
	// 每个 case 对应的 channel 操作
	sel := make([]scase, len(cases))
	// 原始索引，方便返回正确的 case 位置
	orig := make([]int, len(cases))
	// send 和 recv case 数量
	nsends, nrecvs := 0, 0
	// default case 索引（如果存在）
	dflt := -1
	// 遍历 cases，填充 sel
	for i, rc := range cases {
		var j int
		switch rc.dir {
		case selectDefault:
			dflt = i
			continue
		case selectSend:
			j = nsends
			nsends++
		case selectRecv:
			nrecvs++
			j = len(cases) - nrecvs
		}

		sel[j] = scase{c: rc.ch, elem: rc.val}
		orig[j] = i
	}

	// 如果只有 default，直接返回
	if nsends+nrecvs == 0 {
		return dflt, false
	}

	// 如果 sel 里有 default，就要压缩数据
	if nsends+nrecvs < len(cases) {
		copy(sel[nsends:], sel[len(cases)-nrecvs:])
		copy(orig[nsends:], orig[len(cases)-nrecvs:])
	}

	order := make([]uint16, 2*(nsends+nrecvs))

	// 处理 select
	chosen, recvOK := selectgo(&sel[0], &order[0], pc0, nsends, nrecvs, dflt == -1)

	// 还原原始 case 索引
	if chosen < 0 {
		chosen = dflt
	} else {
		chosen = orig[chosen]
	}
	return chosen, recvOK
}
```

```go
func selectgo(cas0 *scase, order0 *uint16, pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool) {
	gp := getg()
	if debugSelect {
		print("select: cas0=", cas0, "\n")
	}
	
	// select 语句最多支持 65536 个 case
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	// 随机轮询顺序（pollorder）和锁顺序
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

	// case 总数
	ncases := nsends + nrecvs
	// 取 cas1 的前 ncases 个作为 select 语句的 case 处理数组
	scases := cas1[:ncases:ncases]
	// 轮询的顺序
	pollorder := order1[:ncases:ncases]
	// 锁顺序
	lockorder := order1[ncases:][:ncases:ncases]

	// 计算 pollorder 轮询顺序
	norder := 0
	allSynctest := true
	for i := range scases {
		cas := &scases[i]

		// 如果 case 为空，则跳过，并设置 cas.elem = nil 以允许 GC 释放内存
		if cas.c == nil {
			cas.elem = nil // allow GC
			continue
		}

		// 处理定时器 case
		if cas.c.timer != nil {
			cas.c.timer.maybeRunChan()
		}

		// 生成 pollorder
		j := cheaprandn(uint32(norder + 1)) // 随机选择一个 pollorder[j] 进行交换
		pollorder[norder] = pollorder[j]
		pollorder[j] = uint16(i)
		norder++
	}
	// 修正 pollorder 和 lockorder 的长度
	pollorder = pollorder[:norder]
	lockorder = lockorder[:norder]

	waitReason := waitReasonSelect
	
	// 构建最大堆，根节点存储 sortkey() 最大的 channel
	for i := range lockorder {
		j := i
		// 每次插入一个新元素，它从 j = i 开始，一直向上比较 sortkey()，如果父节点比当前节点小，就交换它们，直到找到合适位置。
		c := scases[pollorder[i]].c
		for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
			k := (j - 1) / 2
			lockorder[j] = lockorder[k]
			j = k
		}
		lockorder[j] = pollorder[i]
	}
	// 堆排序
	// 依次将最大值放到 lockorder[i]，然后从 0 ~ i-1 重新调整为最大堆。
	for i := len(lockorder) - 1; i >= 0; i-- {
		o := lockorder[i]
		c := scases[o].c
		lockorder[i] = lockorder[0]
		j := 0
		for {
			k := j*2 + 1
			if k >= i {
				break
			}
			if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
				k++
			}
			if c.sortkey() < scases[lockorder[k]].c.sortkey() {
				lockorder[j] = lockorder[k]
				j = k
				continue
			}
			break
		}
		lockorder[j] = o
	}

	// 先对 select 语句涉及的所有通道加锁，保证并发安全（避免竞态条件）。
	// lockorder 是之前计算好的通道加锁顺序，防止死锁。
	sellock(scases, lockorder)

	var (
		sg     *sudog
		c      *hchan
		k      *scase
		sglist *sudog
		sgnext *sudog
		qp     unsafe.Pointer
		nextp  **sudog
	)

	// pollorder 遍历，寻找可用 case
	var casi int
	var cas *scase
	var caseSuccess bool
	var caseReleaseTime int64 = -1
	var recvOK bool
	for _, casei := range pollorder {
		casi = int(casei)
		cas = &scases[casi]
		c = cas.c

		// 如果是接收操作
		if casi >= nsends {
			// 队列已满（发送者被阻塞）。接收数据，并将发送者数据写入队列
			sg = c.sendq.dequeue()
			if sg != nil {
				goto recv
			}
			// 缓冲区有数据，直接获取
			if c.qcount > 0 {
				goto bufrecv
			}
			// 通道已关闭
			if c.closed != 0 {
				goto rclose
			}
		} else {
			// 通道已关闭
			if c.closed != 0 {
				goto sclose
			}
			// 队列为空（接收者被阻塞）。发送数据给接收者
			sg = c.recvq.dequeue()
			if sg != nil {
				goto send
			}
			// 缓冲区未满，直接写入数据
			if c.qcount < c.dataqsiz {
				goto bufsend
			}
		}
	}

	// 如果所有 case 都无法立即执行
	// block = false（select 语句包含 default）：立即返回 -1，表示没有可执行的 case。
	if !block {
		selunlock(scases, lockorder)
		casi = -1
		goto retc
	}

	// 把当前 Goroutine 加入 select 相关的通道队列，等待被唤醒。
	if gp.waiting != nil {
		throw("gp.waiting != nil")
	}
	nextp = &gp.waiting
	// 依次遍历 lockorder，创建 sudog 结构体
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		c = cas.c
		sg := acquireSudog()
		sg.g = gp
		sg.isSelect = true
		// 记录需要传递的值（如果是 send case）
		sg.elem = cas.elem
		sg.releasetime = 0
		if t0 != 0 {
			sg.releasetime = -1
		}
		// 绑定通道
		sg.c = c
		// 将 sudog 添加到 Goroutine 的 waiting 链表
		*nextp = sg
		nextp = &sg.waitlink

		// 把 sudog 挂到通道的等待队列
		if casi < nsends {
			c.sendq.enqueue(sg)
		} else {
			c.recvq.enqueue(sg)
		}

		// 处理 timer 通道
		if c.timer != nil {
			blockTimerChan(c)
		}
	}

	// 挂起 Goroutine，等待 select 事件
	gp.param = nil
	gp.parkingOnChan.Store(true)
	// 挂起
	gopark(selparkcommit, nil, waitReason, traceBlockSelect, 1)
	gp.activeStackChans = false

    // send，recv 会唤醒 gp
    // 这个时候数据已经复制完成了
    // 如果是 recv case，那么 发送方唤醒你时已经完成了 send → recv 的数据复制
    // 如果是 send case，那么 接收方唤醒你时已经完成了 recv ← send 的数据复制

	// 重新锁定 select 相关通道
	sellock(scases, lockorder)

	// 读取 gp.param，获取成功匹配的 case
	gp.selectDone.Store(0)
	sg = (*sudog)(gp.param)
	gp.param = nil
	
	// 当 Goroutine 被唤醒时，移除所有未成功的 case，只保留匹配成功的 case。
	casi = -1
	cas = nil
	caseSuccess = false
	sglist = gp.waiting
	// 遍历 gp.waiting，清理未匹配的 case
	for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
		sg1.isSelect = false
		sg1.elem = nil
		sg1.c = nil
	}
	gp.waiting = nil

	// 解除所有未匹配的 case
	for _, casei := range lockorder {
		k = &scases[casei]
		// 解除 timer 通道的阻塞
		if k.c.timer != nil {
			unblockTimerChan(k.c)
		}
		if sg == sglist {
			// 这个 sudog 是唤醒我们的那个 sudog，记录成功的 case
			casi = int(casei)
			cas = k
			caseSuccess = sglist.success
			if sglist.releasetime > 0 {
				caseReleaseTime = sglist.releasetime
			}
		} else {
			// 没有匹配成功，从通道队列移除
			c = k.c
			if int(casei) < nsends {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		// 释放 sudog
		releaseSudog(sglist)
		sglist = sgnext
	}

	if cas == nil {
		throw("selectgo: bad wakeup")
	}

	// 成功匹配的通道
	c = cas.c

	// 发送操作
	if casi < nsends {
		// caseSuccess == false，说明通道关闭了，进入 sclose 处理逻辑。
		if !caseSuccess {
			goto sclose
		}
	} else {
		recvOK = caseSuccess
	}

	// 释放 select 相关的通道锁
	selunlock(scases, lockorder)
	goto retc

bufrecv:// 从 buf 接收数据
	recvOK = true
	qp = chanbuf(c, c.recvx)
	// 复制数据
	if cas.elem != nil {
		typedmemmove(c.elemtype, cas.elem, qp)
	}
	// 清空内存
	typedmemclr(c.elemtype, qp)
	// 更新接收索引和元素数量
	c.recvx++
	if c.recvx == c.dataqsiz {
		c.recvx = 0
	}
	c.qcount--
	selunlock(scases, lockorder)
	goto retc

bufsend:// 向 buf 发送数据
	// 复制数据
	typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
	// 更新发送索引和元素数量
	c.sendx++
	if c.sendx == c.dataqsiz {
		c.sendx = 0
	}
	c.qcount++
	selunlock(scases, lockorder)
	goto retc

recv:// 从发送者接收数据	
	recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	recvOK = true
	goto retc

rclose:// 从已关闭的通道读取数据	
	selunlock(scases, lockorder)
	recvOK = false
	if cas.elem != nil {
		typedmemclr(c.elemtype, cas.elem)
	}
	goto retc

send:// 向接收者发送数据
	send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	goto retc

retc:
	if caseReleaseTime > 0 {
		blockevent(caseReleaseTime-t0, 1)
	}
	return casi, recvOK

sclose:// 向已关闭的通道发送数据
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
}
```