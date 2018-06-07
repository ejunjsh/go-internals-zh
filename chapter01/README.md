````Bash
$ go version
go version go1.10 linux/amd64
````

# 章节01:入门go汇编 

在我们开始研究运行时和标准库的实现之前，对Go的抽象汇编语言的熟悉是必须的。这本快速指南应该有望让您更快的理解。

---

**目录**
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- ["伪汇编"](#%E4%BC%AA%E6%B1%87%E7%BC%96)
- [分解一个简单的程序](#%E5%88%86%E8%A7%A3%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84%E7%A8%8B%E5%BA%8F)
  - [解剖`add`](#%E8%A7%A3%E5%89%96add)
  - [解剖`main`](#%E8%A7%A3%E5%89%96main)
- [总结goroutines，堆栈和分裂](#%E6%80%BB%E7%BB%93goroutines%E5%A0%86%E6%A0%88%E5%92%8C%E5%88%86%E8%A3%82)
  - [堆栈](#%E5%A0%86%E6%A0%88)
  - [分裂](#%E5%88%86%E8%A3%82)
    - [序言（prologue）](#%E5%BA%8F%E8%A8%80prologue)
    - [结语（epilogue）](#%E7%BB%93%E8%AF%ADepilogue)
  - [减少一些微妙之处](#%E5%87%8F%E5%B0%91%E4%B8%80%E4%BA%9B%E5%BE%AE%E5%A6%99%E4%B9%8B%E5%A4%84)
- [结论](#%E7%BB%93%E8%AE%BA)
- [链接](#%E9%93%BE%E6%8E%A5)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

---

* 本章假设你有汇编程序的基本知识。
* 如果并且当遇到架构特定的问题时，总是假设linux/amd64。
* 我们将始终致力于启用编译器优化。
* 除非另有说明，否则引用的文字和/或注释始终来自官方文档和/或代码库。

## "伪汇编"
Go编译器输出一个抽象的，可移植的汇编形式，实际上并不映射到任何真实的硬件。Go汇编程序然后使用这个伪汇编输出来生成针对目标硬件的具体的，特定于机器的指令。
这个额外的层有许多好处，主要的好处是移植到新体系结构是多么容易。有关更多信息，请参阅本章末尾链接中Rob Pike写的The Design of the Go Assembler。

> 关于Go的汇编器最重要的一点是，它不是底层机器的直接表示。一些细节精确地映射到机器，但有些则不。这是因为编译器套件在通常的管道中不需要汇编程序。相反，编译器在一种半抽象指令集上运行，指令选择部分在代码生成后发生。汇编程序工作在半抽象的形式上，所以当你看到像MOV这样的指令时，工具链实际上为该操作产生的内容可能根本不是移动指令，可能是清除或加载。或者它可能完全对应于具有该名称的机器指令。一般来说，特定于机器的操作往往表现为自己，而像内存移动和子程序调用和返回等更一般的概念则更为抽象。细节因体系结构而异，我们对于不精确性表示歉意; 情况并不明确。

> 汇编程序是一种解析半抽象指令集的描述并将其转变为输入到链接器的指令的方式。

## 分解一个简单的程序
考虑以下Go代码（[direct_topfunc_call.go](https://github.com/teh-cmc/go-internals/blob/master/chapter1_assembly_primer/direct_topfunc_call.go)）：
````go
//go:noinline
func add(a, b int32) (int32, bool) { return a + b, true }

func main() { add(10, 32) }
````
（注意`//go:noinline`这里的编译器指令......不要被咬伤。(译者：原文是`Don't get bitten.`,不知道怎么翻译)）
让我们先编译成汇编先：
````shell
$ GOOS=linux GOARCH=amd64 go tool compile -S direct_topfunc_call.go
````
````Assembly
0x0000 TEXT		"".add(SB), NOSPLIT, $0-16
  0x0000 FUNCDATA	$0, gclocals·f207267fbf96a0178e8758c6e3e0ce28(SB)
  0x0000 FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
  0x0000 MOVL		"".b+12(SP), AX
  0x0004 MOVL		"".a+8(SP), CX
  0x0008 ADDL		CX, AX
  0x000a MOVL		AX, "".~r2+16(SP)
  0x000e MOVB		$1, "".~r3+20(SP)
  0x0013 RET

0x0000 TEXT		"".main(SB), $24-0
  ;; ...omitted stack-split prologue...
  0x000f SUBQ		$24, SP
  0x0013 MOVQ		BP, 16(SP)
  0x0018 LEAQ		16(SP), BP
  0x001d FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
  0x001d FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
  0x001d MOVQ		$137438953482, AX
  0x0027 MOVQ		AX, (SP)
  0x002b PCDATA		$0, $0
  0x002b CALL		"".add(SB)
  0x0030 MOVQ		16(SP), BP
  0x0035 ADDQ		$24, SP
  0x0039 RET
  ;; ...omitted stack-split epilogue...
````
我们将逐行分析这两个函数，以便更好地理解编译器正在做什么。

### 解剖`add`
````Assembly
0x0000 TEXT "".add(SB), NOSPLIT, $0-16
````
* 0x0000：当前指令的偏移，相对于函数的开始。
* TEXT "".add：TEXT指令将该"".add符号声明为该部分的.text一部分（即可运行代码），并指示后面的指令是该函数的主体。在链接时，空字符串""将被当前包的名称替换：即，一旦链接到我们的最终二进制文件中，"".add就会变成空字符串main.add。
* (SB)：SB是保存“静态基址（static-base）”指针的虚拟寄存器，即程序地址空间开始的地址。"".add(SB)声明我们的符号位于从地址空间开始的某个常量偏移量处（由链接器计算）。换句话说，它有一个绝对的直接地址：它是一个全局函数符号。 objdump将为我们确认所有这一切：
````
$ objdump -j .text -t direct_topfunc_call | grep 'main.add'
000000000044d980 g     F .text	000000000000000f main.add
````
>所有用户定义的符号被写为伪寄存器FP(参数和本地)和SB(全局)的偏移量。SB伪寄存器可以被认为是内存的起源，所以符号`foo(SB)`表示名称为foo的函数在内存中的地址。

* NOSPLIT：用来告诉编译器不应该插入栈分裂（stack-split）前导码，这个前导码用来检查当前堆栈是否需要生长。在我们的add函数中，编译器自己设置了标志：它足够聪明，因为add它没有本地变量，也没有自己的栈帧，它不能超过当前栈; 因此在每个调用点运行这些检查将会完全浪费CPU周期。
> "NOSPLIT"：不要插入前导码来检查堆栈是否必须拆分。例程的栈桢，加上它调用的任何东西，都必须放在堆栈段顶部的空闲空间中。用于保护例程，如堆栈分割代码本身。本章末尾，我们将简单介绍一下goroutines和stack-split。

* $0-16：$0表示将被分配的堆栈帧的大小（以字节为单位）同时$16指定调用者传入的参数的大小。
> 在一般情况下，帧大小后面跟着一个由减号分隔的参数大小。（这不是一个减法，只是特殊的语法。）框架大小$24-8表明该函数有一个24字节的帧，并且调用8个字节的参数，它位于调用者的帧上。如果未为TEXT指定NOSPLIT，则必须提供参数大小。对于使用Go原型的汇编函数，go vet会检查参数大小是否正确。
````Assembly
0x0000 FUNCDATA $0, gclocals·f207267fbf96a0178e8758c6e3e0ce28(SB)
0x0000 FUNCDATA $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
````
> FUNCDATA和PCDATA指令包含供垃圾收集器使用的信息; 它们由编译器引入。

现在不要担心这个; 我们稍后会在书中回顾垃圾收集时再回到它。
````Assembly
0x0000 MOVL "".b+12(SP), AX
0x0004 MOVL "".a+8(SP), CX
````
Go调用约定规定每个参数都必须在堆栈上传递，使用调用者堆栈中的预留空间。调用者有责任适当增大（并缩小）栈，以便将参数传递给被调用者，并将潜在的返回值传递给调用者。

Go编译器永远不会从PUSH/POP系列产生指令：通过分别递减或递增虚拟堆栈指针来增加或缩小堆栈SP。
> SP伪寄存器是一个虚拟堆栈指针，用于引用帧本地变量和为函数调用准备的参数。它指向本地堆栈帧的顶部，因此引用应使用范围`[-framesize，0）`中的负偏移量：x-8（SP），y-4（SP）等等。

虽然官方文档指出：“ 所有用户定义的符号都被写为伪寄存器FP（参数和本地数据）的偏移量 ”，这对于手写代码来说曾经是对的。和最新的编译器一样，Go工具套件总是直接在其生成的代码中使用来自堆栈指针的偏移量引用参数和本地值。这允许帧指针在具有较少寄存器（例如x86）的平台上用作额外的通用寄存器。如果你喜欢这种类型的细节，请看本章最后的链接__Stack frame layout on x86-64__。
我们会在下面的链接继续讨论这个问题[issue #2: issue #2: Frame pointer](https://github.com/teh-cmc/go-internals/issues/2)

`"".b+12(SP)`和`"".a+8(SP)`分别指向堆栈顶部以下12个字节和8个字节的地址（请记住：栈向下增长！）。`.a`并且`.b`是被指定位置的任意别名; 虽然它们绝对没有任何语义含义，但在虚拟寄存器上使用相对寻址时，它们是强制性的。关于虚拟帧指针的文档可以这样说：
> FP伪寄存器是用于引用函数参数的虚拟帧指针。编译器维护一个虚拟帧指针，并将堆栈中的参数作为该伪寄存器的偏移量。因此，0（FP）是函数的第一个参数，8（FP）是第二个参数（在64位机器上），依此类推。但是，以这种方式引用函数参数时，必须在first_arg + 0（FP）和second_arg + 8（FP）中放置一个名称。（帧指针的偏移量 - 偏移量的含义与它在SB中的使用截然不同，它与符号的偏移量）。汇编程序强制执行这种约定，拒绝纯0（FP）和8（FP）。实际名称在语义上不相关，但应该用于记录参数的名称。

最后，在这里需要注意两件重要的事情：
1. 第一个参数a不在于0(SP)，而在于8(SP); 这是因为调用者通过CALL伪指令存储其返回地址在0(SP)。
2. 参数以相反的顺序传递; 即第一个参数是最靠近栈顶的。

````Assembly
0x0008 ADDL CX, AX
0x000a MOVL AX, "".~r2+16(SP)
0x000e MOVB $1, "".~r3+20(SP)
````

`ADDL`执行存储在`AX`和`CX`中的两个长字（即4个字节值）的加法运算，然后将最终结果存储在`AX`中。 然后将结果移至`"".~r2+16(SP)`，这个地方是调用方先前已预留了一些堆栈空间并希望通过它找到其返回值。再一次，`"".~r2`这里没有语义含义。
为了演示Go如何处理多个返回值，我们也返回一个常量true布尔值。机制与我们的第一个返回值完全相同; 只是相对于SP的偏移量不同。
````Assembly
0x0013 RET
````
最终的`RET`伪指令告诉Go汇编器插入目标平台的调用约定所需的任何指令，以便正确地从子例程调用中返回。
这很可能会导致代码弹出存储在`0(SP)`处的返回地址，然后跳转回去。

> TEXT块中的最后一条指令必须是某种跳转，通常是RET（伪指令）。（如果不是，链接器会附加一条跳转到自身的指令; TEXT中不会出现跳转。）

一次性说了那么多语法和语义的东西。接下来是我们刚才介绍的内容摘要：

````Assembly
;; Declare global function symbol "".add (actually main.add once linked)
;; Do not insert stack-split preamble
;; 0 bytes of stack-frame, 16 bytes of arguments passed in
;; func add(a, b int32) (int32, bool)
0x0000 TEXT	"".add(SB), NOSPLIT, $0-16
  ;; ...omitted FUNCDATA stuff...
  0x0000 MOVL	"".b+12(SP), AX	    ;; move second Long-word (4B) argument from caller's stack-frame into AX
  0x0004 MOVL	"".a+8(SP), CX	    ;; move first Long-word (4B) argument from caller's stack-frame into CX
  0x0008 ADDL	CX, AX		    ;; compute AX=CX+AX
  0x000a MOVL	AX, "".~r2+16(SP)   ;; move addition result (AX) into caller's stack-frame
  0x000e MOVB	$1, "".~r3+20(SP)   ;; move `true` boolean (constant) into caller's stack-frame
  0x0013 RET			    ;; jump to return address stored at 0(SP)
````
总而言之，下面是可视化表示堆栈在main.add完成执行时的样子：
````
   |    +-------------------------+ <-- 32(SP)              
   |    |                         |                         
 G |    |                         |                         
 R |    |                         |                         
 O |    | main.main's saved       |                         
 W |    |     frame-pointer (BP)  |                         
 S |    |-------------------------| <-- 24(SP)              
   |    |      [alignment]        |                         
 D |    | "".~r3 (bool) = 1/true  | <-- 21(SP)              
 O |    |-------------------------| <-- 20(SP)              
 W |    |                         |                         
 N |    | "".~r2 (int32) = 42     |                         
 W |    |-------------------------| <-- 16(SP)              
 A |    |                         |                         
 R |    | "".b (int32) = 32       |                         
 D |    |-------------------------| <-- 12(SP)              
 S |    |                         |                         
   |    | "".a (int32) = 10       |                         
   |    |-------------------------| <-- 8(SP)               
   |    |                         |                         
   |    |                         |                         
   |    |                         |                         
 \ | /  | return address to       |                         
  \|/   |     main.main + 0x30    |                         
   -    +-------------------------+ <-- 0(SP) (TOP OF STACK)

(diagram made with https://textik.com)
````

### 解剖`main`
为了节省不必要的拉上去看代码，我们把上面的`main`函数放到下面了：
````Assembly
0x0000 TEXT		"".main(SB), $24-0
  ;; ...omitted stack-split prologue...
  0x000f SUBQ		$24, SP
  0x0013 MOVQ		BP, 16(SP)
  0x0018 LEAQ		16(SP), BP
  ;; ...omitted FUNCDATA stuff...
  0x001d MOVQ		$137438953482, AX
  0x0027 MOVQ		AX, (SP)
  ;; ...omitted PCDATA stuff...
  0x002b CALL		"".add(SB)
  0x0030 MOVQ		16(SP), BP
  0x0035 ADDQ		$24, SP
  0x0039 RET
  ;; ...omitted stack-split epilogue...
````
````Assembly
0x0000 TEXT "".main(SB), $24-0
````
这里没有新东西：
* `"".main`（一旦链接就变成main.main）是该.text部分中的全局函数符号，其地址与我们地址空间的起始位置有一定的偏移量。
* 它分配一个24字节的栈帧，不会收到任何参数，也不会返回任何值。

````Assembly
0x000f SUBQ     $24, SP
0x0013 MOVQ     BP, 16(SP)
0x0018 LEAQ     16(SP), BP
````
正如我们上面提到的，Go调用约定要求每个参数必须在堆栈上传递。

我们的调用者通过减少虚拟堆栈指针来`main`增加堆栈帧24个字节（记住堆栈向下增长，所以`SUBQ`这实际上使得堆栈帧变大）。这24个字节中：
* 8个字节（16(SP)- 24(SP)）用于存储帧指针BP（真实的！）的当前值，以允许堆栈展开并便于调试
* 1 + 3字节（12(SP)- 16(SP)）保留给第二个返回值（bool）加上必要的对齐3字节amd64
* 4字节（8(SP)- 12(SP)）保留为第一个返回值（int32）
* 4字节（4(SP)- 8(SP)）保留为参数的值b (int32)
* 4字节（0(SP)- 4(SP)）保留为参数的值a (int32)

最后，在堆栈增长之后，`LEAQ`计算帧指针的新地址并将其存储`BP`。
````Assembly
0x001d MOVQ     $137438953482, AX
0x0027 MOVQ     AX, (SP)
````
调用者将被调用者的参数用一个四字（即8字节值）推送到刚刚增长的堆栈顶部。
虽然它可能看起来就像是随机的垃圾，137438953482实际上对应于`10`和`32`这两个4字节整数参数连接成一个8字节的值：
````shell
$ echo 'obase=2;137438953482' | bc
10000000000000000000000000000000001010
\_____/\_____________________________/
   32                             10
````
````Assembly
0x002b CALL     "".add(SB)
````
我们`CALL`的`add`函数作为相对于静态基址指针的偏移量：即，这是直接跳转到直接地址的直接函数。

注意`CALL`也会把返回地址（8字节值）推到栈顶，所以每个在`add`函数引用`SP`的地方都要以位移量为8来结束
例如参数`"".a`不再试`0(SP)`, 而是 `8(SP)`.

````Assembly
0x0030 MOVQ     16(SP), BP
0x0035 ADDQ     $24, SP
0x0039 RET
````
最后，我们：
1. 恢复帧指针原来的值
2. 将堆栈缩小24个字节以回收先前分配的堆栈空间
3. 请Go编译器插入子例程返回相关的东西

## 总结goroutines，堆栈和分裂
现在不是深入研究goroutines内部结构的时间和地点（后面会讲到），但随着我们开始越来越多地考虑汇编转储，与堆栈管理有关的指令将迅速成为非常熟悉的视图。
我们应该能够快速识别这些模式，并且在我们掌握时了解他们所做的一般想法以及他们为什么这样做。

### 堆栈
由于goroutine的数量在一个go程序中是非确定的或者可能在现实中达到几百万个，所以运行时必须采取保守的做法去分配栈空间给goroutine，否则会吃光可用的内存。

例如每个新的goroutine在运行时的初始大小为2KB（实际上的内幕是栈在堆中分配的）

假设一个goroutine一直运行，可能最后会使用超过刚开始分配的内存（栈溢出）。为了防止这种事情发生，运行时保证一个goroutine在消耗完一个栈的时候，一个两倍大的新栈分配出来，原来的栈的内容会拷贝到新栈上面。

这过程称为栈分配（stack-split）而且这样可以有效的使goroutine栈成为动态大小的栈。

### 分裂
为了使栈分裂工作，编译器会在每个可能溢出堆栈的函数的开始和结尾处插入一些指令。

正如我们在本章前面所看到的那样，为了避免不必要的开销，不可能超出堆栈的函数被标记为`NOSPLIT`提示编译器不要插入这些检查。

让我们先看看我们的`main`函数，这次不需要忽略栈分裂前导码：

````Assembly
0x0000 TEXT	"".main(SB), $24-0
  ;; stack-split prologue
  0x0000 MOVQ	(TLS), CX
  0x0009 CMPQ	SP, 16(CX)
  0x000d JLS	58

  0x000f SUBQ	$24, SP
  0x0013 MOVQ	BP, 16(SP)
  0x0018 LEAQ	16(SP), BP
  ;; ...omitted FUNCDATA stuff...
  0x001d MOVQ	$137438953482, AX
  0x0027 MOVQ	AX, (SP)
  ;; ...omitted PCDATA stuff...
  0x002b CALL	"".add(SB)
  0x0030 MOVQ	16(SP), BP
  0x0035 ADDQ	$24, SP
  0x0039 RET

  ;; stack-split epilogue
  0x003a NOP
  ;; ...omitted PCDATA stuff...
  0x003a CALL	runtime.morestack_noctxt(SB)
  0x003f JMP	0
````
正如你所看到的，栈分裂前导码被分成了序言（prologue）和结语（epilogue）：
* 序言检查是否空间不足，如果是这样的话，跳到结语。
* 另一方面，结语则触发栈扩容，然后跳回序言。

这会创建一个反馈循环，只要没有为我们饥饿的goroutine分配足够大的堆栈，就会继续。

#### 序言（prologue）
````Assembly
0x0000 MOVQ	(TLS), CX   ;; store current *g in CX
0x0009 CMPQ	SP, 16(CX)  ;; compare SP and g.stackguard0
0x000d JLS	58	    ;; jumps to 0x3a if SP <= g.stackguard0
````
`TLS`是一个由运行时维护的虚拟寄存器，它包含一个指向当前的指针`g`，即跟踪一个goroutine所有状态的数据结构。

看一下`g`的定义，它来自于runtime的源代码：
````go
type g struct {
	stack       stack   // 16 bytes
	// stackguard0 is the stack pointer compared in the Go stack growth prologue.
	// It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
	stackguard0 uintptr
	stackguard1 uintptr

	// ...omitted dozens of fields...
}
````
我们可以看到`16(CX)`对应于`g.stackguard0`,它是由运行时维护的一个阀值。当它与栈指针比较的时候，代表是否一个goroutine就要花光内存了。这个序言会检查s是否当前`SP`的值小于或者等于`stackguard0`阀值（它应该更大）,如果是，就跳到结语那里。

#### 结语（epilogue）
````Assembly
0x003a NOP
0x003a CALL	runtime.morestack_noctxt(SB)
0x003f JMP	0
````
结语的内容很直接：它调用一个运行时函数，去扩容栈空间，然后跳回到函数的第一个指令(即序言)

`CALL`指令前面的`NOP`指令的存在，目的是令序言不会直接跳进`CALL`指令。在一些平台，这样做会有可能导致问题；但是用一个`NOP`指令放在实际调用前面，已经是一个通用的做法了。

我们会在下面的链接继续讨论这个问题[issue #4: Clarify "nop before call" paragraph](https://github.com/teh-cmc/go-internals/issues/4)


### 减少一些微妙之处
我们仅仅在这里介绍了冰山一角。

栈增长的内在机制有更多的微妙之处，我们在这里甚至没有提到：整个过程是一个相当复杂的，需要一个自己的章节。

我们会及时回到这些问题上。

## 结论
Go的汇编程序的这个快速介绍应该给你足够的资料开始四处讨论。

随着我们对本书其余部分Go的内部知识的深入挖掘，Go的汇编将成为我们理解幕后发生的一切的最依赖工具之一。

如果您有任何问题或建议，请不要犹豫，开一个issue吧！

## 链接

- [[Official] A Quick Guide to Go's Assembler](https://golang.org/doc/asm)
- [[Official] Go Compiler Directives](https://golang.org/cmd/compile/#hdr-Compiler_Directives)
- [[Official] The design of the Go Assembler](https://www.youtube.com/watch?v=KINIAgRpkDA)
- [[Official] Contiguous stacks Design Document](https://docs.google.com/document/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/pub)
- [[Official] The `_StackMin` constant](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/stack.go#L70-L71)
- [[Discussion] Issue #2: *Frame pointer*](https://github.com/teh-cmc/go-internals/issues/2)
- [[Discussion] Issue #4: *Clarify "nop before call" paragraph*](https://github.com/teh-cmc/go-internals/issues/4)
- [A Foray Into Go Assembly Programming](https://blog.sgmansfield.com/2017/04/a-foray-into-go-assembly-programming/)
- [Dropping Down Go Functions in Assembly](https://www.youtube.com/watch?v=9jpnFmJr2PE)
- [What is the purpose of the EBP frame pointer register?](https://stackoverflow.com/questions/579262/what-is-the-purpose-of-the-ebp-frame-pointer-register)
- [Stack frame layout on x86-64](https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64)
- [How Stacks are Handled in Go](https://blog.cloudflare.com/how-stacks-are-handled-in-go/)
- [Why stack grows down](https://gist.github.com/cpq/8598782)

