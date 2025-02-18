### 介绍

arena 是堆分配的基础单位，它是一块大的连续内存区域

### 主要属性

- ##### spans [pagesPerArena]*mspan

  arena 里每个 page 对应的 mspan
  
  64位系统pagesPerArena=8192，即有8192个页面

- ##### pageInUse [pagesPerArena / 8]uint8

  哪些页被分配给了处于 mSpanInUse 状态的 span

- ##### zeroedBase uintptr

  第一个未使用的字节的偏移量（相对于该 arena 的起始地址）

  在分配内存时，会尝试从当前 zeroedBase 开始
