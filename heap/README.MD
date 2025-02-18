# 内存管理

Go 语言的内存管理由运行时（runtime）负责，实现了高效的内存分配、回收和管理机制。主要涉及以下关键组件：

## 目录

- [堆管理（heap）](heap.md)
- [页面分配器（pageAlloc）](pageAlloc.md)
- [中心缓存（mcentral）](mcentral.md)
- [本地缓存（mcache）](mcache.md)
- [内存区域（arena）](heapArena.md)
- [内存块（span）](mspan.md)
- [页面缓存（pageCache）](pageCache.md)

## 简介

Go 运行时的内存管理采用 **分层设计**，结合了 **堆管理（heap）、页面分配（pageAlloc）、中心缓存（mcentral）、本地缓存（mcache）以及内存区域（arena）** 来优化内存分配和回收。

- **堆管理（heap）**：管理整个 Go 运行时的堆空间，负责对象的动态分配和垃圾回收。
- **页面分配器（pageAlloc）**：按页分配内存，并管理可用页和已分配页的状态。
- **中心缓存（mcentral）**：负责多个 `P` 共享的中等大小对象的分配和回收。
- **本地缓存（mcache）**：每个 `P` 持有一个 `mcache`，用于存储小对象，以减少锁争用。
- **内存区域（arena）**：Go 直接向操作系统申请大块内存，并按需划分给 `heap`、`mcentral` 和 `mcache`。

每个模块都有其独立的职责，同时协同工作，以保证 Go 运行时的内存管理高效、可扩展。
