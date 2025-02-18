### 介绍

unwinder 是 Go 运行时的栈帧遍历器，用于 遍历 goroutine 调用栈，

##### 应用场景

- Panic 追踪：当发生 panic 时，unwinder 可以展开调用栈，找到 panic 发生的位置。
- 性能分析（Profiler）：Go 运行时可以使用 unwinder 获取当前 goroutine 的调用栈，以进行采样分析（如 pprof）。
- 调试（Debugger）：unwinder 可用于生成栈跟踪（stack trace），帮助开发者调试代码。
- GC 相关的栈扫描（Stack Scanning）:
  - Go GC 需要扫描栈，找到所有存活对象的指针。
  - unwinder 可以辅助 GC 解析调用栈，帮助识别栈上的局部变量指针。

##### 结构体

```go
type unwinder struct {
	// 当前栈帧信息，包含 PC（程序计数器）、SP（栈指针）、LR（返回地址）等
	frame stkframe
	
	// 当前正在遍历的 goroutine（g 对象），用 `guintptr` 存储以避免写屏障
	g guintptr
	
	// cgo 相关的栈上下文索引，表示下一个 cgo 栈帧在 `g.cgoCtxt` 中的位置
	cgoCtxt int
	
	// 记录调用者的函数 ID，用于区分不同的运行时函数
	calleeFuncID abi.FuncID
	
    // 记录 `unwinder` 运行时的标志位，控制遍历行为
	flags unwindFlags
}
```