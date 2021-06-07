![](https://img.halfrost.com/Blog/ArticleImage/147_0.jpg)

# 深入研究 Go interface 底层实现

接口是高级语言中的一个规约，是一组方法签名的集合。Go 的 interface 是非侵入式的，具体类型实现 interface 不需要在语法上显式的声明，只需要具体类型的方法集合是 interface 方法集合的超集，就表示该类实现了这一 interface。编译器在编译时会进行 interface 校验。interface 和具体类型不同，它不能实现具体逻辑，也不能定义字段。

在 Go 语言中，interface 和函数一样，都是“第一公民”。interface 可以用在任何使用变量的地方。可以作为结构体内的字段，可以作为函数的形参和返回值，可以作为其他 interface 定义的内嵌字段。interface 在大型项目中常常用来解耦。在层与层之间用 interface 进行抽象和解耦。由于 Go interface 非侵入的设计，使得抽象出来的代码特别简洁，这也符合 Go 语言设计之初的哲学。除了解耦以外，还有一个非常重要的应用，就是利用 interface 实现伪泛型。利用空的 interface 作为函数或者方法参数能够用在需要泛型的场景里。

interface 作为 Go 语言类型系统的灵魂，Go 语言实现多态和反射的基础。新手对其理解不深刻的话，常常会犯下面这个错误：

```go
func main() {
	var x interface{} = nil
	var y *int = nil
	interfaceIsNil(x)
	interfaceIsNil(y)
}

func interfaceIsNil(x interface{}) {
	if x == nil {
		fmt.Println("empty interface")
		return
	}
	fmt.Println("non-empty interface")
}
```

笔者第一次接触到这个问题是强转了 gRPC 里面的一个 interface，然后在外面判断它是否为 nil。结果出 bug 了。当初如果了解对象强制转换成 interface 的时候，不仅仅含有原来的对象，还会包含对象的类型信息，也就不会出 bug 了。

本文将会详细分解 interface 所有底层实现。

> 以下代码基于 Go 1.16 


## 一. 数据结构

### 1. 非空 interface 数据结构
非空的 interface 初始化的底层数据结构是 iface，稍后在汇编代码中能验证这一点。

```go
type iface struct {
	tab  *itab
	data unsafe.Pointer
}
```

tab 中存放的是类型、方法等信息。data 指针指向的 iface 绑定对象的原始数据的副本。这里同样遵循 Go 的统一规则，值传递。tab 是 itab 类型的指针。

```go
// layout of Itab known to compilers
// allocated in non-garbage-collected memory
// Needs to be in sync with
// ../cmd/compile/internal/gc/reflect.go:/^func.WriteTabs.
type itab struct {
	inter *interfacetype
	_type *_type
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```

itab 中包含 5 个字段。inner 存的是 interface 自己的静态类型。\_type 存的是 interface 对应具体对象的类型。itab 中的 \_type 和 iface 中的 data 能简要描述一个变量。\_type 是这个变量对应的类型，data 是这个变量的值。这里的 hash 字段和 \_type 中存的 hash 字段是完全一致的，这么做的目的是为了类型断言(下文会提到)。fun 是一个函数指针，它指向的是具体类型的函数方法。虽然这里只有一个函数指针，但是它可以调用很多方法。在这个指针对应内存地址的后面依次存储了多个方法，利用指针偏移便可以找到它们。  

由于 Go 语言是强类型语言，编译时对每个变量的类型信息做强校验，所以每个类型的元信息要用一个结构体描述。再者 Go 的反射也是基于类型的元信息实现的。\_type 就是所有类型最原始的元信息。

```go
// Needs to be in sync with ../cmd/link/internal/ld/decodesym.go:/^func.commonsize,
// ../cmd/compile/internal/gc/reflect.go:/^func.dcommontype and
// ../reflect/type.go:/^type.rtype.
// ../internal/reflectlite/type.go:/^type.rtype.
type _type struct {
	size       uintptr // 类型占用内存大小
	ptrdata    uintptr // 包含所有指针的内存前缀大小
	hash       uint32  // 类型 hash
	tflag      tflag   // 标记位，主要用于反射
	align      uint8   // 对齐字节信息
	fieldAlign uint8   // 当前结构字段的对齐字节数
	kind       uint8   // 基础类型枚举值
	equal func(unsafe.Pointer, unsafe.Pointer) bool // 比较两个形参对应对象的类型是否相等
	gcdata    *byte    // GC 类型的数据
	str       nameOff  // 类型名称字符串在二进制文件段中的偏移量
	ptrToThis typeOff  // 类型元信息指针在二进制文件段中的偏移量
}
```

有 3 个字段需要解释一下：

- kind，这个字段描述的是如何解析基础类型。在 Go 语言中，基础类型是一个枚举常量，有 26 个基础类型，如下。枚举值通过 kindMask 取出特殊标记位。

```go
const (
	kindBool = 1 + iota
	kindInt
	kindInt8
	kindInt16
	kindInt32
	kindInt64
	kindUint
	kindUint8
	kindUint16
	kindUint32
	kindUint64
	kindUintptr
	kindFloat32
	kindFloat64
	kindComplex64
	kindComplex128
	kindArray
	kindChan
	kindFunc
	kindInterface
	kindMap
	kindPtr
	kindSlice
	kindString
	kindStruct
	kindUnsafePointer

	kindDirectIface = 1 << 5
	kindGCProg      = 1 << 6
	kindMask        = (1 << 5) - 1
)
```

- str 和 ptrToThis，对应的类型是 nameoff 和 typeOff。这两个字段的值是在链接器段合并和符号重定向的时候赋值的。
![](https://img.halfrost.com/Blog/ArticleImage/147_1.png)
链接器将各个 .o 文件中的段合并到输出文件，会进行段合并，有的放入 .text 段，有的放入 .data 段，有的放入 .bss 段。name 和 type 针对最终输出文件所在段内的偏移量 offset 是由 resolveNameOff 和 resolveTypeOff 函数计算出来的，然后链接器把结果保存在 str 和 ptrToThis 中。具体逻辑可以见源码中下面 2 个函数:

```go
func resolveNameOff(ptrInModule unsafe.Pointer, off nameOff) name {}  
func resolveTypeOff(ptrInModule unsafe.Pointer, off typeOff) *_type {}
```

回到 \_type 类型。上文谈到 \_type 是所有类型原始信息的元信息。例如：

```go
type arraytype struct {
	typ   _type
	elem  *_type
	slice *_type
	len   uintptr
}

type chantype struct {
	typ  _type
	elem *_type
	dir  uintptr
}
```

在 arraytype 和 chantype 中保存类型的元信息就是靠 \_type。同样 interface 也有类似的类型定义：

```go
type imethod struct {
	name nameOff
	ityp typeOff
}

type interfacetype struct {
	typ     _type     // 类型元信息
	pkgpath name      // 包路径和描述信息等等
	mhdr    []imethod // 方法
}
```

因为 Go 语言中函数方法是以包为单位隔离的。所以 interfacetype 除了保存 \_type 还需要保存包路径等描述信息。mhdr 存的是各个 interface 函数方法在段内的偏移值 offset，知道偏移值以后才方便调用。 


### 2. 空 interface 数据结构

空的 inferface{} 是没有方法集的接口。所以不需要 itab 数据结构。它只需要存类型和类型对应的值即可。对应的数据结构如下：

```go
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```

从这个数据结构可以看出，只有当 2 个字段都为 nil，空接口才为 nil。空接口的主要目的有 2 个，一是实现“泛型”，二是使用反射。



## 二. 类型转换

举个具体的例子来说明 interface 是如何进行类型转换的。先来看指针类型转换。


### 1. 指针类型

```go
package main

import "fmt"

func main() {
	var s Person = &Student{name: "halfrost"}
	s.sayHello("everyone")
}

type Person interface {
	sayHello(name string) string
	sayGoodbye(name string) string
}

type Student struct {
	name string
}

//go:noinline
func (s *Student) sayHello(name string) string {
	return fmt.Sprintf("%v: Hello %v, nice to meet you.\n", s.name, name)
}

//go:noinline
func (s *Student) sayGoodbye(name string) string {
	return fmt.Sprintf("%v: Hi %v, see you next time.\n", s.name, name)
}

```

利用 go build 和 go tool 命令将上述代码变成汇编代码：

```go
$ go tool compile -S -N -l main.go >main.s1 2>&1
```

main 方法中有 3 个操作，重点关注后 2 个涉及到 interface 的操作：

1. 初始化 Student 对象指针  
2. 将 Student 对象指针转换成 interface  
3. 调用 interface 的方法  

> Plan9 汇编常见寄存器含义：  
> BP: 栈基，栈帧（函数的栈叫栈帧）的开始位置。  
> SP: 栈顶，栈帧的结束位置。  
> PC: 就是IP寄存器，存放CPU下一个执行指令的位置地址。  
> TLS: 虚拟寄存器。表示的是 thread-local storage，Golang 中存放了当前正在执行的g的结构体。  


先来看 Student 初始化的汇编代码：

```go
0x0021 00033 (main.go:6)	LEAQ	type."".Student(SB), AX      // 将 type."".Student 地址放入 AX 中
0x0028 00040 (main.go:6)	MOVQ	AX, (SP)                     // 将 AX 中的值存储在 SP 中
0x002c 00044 (main.go:6)	PCDATA	$1, $0
0x002c 00044 (main.go:6)	CALL	runtime.newobject(SB)        // 调用 runtime.newobject() 方法，生成 Student 对象存入 SB 中
0x0031 00049 (main.go:6)	MOVQ	8(SP), DI                    // 将生成的 Student 对象放入 DI 中
0x0036 00054 (main.go:6)	MOVQ	DI, ""..autotmp_2+40(SP)     // 编译器认为 Student 是临时变量，所以将 DI 放在栈上
0x003b 00059 (main.go:6)	MOVQ	$8, 8(DI)                    // (DI.Name).Len = 8
0x0043 00067 (main.go:6)	PCDATA	$0, $-2
0x0043 00067 (main.go:6)	CMPL	runtime.writeBarrier(SB), $0
0x004a 00074 (main.go:6)	JEQ	78
0x004c 00076 (main.go:6)	JMP	172
0x004e 00078 (main.go:6)	LEAQ	go.string."halfrost"(SB), AX  // 将 "halfrost" 字符串的地址放入 AX 中
0x0055 00085 (main.go:6)	MOVQ	AX, (DI)                      // (DI.Name).Data = &"halfrost"
0x0058 00088 (main.go:6)	JMP	90
0x005a 00090 (main.go:6)	PCDATA	$0, $-1
```

先将 \*\_type 放在 (SP) 栈顶。然后调用 runtime.newobject() 生成 Student 对象。(SP) 栈顶的值即是 newobject() 方法的入参。

```go
func newobject(typ *_type) unsafe.Pointer {
	return mallocgc(typ.size, typ, true)
}
```

PCDATA 用于生成 PC 表格，PCDATA 的指令用法为：PCDATA tableid, tableoffset。PCDATA有个两个参数，第一个参数为表格的类型，第二个是表格的地址。runtime.writeBarrier() 是 GC 相关的方法，感兴趣的可以研究它的源码。以下是 Student 对象临时对象 GC 的一些汇编代码逻辑，由于有 JMP 命令，代码是分开的，笔者在这里将它们汇集在一起。

```go
0x0043 00067 (main.go:6)    PCDATA  $0, $-2
0x0043 00067 (main.go:6)    CMPL    runtime.writeBarrier(SB), $0
0x004a 00074 (main.go:6)    JEQ 78
0x004c 00076 (main.go:6)    JMP 172
......
0x00ac 00172 (main.go:6)	PCDATA	$0, $-2
0x00ac 00172 (main.go:6)	LEAQ	go.string."halfrost"(SB), AX
0x00b3 00179 (main.go:6)	CALL	runtime.gcWriteBarrier(SB)
0x00b8 00184 (main.go:6)	JMP	90
0x00ba 00186 (main.go:6)	NOP
```

78 对应的十六进制是 0x004e，172 对应的十六进制是 0x00ac。先对比 runtime.writeBarrier(SB) 和 \$0 存的是否一致，如果相同则 JMP 到 0x004e 行，如果不同则 JMP 到 0x00ac 行。0x004e 行和 0x00ac 行代码完全相同，都是将字符串 "halfrost" 的地址放入 AX 中，不过 0x00ac 行执行完会紧接着调用 runtime.gcWriteBarrier(SB)。执行完成以后再回到 0x005a 行。

第一步结束以后，内存中存了 3 个值。临时变量 .autotmp\_2 放在 +40(SP) 的地址处，它也就是临时 Student 对象。

![](https://img.halfrost.com/Blog/ArticleImage/147_2_1.png)


接下来是第二步，将 Student 对象转换成 interface。

```go
0x005a 00090 (main.go:6)	MOVQ	""..autotmp_2+40(SP), AX
0x005f 00095 (main.go:6)	MOVQ	AX, ""..autotmp_1+48(SP)
0x0064 00100 (main.go:6)	LEAQ	go.itab.*"".Student,"".Person(SB), CX
0x006b 00107 (main.go:6)	MOVQ	CX, "".s+56(SP)
0x0070 00112 (main.go:6)	MOVQ	AX, "".s+64(SP)
```

经过上面几行汇编代码，成功的构造出了 itab 结构体。在汇编代码中可以找到 itab 的内存布局：

```go
go.itab.*"".Student,"".Person SRODATA dupok size=40
	0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 0c 31 79 12 00 00 00 00 00 00 00 00 00 00 00 00  .1y.............
	0x0020 00 00 00 00 00 00 00 00                          ........
	rel 0+8 t=1 type."".Person+0
	rel 8+8 t=1 type.*"".Student+0
	rel 24+8 t=1 "".(*Student).sayGoodbye+0
	rel 32+8 t=1 "".(*Student).sayHello+0
```

itab 结构体的首字节里面存的是 inter \*interfacetype，此处即 Person interface。第二个字节中存的是 \*\_type，这里是第一步生成的，放在 (SP) 地址处的地址。第四个字节中存的是 fun [1]uintptr，对应 sayGoodbye 方法的首地址。第五个字节中存的也是 fun [1]uintptr，对应 sayHello 方法的首地址。回顾上一章节里面的 itab 数据结构：

```go
type itab struct {
    inter *interfacetype // 8 字节
    _type *_type         // 8 字节
    hash  uint32 		 // 4 字节，填充使得内存对齐
    _     [4]byte        // 4 字节
    fun   [1]uintptr     // 8 字节
}
```
现在就很明确了为什么 fun 只需要存一个函数指针。每个函数指针都是 8 个字节，如果 interface 里面包含多个函数，只需要 fun 往后顺序偏移多个字节即可。第二步结束以后，内存中存储了以下这些值：


![](https://img.halfrost.com/Blog/ArticleImage/147_3_2.png)

在第二步中新建了一个临时变量 .autotmp\_1 放在 +48(SP) 地址处。并且利用第一步中生成的 Student 临时变量构造出了 \*itab。值得说明的是，虽然汇编代码并没有显示调用函数生成 iface，但是此时已经生成了 iface。

```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}
```

如上图，+56(SP) 处存的是 \*itab，+64(SP) 处存的是 unsafe.Pointer，这里的指针和 +8(SP) 的指针是完全一致的。接下来就是最后一步，调用 interface 的方法。

```go
0x0075 00117 (main.go:7)	MOVQ	"".s+56(SP), AX
0x007a 00122 (main.go:7)	TESTB	AL, (AX)
0x007c 00124 (main.go:7)	MOVQ	32(AX), AX
0x0080 00128 (main.go:7)	MOVQ	"".s+64(SP), CX
0x0085 00133 (main.go:7)	MOVQ	CX, (SP)
0x0089 00137 (main.go:7)	LEAQ	go.string."everyone"(SB), CX
0x0090 00144 (main.go:7)	MOVQ	CX, 8(SP)
0x0095 00149 (main.go:7)	MOVQ	$8, 16(SP)
0x009e 00158 (main.go:7)	NOP
0x00a0 00160 (main.go:7)	CALL	AX
```

先取出调用方法的真正对象，放入 (SP) 中，再依次将方法中的入参按照顺序放在 +8(SP) 之后。然后调用函数指针对应的方法。从汇编代码中可以看到，AX 直接从取出了 \*itab 指针存的内存地址，然后偏移到了 +32 的位置，这里是要调用的方法 sayHello 的内存地址。最后从栈顶依次取出需要的参数，即算完成 iterface 方法调用。方法调用前一刻，内存中的状态如下，主要关注 AX 的地址以及栈顶的所有参数信息。

![](https://img.halfrost.com/Blog/ArticleImage/147_4_1.png)

栈顶依次存放的是方法的调用者，参数。调用格式可以表示为 func(reciver, param1)。


### 2. 结构体类型

指针类型和结构体类型在类型转换中会有哪些区别？这一节好好分析对比一下。测试代码和指针类型大体一致，只是类型转换的时候换成了结构体，方法实现也换成了结构体，其他都没有变。

```go
package main

import "fmt"

func main() {
	var s Person = Student{name: "halfrost"}
	s.sayHello("everyone")
}

type Person interface {
	sayHello(name string) string
	sayGoodbye(name string) string
}

type Student struct {
	name string
}

//go:noinline
func (s Student) sayHello(name string) string {
	return fmt.Sprintf("%v: Hello %v, nice to meet you.\n", s.name, name)
}

//go:noinline
func (s Student) sayGoodbye(name string) string {
	return fmt.Sprintf("%v: Hi %v, see you next time.\n", s.name, name)
}
```

用同样的命令生成对应的汇编代码：

```go
$ go tool compile -S -N -l main.go >main.s2 2>&1
```

对比相同的 3 个环节：

1. 初始化 Student 对象  
2. 将 Student 对象转换成 interface  
3. 调用 interface 的方法 


```go
0x0021 00033 (main.go:6)	XORPS	X0, X0                       // X0 置 0
0x0024 00036 (main.go:6)	MOVUPS	X0, ""..autotmp_1+64(SP)     // 清空 +64(SP)
0x0029 00041 (main.go:6)	LEAQ	go.string."halfrost"(SB), AX
0x0030 00048 (main.go:6)	MOVQ	AX, ""..autotmp_1+64(SP)
0x0035 00053 (main.go:6)	MOVQ	$8, ""..autotmp_1+72(SP)
0x003e 00062 (main.go:6)	MOVQ	AX, (SP)
0x0042 00066 (main.go:6)	MOVQ	$8, 8(SP)
0x004b 00075 (main.go:6)	PCDATA	$1, $0
```

这段代码将 "halfrost" 放入内存相应的位置。上述代码 1-8 行，将字符串 "halfrost" 地址和长度 8 拷贝到 +0(SP)，+8(SP) 和 +64(SP)，+72(SP) 中。从这里可以了解到普通的临时变量在内存中布局是怎么样的。从上述汇编代码中可以看出，编译器发现这个变量只是临时变量，都没有调用 runtime.newobject()，仅仅是将它的每个基本类型的字段生成好放在内存中。


![](https://img.halfrost.com/Blog/ArticleImage/147_6_.png)


```go
0x004b 00075 (main.go:6)	CALL	runtime.convTstring(SB)
0x0050 00080 (main.go:6)	MOVQ	16(SP), AX
0x0055 00085 (main.go:6)	MOVQ	AX, ""..autotmp_2+40(SP)
0x005a 00090 (main.go:6)	LEAQ	go.itab."".Student,"".Person(SB), CX
0x0061 00097 (main.go:6)	MOVQ	CX, "".s+48(SP)
0x0066 00102 (main.go:6)	MOVQ	AX, "".s+56(SP)
```

上述代码生成了 interface。第 1 行调用了 runtime.convTstring()。

```go
func convTstring(val string) (x unsafe.Pointer) {
	if val == "" {
		x = unsafe.Pointer(&zeroVal[0])
	} else {
		x = mallocgc(unsafe.Sizeof(val), stringType, true)
		*(*string)(x) = val
	}
	return
}
```

runtime.convTstring() 会从栈顶 +0(SP) 取出入参 "halfrost" 和长度 8。在栈上生成了一个字符串的变量，返回了它的指针放在 +16(SP) 中，并拷贝到 +40(SP) 里。第 4 行生成了 itab 的指针，这里和上一章里面一致，不再赘述。至此，iface 生成了，\*itab 和 unsafe.Pointer 分别存在 +48(SP) 和 +56(SP) 中。



![](https://img.halfrost.com/Blog/ArticleImage/147_5_1.png)



```go
0x006b 00107 (main.go:7)	MOVQ	"".s+48(SP), AX
0x0070 00112 (main.go:7)	TESTB	AL, (AX)
0x0072 00114 (main.go:7)	MOVQ	32(AX), AX
0x0076 00118 (main.go:7)	MOVQ	"".s+56(SP), CX
0x007b 00123 (main.go:7)	MOVQ	CX, (SP)
0x007f 00127 (main.go:7)	LEAQ	go.string."everyone"(SB), CX
0x0086 00134 (main.go:7)	MOVQ	CX, 8(SP)
0x008b 00139 (main.go:7)	MOVQ	$8, 16(SP)
0x0094 00148 (main.go:7)	CALL	AX
```

最后一步是调用 interface 方法。这一步和上一节中的流程基本一致。先通过 itab  指针找到函数指针。然后将要调用的方法的入参都放在栈顶。最后调用即可。此时内存布局如下图：

![](https://img.halfrost.com/Blog/ArticleImage/147_7_0.png)


看到这里可能有读者好奇，为什么结构体类型转换里面也没有 runtime.convT2I() 方法调用呢？笔者认为这里是编译器的一些优化导致的。

```go
func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
	t := tab._type
	if raceenabled {
		raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2I))
	}
	if msanenabled {
		msanread(elem, t.size)
	}
	x := mallocgc(t.size, t, true)
	typedmemmove(t, x, elem)
	i.tab = tab
	i.data = x
	return
}
```

runtime.convT2I() 这个方法会生成一个 iface，在堆上生成 iface.data，并且会 typedmemmove()。笔者找了 2 个相关的 PR，感兴趣的可以看看。[optimize convT2I as a two-word copy when T is pointer-shaped](https://go-review.googlesource.com/c/go/+/20901/9)，[cmd/compile: optimize remaining convT2I calls](https://go-review.googlesource.com/c/go/+/20902)。这里仅仅涉及类型转换，所以在内存中构造出 \*itab 和 unsafe.Pointer 就够用了。编译器觉得没有必要调用 runtime.convT2I() 再构造出 iface 多此一举。

### 3. 隐式类型转换

日常开发中要注意隐式类型转换，一不小心会带来 bug。例如，自定义的 error 类型会因为隐藏的类型转换变为非 nil。代码如下：

```go
package main

import "fmt"

type GrpcError struct{}

func (e GrpcError) Error() string {
	return "GrpcError"
}

func main() {
	err := cal()
	fmt.Println(err)            // 打印：<nil>
	fmt.Println(err == nil)     // 打印：false
}

func cal() error {
	var err *GrpcError = nil
	return err
}
```

项目中可能会把 gRPC 框架抛出来的错误再封装一层，将返回的错误信息可读性变得更强。殊不知一不小心会带来 bug。上述代码在 main 中判断 err 是否为 nil，答案是 false。error 是一个非空 interface，底层数据结构是 iface，尽管 data 是 nil，但是 \*itab 并不为空，所以 err == nil 答案为 false。

看到这里可能就有读者想问，这种隐式转换有什么用。这个转换恰恰是一个精妙的设计。由本节前 2 节的内容，我们知道将一个对象传递给 interface{} 类型，编译器自动会将它转换成相关类型的数据结构。如果不这样设计，Go 语言设计者还需要再为它单独设计一套类型数据结构来支持反射特性。Go 语言设计者看到了 interface 的特点，基于它的动态类型转换实现了反射特性，事半功倍。


## 三. 类型断言 Type Assertion

作为 interface 另一个重要应用就是类型断言。针对非空接口和空接口，分别来看看底层汇编代码是如何处理的。

### 1. 非空接口

测试代码如下：

```go
func main() {
	var s Person = &Student{name: "halfrost"}
	v, ok := s.(Person)
	if !ok {
		fmt.Printf("%v\n", v)
	}
}
```

利用相同的命令将上述代码转换成汇编代码。

```go
go tool compile -S -N -l main.go >main.s3 2>&1
```

main 函数第一行生成 Student 对象的指针，并将它赋值给 Person 接口，这段代码在上一章中出现多次，对应的汇编代码也没有发生变化：

```go
0x002f 00047 (main.go:8)	LEAQ	type."".Student(SB), AX
0x0036 00054 (main.go:8)	MOVQ	AX, (SP)
0x003a 00058 (main.go:8)	PCDATA	$1, $0
0x003a 00058 (main.go:8)	CALL	runtime.newobject(SB)
0x003f 00063 (main.go:8)	MOVQ	8(SP), DI
0x0044 00068 (main.go:8)	MOVQ	DI, ""..autotmp_7+80(SP)
0x0049 00073 (main.go:8)	MOVQ	$8, 8(DI)
0x0051 00081 (main.go:8)	PCDATA	$0, $-2
0x0051 00081 (main.go:8)	CMPL	runtime.writeBarrier(SB), $0
0x0058 00088 (main.go:8)	JEQ	95
0x005a 00090 (main.go:8)	JMP	529
0x005f 00095 (main.go:8)	LEAQ	go.string."halfrost"(SB), AX
0x0066 00102 (main.go:8)	MOVQ	AX, (DI)
0x0069 00105 (main.go:8)	JMP	107
0x006b 00107 (main.go:8)	PCDATA	$0, $-1
0x006b 00107 (main.go:8)	MOVQ	""..autotmp_7+80(SP), AX
0x0070 00112 (main.go:8)	MOVQ	AX, ""..autotmp_3+88(SP)
0x0075 00117 (main.go:8)	LEAQ	go.itab.*"".Student,"".Person(SB), CX
0x007c 00124 (main.go:8)	MOVQ	CX, "".s+120(SP)
0x0081 00129 (main.go:8)	MOVQ	AX, "".s+128(SP)
```

这里不再对上述代码进行分析，详细的见上一章。iface 结构体也生成了，在 +120(SP) ~ +128(SP) 处。到此内存布局情况如下图：

![](https://img.halfrost.com/Blog/ArticleImage/147_8_0.png)

接下来的代码是类型推断的关键代码。由于汇编代码过长，笔者将它拆成 2 部分。第一部分是类型断言的关键部分。

```go
0x0089 00137 (main.go:9)	XORPS	X0, X0
0x008c 00140 (main.go:9)	MOVUPS	X0, ""..autotmp_4+152(SP)
0x0094 00148 (main.go:9)	MOVQ	"".s+120(SP), AX
0x0099 00153 (main.go:9)	MOVQ	"".s+128(SP), CX
0x00a1 00161 (main.go:9)	LEAQ	type."".Person(SB), DX
0x00a8 00168 (main.go:9)	MOVQ	DX, (SP)
0x00ac 00172 (main.go:9)	MOVQ	AX, 8(SP)
0x00b1 00177 (main.go:9)	MOVQ	CX, 16(SP)
0x00b6 00182 (main.go:9)	CALL	runtime.assertI2I2(SB)
```

在上述代码中，可以看到，为了调用 runtime.assertI2I2() 方法，连续在栈顶放入了 3 个参数。分别是 \*interfacetype，\*itab 和 unsafe.Pointer。对应 runtime.assertI2I2() 源码：

```go
func assertI2I2(inter *interfacetype, i iface) (r iface, b bool) {
	tab := i.tab
	if tab == nil {
		return
	}
	if tab.inter != inter {
		tab = getitab(inter, tab._type, true)
		if tab == nil {
			return
		}
	}
	r.tab = tab
	r.data = i.data
	b = true
	return
}
```

上述代码中入参虽然是 2 个，但是 iface 可以拆成 2 个，即 \*itab 和 unsafe.Pointer。所以栈顶连续的 +0(SP)，+8(SP)，+16(SP) 满足了函数入参的需求。上述代码逻辑很简单，如果 iface 中的 itab.inter 和第一个入参 \*interfacetype 相同，说明类型相同，直接返回入参 iface 的相同类型，布尔值为 true；如果 iface 中的 itab.inter 和第一个入参 \*interfacetype 不相同，则重新根据 \*interfacetype 和 iface.tab 去构造 tab。构造的过程会查找  itabTable。如果类型不匹配，或者不是属于同一个 interface 类型，都会失败。getitab() 方法第三个参数是 canfail，这里传入了 true，表示构建 \*itab 允许失败，失败以后返回 nil。回到 runtime.assertI2I2() 方法中，tab 构建失败以后为 nil，直接 return，导致外部接收到的 iface 是 nil，bool 也为 false。

第二部分无非是赋值部分，没有难度。

```go
0x00bb 00187 (main.go:9)	MOVQ	24(SP), AX
0x00c0 00192 (main.go:9)	MOVQ	32(SP), CX
0x00c5 00197 (main.go:9)	MOVBLZX	40(SP), DX
0x00ca 00202 (main.go:9)	MOVQ	AX, ""..autotmp_4+152(SP)
0x00d2 00210 (main.go:9)	MOVQ	CX, ""..autotmp_4+160(SP)
0x00da 00218 (main.go:9)	MOVB	DL, ""..autotmp_5+71(SP)
0x00de 00222 (main.go:9)	MOVQ	""..autotmp_4+152(SP), AX
0x00e6 00230 (main.go:9)	MOVQ	""..autotmp_4+160(SP), CX
0x00ee 00238 (main.go:9)	MOVQ	AX, "".v+104(SP)
0x00f3 00243 (main.go:9)	MOVQ	CX, "".v+112(SP)
0x00f8 00248 (main.go:9)	MOVBLZX	""..autotmp_5+71(SP), AX
0x00fd 00253 (main.go:9)	MOVB	AL, "".ok+70(SP)
```

runtime.assertI2I2() 方法的返回值放在 +24(SP)、+32(SP)、+40(SP) 中。返回值是 3 个值，因为把 iface 拆成了 2 个值。注意这里 +40(SP) 用的是 MOVBLZX 命令，因为 bool 是 uint8，之后在移动过程中，也只用到了低 8 位，所以不是用的 DX 而是 DL。经过临时变量的转移，最终返回值放在了变量 v 和 ok 中。v 在内存里 +104(SP) ~ +112(SP)，ok 在内存里 +70(SP)。


这里再提一点的是，如果类型推断是一个具体的类型，编译器会直接构造出 iface，而不会调用 runtime.assertI2I2() 构造 iface。例如下面的代码，类型推断处写的是具体的一个类型：

```go
func main() {
	var s Person = &Student{name: "halfrost"}
	v, ok := s.(*Student)
	if !ok {
		fmt.Printf("%v\n", v)
	}
}
```

编译器在处理转换成汇编代码的时候，会做优化，不会再调用 runtime.assertI2I2() 查找 itabTable。具体处理逻辑见下面汇编代码。

```go
0x0075 00117 (main.go:8)	LEAQ	go.itab.*"".Student,"".Person(SB), CX
0x007c 00124 (main.go:8)	MOVQ	CX, "".s+104(SP)
0x0081 00129 (main.go:8)	MOVQ	AX, "".s+112(SP)
0x0086 00134 (main.go:9)	MOVQ	$0, ""..autotmp_3+96(SP)
0x008f 00143 (main.go:9)	MOVQ	"".s+112(SP), AX
0x0094 00148 (main.go:9)	LEAQ	go.itab.*"".Student,"".Person(SB), CX
0x009b 00155 (main.go:9)	NOP
0x00a0 00160 (main.go:9)	CMPQ	"".s+104(SP), CX
```

上述代码中，先构造出 iface，其中 \*itab 存在内存 +104(SP) 中，unsafe.Pointer 存在 +112(SP) 中。然后在类型推断的时候又重新构造了一遍 \*itab，最后将新的 \*itab 和前一次 +104(SP) 里的 \*itab 进行对比。

小结：**非空接口类型推断的实质是 iface 中 \*itab 的对比**。\*itab 匹配成功会在内存中组装返回值。匹配失败直接清空寄存器，返回默认值。


### 2. 空接口


在来看看空接口的类型推断底层是怎么样的。测试代码如下：

```go
func main() {
	var s interface{} = &Student{name: "halfrost"}
	v, ok := s.(int)
	if !ok {
		fmt.Printf("%v\n", v)
	}
}
```

利用相同的命令将上述代码转换成汇编代码。

```go
go tool compile -S -N -l main.go >main.s4 2>&1
```

main 函数第一行生成 Student 对象的指针，并将它赋值给空接口，这段代码在上一章中出现多次，对应的汇编代码也没有发生变化：

```go
0x002f 00047 (main.go:8)	XORPS	X0, X0
0x0032 00050 (main.go:8)	MOVUPS	X0, ""..autotmp_8+136(SP)
0x003a 00058 (main.go:8)	LEAQ	""..autotmp_8+136(SP), AX
0x0042 00066 (main.go:8)	MOVQ	AX, ""..autotmp_7+88(SP)
0x0047 00071 (main.go:8)	TESTB	AL, (AX)
0x0049 00073 (main.go:8)	MOVQ	$8, ""..autotmp_8+144(SP)
0x0055 00085 (main.go:8)	LEAQ	go.string."halfrost"(SB), CX
0x005c 00092 (main.go:8)	MOVQ	CX, ""..autotmp_8+136(SP)
0x0064 00100 (main.go:8)	MOVQ	AX, ""..autotmp_3+96(SP)
0x0069 00105 (main.go:8)	LEAQ	type.*"".Student(SB), CX
0x0070 00112 (main.go:8)	MOVQ	CX, "".s+120(SP)
0x0075 00117 (main.go:8)	MOVQ	AX, "".s+128(SP)
```

赋值给空接口，并不会新建临时变量，数据都存在栈上。上述代码执行完，就是组装了一个 eface 在内存中，内存布局如下：


![](https://img.halfrost.com/Blog/ArticleImage/147_9_0.png)

在第二章中，我们知道 eface 是空接口的数据结构，它包含 2 个字段：


```go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```

从内存中可以看到 eface 的 \*\_type 存在内存的 +120(SP) 处，unsafe.Pointer 存在了 +128(SP) 处。注意上图中，有多处的值是一样的，+88(SP)，+96(SP)，+128(SP)，这 3 个地址下的值和 AX 寄存器中存的值是一样的，存的都是 +136(SP) 的地址值。再来看看空接口的类型推断汇编实现：


```go
0x007d 00125 (main.go:9)	MOVQ	"".s+120(SP), AX
0x0082 00130 (main.go:9)	MOVQ	"".s+128(SP), CX
0x008a 00138 (main.go:9)	LEAQ	type.int(SB), DX
0x0091 00145 (main.go:9)	CMPQ	DX, AX
0x0094 00148 (main.go:9)	JEQ	155
0x0096 00150 (main.go:9)	JMP	423
```

从上面这段代码里面可以看出来，空接口的类型断言很简单，就是 eface 的第一个字段 \*\_type 和要比较类型的 \*\_type 进行对比，如果相同就准备接下来的返回值。

```go
0x009b 00155 (main.go:9)	MOVQ	(CX), AX
0x009e 00158 (main.go:9)	MOVL	$1, CX
0x00a3 00163 (main.go:9)	JMP	165
0x00a5 00165 (main.go:9)	MOVQ	AX, ""..autotmp_4+80(SP)
0x00aa 00170 (main.go:9)	MOVB	CL, ""..autotmp_5+71(SP)
0x00ae 00174 (main.go:9)	MOVQ	""..autotmp_4+80(SP), AX
0x00b3 00179 (main.go:9)	MOVQ	AX, "".v+72(SP)
0x00b8 00184 (main.go:9)	MOVBLZX	""..autotmp_5+71(SP), AX
0x00bd 00189 (main.go:9)	MOVB	AL, "".ok+70(SP)
```

如果类型断言推断正确，就准备返回值，经过中间一些临时变量的传递，最终 v 保存在内存中 +72(SP) 处。ok 保存在内存 +70(SP) 处。最终内存中的状态如下所示：


![](https://img.halfrost.com/Blog/ArticleImage/147_10_.png)



```go
0x01a7 00423 (main.go:11)	XORL	AX, AX
0x01a9 00425 (main.go:11)	XORL	CX, CX
0x01ab 00427 (main.go:9)	JMP	165
0x01b0 00432 (main.go:9)	NOP
```

如果断言失败，清空 AX 和 CX 寄存器。AX 和 CX 中存的是 eface 结构体里面的 2 个字段。


小结：**空接口类型推断的实质是 eface 中 \*_type 的对比**。\*\_type 匹配成功会在内存中组装返回值。匹配失败直接清空寄存器，返回默认值。




## 四. 类型查询 Type Switches


类型查询也是接口运算的一种。这一节详细分析一下类型查询的底层原理。首先需要解释的一点是，类型查询的对象必须是接口类型，因为一个具体的类型是固定的，声明以后就不会变化，所以具体类型的变量都不存在类型查询的运算。

先做一个约定，main 函数的第一行，不管是生成 Student 还是生成 \*Student 对本节都没有影响。

```go
	var s Person = &Student{name: "halfrost"}
	var s Person = Student{name: "halfrost"}
```

上面这 2 行都会生成 Person 这样的 interface 类型，生成过程中的区别在第二章类型转换中详细讲解了，这一章不再赘述。本章重点讲解的是下面 switch-case 的内容。


### 1. 非空接口

本节中将会用如下的代码进行深入研究。

```go
func main() {
	var s Person = &Student{name: "halfrost"}
	switch s.(type) {
	case Person:
		person := s.(Person)
		person.sayHello("everyone")
	case *Student:
		student := s.(*Student)
		student.sayHello("everyone")
	case Student:
		student := s.(Student)
		student.sayHello("everyone")
	}
}
```

针对 Type Switches 还有一点需要说明的。case 后面可以跟非接口的类型名，也可以跟接口类型名。如上述代码，case 后面可以跟 Person 这个接口名，也可以跟 Student 这样非接口的类型名。接口变量和哪个类型先匹配上了，就是哪个类型了。例如先和 Person 匹配上了，那么 s 就是 Person 类型，不会继续往下匹配了。**fallthrough 语句不能在 Type Switches 中使用**。强行使用，编译器会报错，`fallthrough statement out of placecompiler`。这也符合常理，不会有一种类型能符合所有类型。将上述代码转换成汇编代码：

```go
$ go tool compile -S -N -l main.go >main.s5 2>&1
```

生成的汇编代码生成 Person 类型的代码这里不再赘述，直接从 switch 开始分析。这里有 3 个 case，就分 3 部分分别分析。首先是第一部分，匹配 Person 类型。下图是此时内存中的情况：

![](https://img.halfrost.com/Blog/ArticleImage/147_11_1.png)

```go
0x0086 00134 (main.go:9)	MOVQ	"".s+96(SP), AX
0x008b 00139 (main.go:9)	MOVQ	"".s+104(SP), CX
0x0090 00144 (main.go:9)	MOVQ	AX, ""..autotmp_8+128(SP)
0x0098 00152 (main.go:9)	MOVQ	CX, ""..autotmp_8+136(SP)
0x00a0 00160 (main.go:9)	TESTQ	AX, AX
0x00a3 00163 (main.go:9)	JNE	170
0x00a5 00165 (main.go:9)	JMP	750
0x00aa 00170 (main.go:9)	MOVL	16(AX), AX
0x00ad 00173 (main.go:9)	MOVL	AX, ""..autotmp_10+52(SP)
```

生成 Person 的 iface 在内存的 +128(SP) ~ +136(SP) 中。(16)AX 取出的是 \*itab 中的 hash 值。然后存入 +52(SP) 中。接下来是匹配 case Person 的代码了。


```go
0x00b1 00177 (main.go:10)	MOVQ	""..autotmp_8+128(SP), AX
0x00b9 00185 (main.go:10)	MOVQ	""..autotmp_8+136(SP), CX
0x00c1 00193 (main.go:10)	LEAQ	type."".Person(SB), DX
0x00c8 00200 (main.go:10)	MOVQ	DX, (SP)
0x00cc 00204 (main.go:10)	MOVQ	AX, 8(SP)
0x00d1 00209 (main.go:10)	MOVQ	CX, 16(SP)
0x00d6 00214 (main.go:10)	PCDATA	$1, $1
0x00d6 00214 (main.go:10)	CALL	runtime.assertI2I2(SB)
0x00db 00219 (main.go:10)	MOVBLZX	40(SP), AX
0x00e0 00224 (main.go:10)	MOVB	AL, ""..autotmp_9+51(SP)
0x00e4 00228 (main.go:10)	TESTB	AL, AL
0x00e6 00230 (main.go:10)	JNE	237
0x00e8 00232 (main.go:10)	JMP	383
0x00ed 00237 (main.go:10)	PCDATA	$1, $-1
0x00ed 00237 (main.go:10)	JMP	239
```

上述代码主要是调用 runtime.assertI2I2()，这个方法源码第三章类型推断里面分析过了，这里不再赘述。这个方法需要 2 个入参，分别是 \*interfacetype, iface，DX 中放的是 type(Person) 的地址，即 \*interfacetype，AX 和 CX 分别存的是 \*Student 的 iface.\*itab 和 iface.unsafe.Pointer，如果匹配上了，返回 bool 放在 AX 中。如果为 true，则表示 TESTB 不等，那么执行 JNE 237。如果为 false，代表与 Person 匹配失败，则表示 TESTB 相等，那么执行 JMP 383。先看匹配成功的情况，即 TESTB 不等：

```go
0x00ed 00237 (main.go:10)	PCDATA	$1, $-1
0x00ed 00237 (main.go:10)	JMP	239
0x00ef 00239 (main.go:11)	XORPS	X0, X0
0x00f2 00242 (main.go:11)	MOVUPS	X0, ""..autotmp_5+160(SP)
0x00fa 00250 (main.go:11)	MOVQ	"".s+96(SP), AX
0x00ff 00255 (main.go:11)	MOVQ	"".s+104(SP), CX
0x0104 00260 (main.go:11)	LEAQ	type."".Person(SB), DX
0x010b 00267 (main.go:11)	MOVQ	DX, (SP)
0x010f 00271 (main.go:11)	MOVQ	AX, 8(SP)
0x0114 00276 (main.go:11)	MOVQ	CX, 16(SP)
0x0119 00281 (main.go:11)	PCDATA	$1, $0
0x0119 00281 (main.go:11)	CALL	runtime.assertI2I(SB)
0x011e 00286 (main.go:11)	MOVQ	24(SP), AX
0x0123 00291 (main.go:11)	MOVQ	32(SP), CX
0x0128 00296 (main.go:11)	MOVQ	AX, ""..autotmp_5+160(SP)
0x0130 00304 (main.go:11)	MOVQ	CX, ""..autotmp_5+168(SP)
0x0138 00312 (main.go:11)	MOVQ	AX, "".person+112(SP)
0x013d 00317 (main.go:11)	MOVQ	CX, "".person+120(SP)
```

在调用 Type Switches 之前，从内存图中可以看到 +96(SP) 存的是 \*itab，+104(SP) 存的是 unsafe.Pointer。在调用 runtime.assertI2I() 方法之前先把 3 个入参都放在栈顶。(SP)、+8(SP)、+16(SP) 分别放的是 \*interfacetype，\*itab 和 unsafe.Pointer。runtime.assertI2I() 源码如下：

```go
func assertI2I(inter *interfacetype, i iface) (r iface) {
	tab := i.tab
	if tab == nil {
		// explicit conversions require non-nil interface value.
		panic(&TypeAssertionError{nil, nil, &inter.typ, ""})
	}
	if tab.inter == inter {
		r.tab = tab
		r.data = i.data
		return
	}
	r.tab = getitab(inter, tab._type, false)
	r.data = i.data
	return
}
```

assertI2I() 方法比 assertI2I2() 方法返回值少了一个 bool 变量。所以函数名也少了一个 2。assertI2I() 方法比 assertI2I2() 方法更加危险，因为可能出现 panic。如果匹配上了，返回一个 iface，这个 iface 和入参的 iface 里面的值是一样的，也就是复制了一个。返回的 iface.\*itab 放在 +112(SP) 中。接下来是调用方法的代码。

```go
0x0142 00322 (main.go:12)	MOVQ	"".person+112(SP), AX
0x0147 00327 (main.go:12)	TESTB	AL, (AX)
0x0149 00329 (main.go:12)	MOVQ	32(AX), AX
0x014d 00333 (main.go:12)	MOVQ	"".person+120(SP), CX
0x0152 00338 (main.go:12)	MOVQ	CX, (SP)
0x0156 00342 (main.go:12)	LEAQ	go.string."everyone"(SB), CX
0x015d 00349 (main.go:12)	MOVQ	CX, 8(SP)
0x0162 00354 (main.go:12)	MOVQ	$8, 16(SP)
0x016b 00363 (main.go:12)	CALL	AX
```

上述代码在第二章类型转换中出现了多次，这里不再赘述了。主要是找到函数指针，然后将函数需要的入参放在栈顶，最后调用方法即可。回到 runtime.assertI2I2() 方法调用之后，如果为 bool 为 false，代表与 Person 匹配失败，则表示 TESTB 相等，那么执行 JMP 383。


```go
0x017f 00383 (main.go:9)	CMPL	""..autotmp_10+52(SP), $309932300
0x0187 00391 (main.go:10)	JEQ	398
0x0189 00393 (main.go:10)	JMP	546
```

比较 +52(SP) 和 309932300，+52(SP) 是之前存的 hash 值。如果相等则跳转到 398。309932300 对应的十六进制是 0x1279310c，在内存中查找这个值，可以找到是 \*Student 类型中 \*itab 里面的的 hash 值。


```go
go.itab.*"".Student,"".Person SRODATA dupok size=40
	0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 0c 31 79 12 00 00 00 00 00 00 00 00 00 00 00 00  .1y.............
	0x0020 00 00 00 00 00 00 00 00                          ........
```

如上图，内存中前 16 字节分别是 ，从 24 字节低 4 个字节开始是 hash 值，高 4 个字节是填充位，为了内存对齐的，这里全部填充了 0。如果 hash 值相同，代表匹配上了，那么 JEQ 398 。如果没有匹配上则 JMP	546。先看匹配上的情况：

```go
0x018e 00398 (main.go:13)	LEAQ	go.itab.*"".Student,"".Person(SB), AX
0x0195 00405 (main.go:13)	CMPQ	""..autotmp_8+128(SP), AX
0x019d 00413 (main.go:13)	JEQ	418
```

hash 匹配上了只是第一步，还需要再匹配 \*itab 是否相同。hash 和 \*itab 两者都匹配完成，才算是走到了这个对应的 case 中。接下来是类型断言的过程：

```go
0x01b5 00437 (main.go:14)	MOVQ	"".s+104(SP), AX
0x01ba 00442 (main.go:14)	MOVQ	"".s+96(SP), CX
0x01bf 00447 (main.go:14)	LEAQ	go.itab.*"".Student,"".Person(SB), DX
0x01c6 00454 (main.go:14)	CMPQ	CX, DX
0x01c9 00457 (main.go:14)	JEQ	464
0x01cb 00459 (main.go:14)	JMP	806
0x01d0 00464 (main.go:14)	MOVQ	AX, "".student+56(SP)
```

上述代码对应的是 main 函数中的第 7 行。

```go
student := s.(*Student)
```

这里类型断言还做了一次 \*itab 的对比，如果一致，则接下来进行方法调用前的入参准备工作，把所有的入参都放入栈顶。

```go
0x01d5 00469 (main.go:15)	TESTB	AL, (AX)
0x01d7 00471 (main.go:15)	MOVQ	(AX), CX
0x01da 00474 (main.go:15)	MOVQ	8(AX), AX
0x01de 00478 (main.go:15)	MOVQ	CX, ""..autotmp_11+176(SP)
0x01e6 00486 (main.go:15)	MOVQ	AX, ""..autotmp_11+184(SP)
0x01ee 00494 (main.go:15)	MOVQ	CX, (SP)
0x01f2 00498 (main.go:15)	MOVQ	AX, 8(SP)
0x01f7 00503 (main.go:15)	LEAQ	go.string."everyone"(SB), AX
0x01fe 00510 (main.go:15)	MOVQ	AX, 16(SP)
0x0203 00515 (main.go:15)	MOVQ	$8, 24(SP)
0x020c 00524 (main.go:15)	PCDATA	$1, $0
0x020c 00524 (main.go:15)	CALL	"".Student.sayHello(SB)
```

用 \*Student 指针调用 sayHello() 方法一共需要 4 个参数，分别放在了内存 (SP)、+8(SP)、+16(SP)、+24(SP) 的位置。依次放的内容是 \*("halfrost")、8、\*(everyone)、8 。最后调用方法的时候从栈顶取走这 4 个入参即完成调用。

回到 case 判断，如果没有匹配上，则会 JMP 546：

```go
0x0222 00546 (main.go:9)	CMPL	""..autotmp_10+52(SP), $-736059430
0x022a 00554 (main.go:10)	JEQ	561
```

这段代码又是判断 hash 值。说明第二个 case 没有匹配上就开始匹配第三个 case。注意到这里打印出来的是有符号的十进制数，在计算 hash 的时候要换成十六进制。十进制 -736059430 换成二进制是 10101011110111110110000000100110，负数的反码是符号位不变，其他每一位取反，取反则为 11010100001000001001111111011001，负数的补码是反码 + 1，则补码是 11010100001000001001111111011010，转换成 16 进制即为 0xd4209fda。在内存中搜索 0xd4209fda，可以找到下面的内存布局：


```go
go.itab."".Student,"".Person SRODATA dupok size=40
	0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 da 9f 20 d4 00 00 00 00 00 00 00 00 00 00 00 00  .. .............
	0x0020 00 00 00 00 00 00 00 00                          ........
```

可以看到 0xd4209fda 正好是 Student 类型对应 \*itab 的 hash 值。 接下来的汇编代码和第二个 case 的代码逻辑完全一致，这里不再贴出完整的汇编代码。hash 值匹配上了以后，再进行 \*itab 的匹配，两者都匹配成功，便会进入 case 里面，进行类型断言，类型断言还会再进行一次 \*itab 的对比，如果对比相同，则会进行方法调用前的参数准备。将所有入参放入栈顶，最后调用方法。


如果上面每个 case 都匹配失败了，会 JMP 367，退出 Type Switches。

```go
0x016f 00367 (main.go:9)	PCDATA	$1, $-1
0x016f 00367 (main.go:9)	MOVQ	192(SP), BP
0x0177 00375 (main.go:9)	ADDQ	$200, SP
0x017e 00382 (main.go:9)	RET
```

退出的汇编代码没有什么处理逻辑，就是清理现场，返回。小结：

- Type Switches case 如果跟的是非空接口的类型名，则会调用 runtime.assertI2I2() 判断 case 是否匹配，如果匹配成功，进入 case 内部类型断言会再调用 runtime.assertI2I() 拿到 iface。
- Type Switches case 如果跟的是非接口的类型名，则先根据 hash 值匹配类型，hash 匹配成功再匹配 \*itab，两个都匹配成功才能进入 case 内部。进入以后的类型断言还会再判断一次 \*itab 是否一致。


### 2. 空接口

再来看看空接口。本节中将会用如下的代码进行深入研究。

```go
func main() {
	var s interface{} = &Student{name: "halfrost"}
	switch s.(type) {
	case Person:
		person := s.(Person)
		person.sayHello("everyone")
	case *Student:
		student := s.(*Student)
		student.sayHello("everyone")
	case Student:
		student := s.(Student)
		student.sayHello("everyone")
	}
}
```

用相同的命令将上述代码转换成汇编代码：

```go
$ go tool compile -S -N -l main.go >main.s6 2>&1
```

由于有大量逻辑和非空接口是一样的，所以这里重点分析不同的地方。main 函数第一行生成 Student 的指针并且类型转换成 interface{} 类型，这段代码在第二章中出现了，这里不再赘述。先来看 main 函数第二行：

```go
0x00b1 00177 (main.go:10)	MOVQ	""..autotmp_8+128(SP), AX
0x00b9 00185 (main.go:10)	MOVQ	""..autotmp_8+136(SP), CX
0x00c1 00193 (main.go:10)	LEAQ	type."".Person(SB), DX
0x00c8 00200 (main.go:10)	MOVQ	DX, (SP)
0x00cc 00204 (main.go:10)	MOVQ	AX, 8(SP)
0x00d1 00209 (main.go:10)	MOVQ	CX, 16(SP)
0x00d6 00214 (main.go:10)	PCDATA	$1, $1
0x00d6 00214 (main.go:10)	CALL	runtime.assertE2I2(SB)
```

可以看到上述汇编代码逻辑和非空接口的逻辑基本一致，只是调用的方法不同。非空接口调用的是 runtime.assertI2I2()，而非空接口这里调用的是 runtime.assertE2I2()。它的源码如下：

```go
func assertE2I2(inter *interfacetype, e eface) (r iface, b bool) {
	t := e._type
	if t == nil {
		return
	}
	tab := getitab(inter, t, true)
	if tab == nil {
		return
	}
	r.tab = tab
	r.data = e.data
	b = true
	return
}
```

这段代码逻辑和 assertI2I2() 大体一致，只不过这里是把 eface 转换成 iface。通过调用 getitab() 方法，把 eface 中的 \_type 组装成 \*itab，再拼上 eface 的 data，即构成了 iface 了。成功匹配进入 case 以后，进行类型推断：

```go
0x00fa 00250 (main.go:11)	MOVQ	"".s+96(SP), AX
0x00ff 00255 (main.go:11)	MOVQ	"".s+104(SP), CX
0x0104 00260 (main.go:11)	LEAQ	type."".Person(SB), DX
0x010b 00267 (main.go:11)	MOVQ	DX, (SP)
0x010f 00271 (main.go:11)	MOVQ	AX, 8(SP)
0x0114 00276 (main.go:11)	MOVQ	CX, 16(SP)
0x0119 00281 (main.go:11)	PCDATA	$1, $0
0x0119 00281 (main.go:11)	CALL	runtime.assertE2I(SB)
```

此处代码逻辑和非空接口也是一致的，只是调用的方法不同。这里调用的是 runtime.assertE2I() 方法：

```go
func assertE2I(inter *interfacetype, e eface) (r iface) {
	t := e._type
	if t == nil {
		// explicit conversions require non-nil interface value.
		panic(&TypeAssertionError{nil, nil, &inter.typ, ""})
	}
	r.tab = getitab(inter, t, false)
	r.data = e.data
	return
}
```

runtime.assertE2I() 同 runtime.assertI2I() 一样都是“危险”方法，可能会发生 panic。方法返回 iface。再往下就是调用 sayHello() 方法，逻辑和非空接口完全一致，剩下的 2 个 case 匹配过程也和非空接口完全一致，这里就不分析了。

值得一提的是，在匹配非接口类型的 hash 值时，hash 值只和字段和方法有关，和字段内具体的值无关。也就是说非空接口和空接口这 2 次匹配 \*Student 和 Student 类型 hash 值，都是一样的，0x1279310c 和 0xd4209fda。这是符合常理的，对象存储的字段值不同，并不改变对象的类型，只要类型完全一致，hash 值就相同。小结：

- Type Switches case 如果跟的是空接口的类型名，则会调用 runtime.assertE2I2() 判断 case 是否匹配，如果匹配成功，进入 case 内部类型断言会再调用 runtime.assertE2I() 拿到 iface。

## 五. 动态派发 

虽然 Go 并不是严格意义的面向对象语言，但是 Go 中 interface 可以动态派发方法，实现类似面向对象语言中的多态的特性。

多态是一种运行期的行为，它有以下几个特点：  
- 一种类型具有多种类型的能力  
- 允许不同的对象对同一消息做出灵活的反应  
- 以一种通用的方式对待个使用的对象  
- 非动态语言必须通过继承和接口的方式来实现  

本节中的测试代码在前几章已经出现过，只是为单独挑出来提动态派发的概念。

```go
func main() {
	var s Person = &Student{name: "halfrost"}
	s.sayHello("everyone")
}
```

将上述代码转换成汇编代码以后，根据汇编代码画出内存布局图，如下图：

![](https://img.halfrost.com/Blog/ArticleImage/147_4_1.png)

找到方法调用的汇编代码：

```go
0x0075 00117 (main.go:7)    MOVQ    "".s+56(SP), AX
0x007a 00122 (main.go:7)    TESTB   AL, (AX)
0x007c 00124 (main.go:7)    MOVQ    32(AX), AX
0x0080 00128 (main.go:7)    MOVQ    "".s+64(SP), CX
0x0085 00133 (main.go:7)    MOVQ    CX, (SP)
0x0089 00137 (main.go:7)    LEAQ    go.string."everyone"(SB), CX
0x0090 00144 (main.go:7)    MOVQ    CX, 8(SP)
0x0095 00149 (main.go:7)    MOVQ    $8, 16(SP)
0x00a0 00160 (main.go:7)    CALL    AX
```

在上面代码中可以看到，为了调用动态派发的方法，AX 寄存器根据 \*itab 里面存的 func 指针，做了一次寻址的过程，32(AX) 找到了要派发的方法的地址。然后将方法需要的入参都放入栈顶。如果此处不做动态派发，汇编代码会有什么不同的处理逻辑呢？把代码改成下面这样：

```go
func main() {
	var s *Student = &Student{name: "halfrost"}
	s.sayHello("everyone")
}
```

转成汇编代码以后取出方法调用那一行的代码如下：

```go
0x004b 00075 (main.go:20)	MOVQ	AX, (SP)
0x004f 00079 (main.go:20)	LEAQ	go.string."everyone"(SB), AX
0x0056 00086 (main.go:20)	MOVQ	AX, 8(SP)
0x005b 00091 (main.go:20)	MOVQ	$8, 16(SP)
0x0064 00100 (main.go:20)	PCDATA	$1, $0
0x0064 00100 (main.go:20)	CALL	"".(*Student).sayHello(SB)
```

可以看到代码少了方法寻址的过程，这里直接将入参放入栈顶，调用方法。

总结：fun 指针保存的是实体类型实现的函数列表首地址，可以通过寻址找到要调用的方法。当函数传入不同的实体类型时，调用的实际上是不同的函数实现，从而实现多态。


关于动态派发的过程，其实有 2 部分的性能损失，一部分是上面提到的，动态调用方法。这是一个函数指针的间接调用，还要经过地址偏移动态计算以后的跳转。还有一部分是构造 iface 的过程。在第一种动态派发的代码中，内存中构造了一个完整的 iface。而在第二种直接方法调用的代码中，并没有构造 iface，直接把入参放入栈顶，直接调用那个方法。针对这 2 部分的性能损失，可能有读者会担心损耗很大。在 Github 上有不少人公布了关于这里的性能测试代码。笔者不贴完整测试代码了。直接说结论：

- 指针实现的动态派发造成的性能损失非常小，相对于一些复杂逻辑的处理函数，这点性能损失几乎可以忽略不计。
- 结构体实现的动态派发性能损耗比较大。结构体在方法调用的时候需要传值，拷贝参数，这里导致性能损失比较大。

所以在开发中，所有动态派发的代码用指针来实现。

至此，所有关于 interface 底层原理的部分都讲解完了。interface 的应用将放在反射的文章里面讲解。