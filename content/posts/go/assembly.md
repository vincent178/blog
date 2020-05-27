---
title: "浅谈 Go 语言的汇编"
date: 2020-01-01T00:00:00+08:00
draft: false
---

<small>命令默认跑在 MacBook Pro (15-inch, 2018)，Intel Core i7</small>
<br>
<small>go version go1.13.1 darwin/amd64</small>

因为我之前没有学过汇编，所以文中难免有错误或者疏漏的地方，如果你发现有疑问，尽情的提出来吧。

> Assembler is how you talk to machine in the lowest level.
>    -- Rob Pike

当然，学习和理解 Go 的汇编并不是为了手撸汇编，而是给我们一种了解语言的工具。这是我身边同学的两个问题：

1. new 操作符和初始化结构体的异同
```go
type Person struct {
  Name string
}

// what's the difference between
new(Person)

// and
&Person{}
```

2. Go 的结构体 method 和普通的参数异同

```go
// what's the difference between
func (p *Person) name() string {
  return p.Name
}

// and
func name(p *Person) string {
  return p.Name
}
```
我们尝试用这个工具来解释一下。

## 准备知识

### “伪”汇编
Go tool 生成的汇编是基于 [plan 9](https://9p.io/sys/doc/asm.html), 但不是真正可以机器执行的汇编，是一层对于不同汇编语言的抽象，比如 MOV 指令，在最终可能会被翻译成 clear 或者 load 指令。Rob Pike 在 GopherCon 2016 的 [Go 汇编设计](https://www.youtube.com/watch?v=KINIAgRpkDA)有详细介绍，这层额外的抽象帮助 Go 很容易的迁移到不同的平台。

### X86 栈顶
Intel X86 架构的栈是[头朝下的结构](https://eli.thegreenplace.net/2011/02/04/where-the-top-of-the-stack-is-on-x86/)
![stack top](./../images/stack_top.png)
栈是用 `SUB` 指令向下增加，用 `ADD` 指令缩小。

### 寄存器
这一部分是最让我困惑的，特别是看了[官方文档](https://golang.org/doc/asm)后，会更加无从下手。我觉得说的比较清楚的是[go-internals](https://github.com/teh-cmc/go-internals/tree/master/chapter1_assembly_primer)，

* SB: 虚拟寄存器，可以视为内存，保存的是方法或者结构体等全局符号。[^1]: https://golang.org/doc/asm
* BP
* SP

## 举个例子

```go
package main

//go:noinline
func add(a, b int32) int32 {
  return a + b
}

func main() {
  add(1, 2)
}
```
在命令行中调用`go tool compile -S main.go`, 裁剪掉不重要的内容后的输出如下

```assembly
"".add STEXT nosplit size=15 args=0x10 locals=0x0
        0x0000 00000 (main.go:4)        TEXT    "".add(SB), NOSPLIT|ABIInternal, $0-16
        0x0000 00000 (main.go:4)        FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (main.go:4)        FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (main.go:4)        FUNCDATA        $2, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (main.go:5)        PCDATA  $0, $0
        0x0000 00000 (main.go:5)        PCDATA  $1, $0
        0x0000 00000 (main.go:5)        MOVL    "".b+12(SP), AX
        0x0004 00004 (main.go:5)        MOVL    "".a+8(SP), CX
        0x0008 00008 (main.go:5)        ADDL    CX, AX
        0x000a 00010 (main.go:5)        MOVL    AX, "".~r2+16(SP)
        0x000e 00014 (main.go:5)        RET
"".main STEXT size=65 args=0x0 locals=0x18
        0x0000 00000 (main.go:8)        TEXT    "".main(SB), ABIInternal, $24-0
        0x000f 00015 (main.go:8)        SUBQ    $24, SP
        0x0013 00019 (main.go:8)        MOVQ    BP, 16(SP)
        0x0018 00024 (main.go:8)        LEAQ    16(SP), BP
        0x001d 00029 (main.go:8)        FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x001d 00029 (main.go:8)        FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x001d 00029 (main.go:8)        FUNCDATA        $2, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x001d 00029 (main.go:9)        PCDATA  $0, $0
        0x001d 00029 (main.go:9)        PCDATA  $1, $0
        0x001d 00029 (main.go:9)        MOVQ    $8589934593, AX
        0x0027 00039 (main.go:9)        MOVQ    AX, (SP)
        0x002b 00043 (main.go:9)        CALL    "".add(SB)
        0x0030 00048 (main.go:10)       MOVQ    16(SP), BP
        0x0035 00053 (main.go:10)       ADDQ    $24, SP
        0x0039 00057 (main.go:10)       RET
```

一行一行来看

### `add`

```
0x0000 00000 (main.go:4)        TEXT    "".add(SB), NOSPLIT|ABIInternal, $0-16
```
* 0x0000: 当前指令相对于当前函数的偏移量
* TEXT "".add(SB): TEXT 指令声明了 add 是 .text 段的一部分。SB 是一个虚拟寄存器，保存了静态基地址(static-base) 指针，即我们程序地址空间的开始地址。"".add(SB) 表明我们的符号位于某个固定的相对地址空间起始处的偏移位置。
* $0-16: $0 代表即将分配的栈帧大小，而 $16 指定了调用方传入的参数大小。

```
FUNCDATA
PCDATA
```
* FUNCDATA 以及 PCDATA 指令包含有被垃圾回收所使用的信息；这些指令是被编译器加入的。

```
0x0000 00000 (main.go:5)        MOVL    "".b+12(SP), AX
0x0004 00004 (main.go:5)        MOVL    "".a+8(SP), CX
0x0008 00008 (main.go:5)        ADDL    CX, AX
```
* SP 是一个存放栈顶指针的寄存器，比较混淆的是 Go 的[官方文档](https://golang.org/doc/asm) 有提到这是虚拟寄存器，但我比较倾向于 [讨论](https://github.com/teh-cmc/go-internals/issues/21) 中的结论，这个是硬件寄存器。
* MOVL: L 这里代表 Long，4 字节的值
* 参数是反序传入，a 离栈顶更近 8(SP)， b 是 12(SP)
* ADDL CX, AX: 相加并最终存入 AX 中。

```
0x000a 00010 (main.go:5)        MOVL    AX, "".~r2+16(SP)
```
* 将 AX 的值保存到 16(SP)

```
0x000e 00014 (main.go:5)        RET
```
* 方法调用方将返回地址保存在 0(SP)，RET 就是跳转到 0(SP) 中保存的地址

### `main`

```
0x000f 00015 (main.go:8)        SUBQ    $24, SP
```
* SUBQ 增加栈的容量


```
FP: Frame pointer: arguments and locals.
PC: Program counter: jumps and branches.
SB: Static base pointer: global symbols.
SP: Stack pointer: top of stack.
```

TEXT 指令声明了 add 是 .text 段的一部分

SB 是虚拟寄存器，存放的是 static base 静态基指针

$0 栈大小
$16 参数大小 (caller)

8 byte for arguments
8 byte for return value (内存对齐 memory alignment)

SP stack pointer
BP base frame

实际上, go 没有 FP, 而 SP 也是实际 x86 中 stack head down 模式

0(SP) return pointer
8(SP) a
12(SP) b
16(SP) return value

验证一下这个猜想
```go
package main

//go:noinline
func add(a, b int32) int32 {
	return a+b
}

func main() {
	x := add(1, 2)
	println(x)
}
```

```
"".main STEXT size=104 args=0x0 locals=0x20
        0x0000 00000 (main.go:8)        TEXT    "".main(SB), ABIInternal, $32-0
        0x0000 00000 (main.go:8)        MOVQ    (TLS), CX
        0x0009 00009 (main.go:8)        CMPQ    SP, 16(CX)
        0x000d 00013 (main.go:8)        JLS     97
        0x000f 00015 (main.go:8)        SUBQ    $32, SP
        0x0013 00019 (main.go:8)        MOVQ    BP, 24(SP)
        0x0018 00024 (main.go:8)        LEAQ    24(SP), BP
        0x001d 00029 (main.go:8)        FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x001d 00029 (main.go:8)        FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x001d 00029 (main.go:8)        FUNCDATA        $2, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x001d 00029 (main.go:9)        PCDATA  $0, $0
        0x001d 00029 (main.go:9)        PCDATA  $1, $0
        0x001d 00029 (main.go:9)        MOVQ    $8589934593, AX
        0x0027 00039 (main.go:9)        MOVQ    AX, (SP)
        0x002b 00043 (main.go:9)        CALL    "".add(SB)
        0x0030 00048 (main.go:9)        MOVLQSX 8(SP), AX
        0x0035 00053 (main.go:9)        MOVQ    AX, ""..autotmp_1+16(SP)
        0x003a 00058 (main.go:10)       CALL    runtime.printlock(SB)
        0x003f 00063 (main.go:10)       MOVQ    ""..autotmp_1+16(SP), AX
        0x0044 00068 (main.go:10)       MOVQ    AX, (SP)
        0x0048 00072 (main.go:10)       CALL    runtime.printint(SB)
        0x004d 00077 (main.go:10)       CALL    runtime.printnl(SB)
        0x0052 00082 (main.go:10)       CALL    runtime.printunlock(SB)
        0x0057 00087 (main.go:11)       MOVQ    24(SP), BP
        0x005c 00092 (main.go:11)       ADDQ    $32, SP
        0x0060 00096 (main.go:11)       RET

```

关注这一行
MOVLQSX 8(SP), AX 验证了前面的猜想

再回到最开始的问题

&Person{} 和 new(Person) 是一样的

```
        0x001d 00029 (main.go:14)       MOVL    $0, ""..autotmp_3+12(SP)
        0x0025 00037 (main.go:14)       MOVL    $0, ""..autotmp_3+12(SP)
        0x002d 00045 (main.go:15)       MOVL    $0, ""..autotmp_4+8(SP)
"".(*Person).name STEXT nosplit size=1 args=0x8 locals=0x0
        0x0000 00000 (main.go:7)        TEXT    "".(*Person).name(SB), NOSPLIT|ABIInternal, $0-8
        0x0000 00000 (main.go:7)        FUNCDATA        $0, gclocals·2a5305abe05176240e61b8620e19a815(SB)
        0x0000 00000 (main.go:7)        FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (main.go:7)        FUNCDATA        $2, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (main.go:8)        RET
        0x0000 c3                                               .
"".name STEXT nosplit size=1 args=0x8 locals=0x0
        0x0000 00000 (main.go:11)       TEXT    "".name(SB), NOSPLIT|ABIInternal, $0-8
        0x0000 00000 (main.go:11)       FUNCDATA        $0, gclocals·2a5305abe05176240e61b8620e19a815(SB)
        0x0000 00000 (main.go:11)       FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (main.go:11)       FUNCDATA        $2, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (main.go:12)       RET
        0x0000 c3                                               .
"".main STEXT size=74 args=0x0 locals=0x10
        0x0000 00000 (main.go:20)       TEXT    "".main(SB), ABIInternal, $16-0
        0x0000 00000 (main.go:20)       MOVQ    (TLS), CX
        0x0009 00009 (main.go:20)       CMPQ    SP, 16(CX)
        0x000d 00013 (main.go:20)       JLS     67
        0x000f 00015 (main.go:20)       SUBQ    $16, SP
        0x0013 00019 (main.go:20)       MOVQ    BP, 8(SP)
        0x0018 00024 (main.go:20)       LEAQ    8(SP), BP
        0x001d 00029 (main.go:20)       FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x001d 00029 (main.go:20)       FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x001d 00029 (main.go:20)       FUNCDATA        $2, gclocals·9fb7f0986f647f17cb53dda1484e0f7a(SB)
        0x001d 00029 (main.go:22)       PCDATA  $0, $1
        0x001d 00029 (main.go:22)       PCDATA  $1, $0
        0x001d 00029 (main.go:22)       LEAQ    ""..autotmp_2+8(SP), AX
        0x0022 00034 (main.go:22)       PCDATA  $0, $0
        0x0022 00034 (main.go:22)       MOVQ    AX, (SP)
        0x0026 00038 (main.go:22)       CALL    "".(*Person).name(SB)
        0x002b 00043 (main.go:23)       PCDATA  $0, $1
        0x002b 00043 (main.go:23)       LEAQ    ""..autotmp_2+8(SP), AX
        0x0030 00048 (main.go:23)       PCDATA  $0, $0
        0x0030 00048 (main.go:23)       MOVQ    AX, (SP)
        0x0034 00052 (main.go:23)       CALL    "".name(SB)
        0x0039 00057 (main.go:24)       MOVQ    8(SP), BP
        0x003e 00062 (main.go:24)       ADDQ    $16, SP
        0x0042 00066 (main.go:24)       RET
```

```go
//go:noinline
func (p *Person) name(age int32) bool{
	return p.age == age
}
```

```
"".(*Person).name STEXT nosplit size=17 args=0x18 locals=0x0
        0x0000 00000 (main.go:8)        TEXT    "".(*Person).name(SB), NOSPLIT|ABIInternal, $0-24
        0x0000 00000 (main.go:8)        FUNCDATA        $0, gclocals·1a65e721a2ccc325b382662e7ffee780(SB)
        0x0000 00000 (main.go:8)        FUNCDATA        $1, gclocals·69c1753bd5f81501d95132d08af04464(SB)
        0x0000 00000 (main.go:8)        FUNCDATA        $2, gclocals·568470801006e5c0dc3947ea998fe279(SB)
        0x0000 00000 (main.go:9)        PCDATA  $0, $0
        0x0000 00000 (main.go:9)        PCDATA  $1, $0
        0x0000 00000 (main.go:9)        MOVL    "".age+16(SP), AX
        0x0004 00004 (main.go:9)        PCDATA  $0, $1
        0x0004 00004 (main.go:9)        PCDATA  $1, $1
        0x0004 00004 (main.go:9)        MOVQ    "".p+8(SP), CX
        0x0009 00009 (main.go:9)        PCDATA  $0, $0
        0x0009 00009 (main.go:9)        CMPL    (CX), AX
        0x000b 00011 (main.go:9)        SETEQ   "".~r1+24(SP)
        0x0010 00016 (main.go:9)        RET
```

* https://golang.org/doc/asm
* https://eli.thegreenplace.net/2011/02/04/where-the-top-of-the-stack-is-on-x86/
* https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64
* https://www.quora.com/What-is-a-quadword-in-64-bit-assembly-language#
* http://www.mit.edu/afs.new/sipb/project/golang/doc/asm.html
