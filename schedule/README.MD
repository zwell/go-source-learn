# 进程管理

Go 运行时的进程管理围绕 Goroutine（G）、机器线程（M）、处理器（P） 和调度单元（sudog）进行调度，实现高效的 M:N 调度模型。

## 目录

- [主调度流程](schedt.md)
- [Goroutine（G）](g.md)
- [机器线程（M）](m.md)
- [处理器（P）](p.md)
- [调度单元（sudog）](sudog.md)

## 简介

Go 语言的调度模型采用 G-M-P 机制，确保 Goroutine 能够高效执行，并最大限度减少线程切换的开销。

- **主调度流程**：协调 G、M 和 P 之间的任务分配，决定 Goroutine 何时运行或被挂起。
- **Goroutine（G）**：Go 语言的轻量级线程，存储执行上下文和栈信息。
- **机器线程（M）**：OS 级线程，负责执行 G，并与 P 绑定进行任务调度。
- **处理器（P）**：管理 M 和 G 之间的调度关系，控制 Goroutine 在 CPU 上的执行。
- **调度单元（sudog）**：封装 Goroutine 的等待信息，用于同步原语（如 channel 发送和接收）。

## 细节解析

### **1. 主调度流程**
Go 运行时通过 `schedule()` 实现 Goroutine 调度，主要流程如下：
1. 查找可运行的 G（本地 P 的运行队列、全局运行队列、网络轮询等）。
2. 如果找到 G，则执行；如果没有，则尝试从其他 P 窃取任务。
3. 如果仍然没有任务，则让 M 进入休眠。

### **2. G（Goroutine）**
Goroutine 代表 Go 运行时的并发执行单元，其调度状态包括：
- `_Grunnable`（可运行）
- `_Gwaiting`（等待 IO 或同步操作）
- `_Grunning`（正在运行）
- `_Gdead`（已销毁，可复用）

### **3. M（Machine，线程）**
M 代表 OS 线程，每个 M 需要绑定一个 P 才能执行 G。
- M 的生命周期由 `newm()`、`startm()` 和 `stopm()` 控制。
- M 进入休眠时，会释放 P，并等待新的任务被唤醒。

### **4. P（Processor，处理器）**
P 是调度的核心单位，最多有 `GOMAXPROCS` 个 P。
- P 维护 G 的本地运行队列。
- 负责执行 `runqget()` 和 `runqput()` 进行任务调度。
- 当 P 队列为空时，可能会从全局队列或其他 P 窃取任务。

### **5. Sudog（调度单元）**
`sudog` 结构用于存储 Goroutine 的等待信息。
- 主要用于 channel 发送/接收的阻塞队列。
- 关联 `waitq` 队列，确保 Goroutine 在同步操作时正确唤醒。
