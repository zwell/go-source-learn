### 介绍

每个 Goroutine 都有自己的栈，而一个栈由多个栈帧（stkframe）组成。这些栈帧按照调用顺序依次排列，形成调用栈（Call Stack）。

每个函数调用都会分配一个栈帧，存储其局部变量、返回地址、参数等信息。

##### 结构体

单个物理栈帧

```go
type stkframe struct {
	// 当前栈帧的函数信息
	fn funcInfo

	// 当前帧的 PC（程序计数器），即代码的执行地址
	pc uintptr
	
	// 继续执行的 PC（如果继续执行的话）
	continpc uintptr

	// 调用者的返回地址（存储在 link register 中）
	lr uintptr
	// 当前帧的栈指针（stack pointer, SP）
	sp uintptr
	// 调用者的栈指针（frame pointer, FP），即该帧的 FP
	fp uintptr
	// 局部变量的起始地址
	varp uintptr
	// 函数参数的地址
	argp uintptr
}
```


### 主要方法

##### argBytes 函数参数字节数

计算函数调用的参数帧大小（frame size），也就是该栈帧中函数的参数占用的字节数。

```go
func (frame *stkframe) argBytes() uintptr {
	// 已知参数大小，则直接返回
	if frame.fn.args != abi.ArgsSizeUnknown {
		return uintptr(frame.fn.args)
	}
	// 计算参数大小（不常见）
	argMap, _ := frame.argMapInternal()
	return uintptr(argMap.n) * goarch.PtrSize
}
```

##### argMapInternal 参数指针映射表

获取当前栈帧的参数指针映射表

```go
func (frame *stkframe) argMapInternal() (argMap bitvector, hasReflectStackObj bool) {
	f := frame.fn
	// 如果 f.args（参数大小）已知，直接计算返回
	if f.args != abi.ArgsSizeUnknown {
		argMap.n = f.args / goarch.PtrSize
		return
	}
	// 特殊情况：处理 reflect 相关的函数
	switch funcname(f) {
	case "reflect.makeFuncStub", "reflect.methodValueCall":
		// 计算 arg0（即 reflectMethodValue 的地址）
		arg0 := frame.sp + sys.MinFrameSize

		// 检查栈帧是否合法
		minSP := frame.fp
		if !usesLR {
			minSP -= goarch.PtrSize
		}
		if arg0 >= minSP {
			return bitvector{}, false // No locals, so also no stack objects
		}
		// 提取 reflectMethodValue
		hasReflectStackObj = true
		mv := *(**reflectMethodValue)(unsafe.Pointer(arg0))
		// 验证返回值是否有效
		retValid := *(*bool)(unsafe.Pointer(arg0 + 4*goarch.PtrSize))
		if mv.fn != f.entry() {
			print("runtime: confused by ", funcname(f), "\n")
			throw("reflect mismatch")
		}
		// 计算参数指针映射
		argMap = *mv.stack
		if !retValid {
			n := int32((mv.argLen &^ (goarch.PtrSize - 1)) / goarch.PtrSize)
			if n < argMap.n {
				argMap.n = n
			}
		}
	}
	return
}
```

##### getStackMap 栈帧信息

获取某个栈帧（stack frame）中局部变量（locals）和函数参数（arguments）的存活指针位图（bitvector），以及栈对象列表（stack object records）

```go
func (frame *stkframe) getStackMap(debug bool) (locals, args bitvector, objs []stackObjectRecord) {
	// 获取目标 PC
	targetpc := frame.continpc
	if targetpc == 0 {
		return
	}

	// 计算 PC 数据索引
	f := frame.fn
	pcdata := int32(-1)
	if targetpc != f.entry() {
		targetpc--
		pcdata = pcdatavalue(f, abi.PCDATA_StackMapIndex, targetpc)
	}
	if pcdata == -1 {
		pcdata = 0
	}

	// 计算局部变量的指针位图
	size := frame.varp - frame.sp
	var minsize uintptr
	switch goarch.ArchFamily {
	case goarch.ARM64:
		minsize = sys.StackAlign
	default:
		minsize = sys.MinFrameSize
	}
	if size > minsize {
		stackid := pcdata
		stkmap := (*stackmap)(funcdata(f, abi.FUNCDATA_LocalsPointerMaps))
	}

	// 获取参数的指针位图
	var isReflect bool
	args, isReflect = frame.argMapInternal()
	if args.n > 0 && args.bytedata == nil {
		stackmap := (*stackmap)(funcdata(f, abi.FUNCDATA_ArgsPointerMaps))
		if stackmap.nbit == 0 {
			args.n = 0
		} else {
			args = stackmapdata(stackmap, pcdata)
		}
	}

	// 获取栈对象列表
	if (GOARCH == "amd64" || GOARCH == "arm64" || GOARCH == "loong64" || GOARCH == "ppc64" || GOARCH == "ppc64le" || GOARCH == "riscv64") &&
		unsafe.Sizeof(abi.RegArgs{}) > 0 && isReflect {
		objs = methodValueCallFrameObjs[:]
	} else {
		p := funcdata(f, abi.FUNCDATA_StackObjects)
		if p != nil {
			n := *(*uintptr)(p)
			p = add(p, goarch.PtrSize)
			r0 := (*stackObjectRecord)(noescape(p))
			objs = unsafe.Slice(r0, int(n))
		}
	}

	return
}
```