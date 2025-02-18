### 介绍

管理 Goroutine 在通道（channel）通信、锁等待等情况下的挂起和唤醒

Goroutine 在等待特定事件（比如 channel 发送/接收、信号量等待等）时的一个“排队元素”。

***简单理解：sudog 是 Goroutine 的“预约牌”，挂起时存进去，唤醒时取出来。***

### 结构体

```go
type sudog struct {
	// 关联的 G
	g *g

	// 等待列表中的下一个 sudog
	next *sudog
	// 等待列表中的前一个 sudog
	prev *sudog
	// 通道操作的数据元素（可能指向栈）
	elem unsafe.Pointer
	
	// 是否参与了 `select` 操作
	isSelect bool
	
	// 通道操作是否成功（`true` 表示成功，`false` 表示通道关闭等失败场景）
	success bool
	
	// 等待队列中非头部节点的数量（仅在队列头有效）
	waiters uint16

	// 树或链表结构，用于信号量或其他复杂等待队列的实现
	// 二叉树中的父节点（用于 `semaRoot`）
	parent *sudog
	// 等待列表或信号量队列中的下一个节点
	waitlink *sudog 
	// 信号量队列的尾节点（用于 `semaRoot`）
	waittail *sudog
	// 关联的通道对象
	c *hchan
```