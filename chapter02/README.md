````Bash
$ go version
go version go1.10 linux/amd64
````

# 章节02: 接口

这一章覆盖了go接口的内在原理

特别的，我们会看到：
- 函数和方法在运行时是怎么被调用的
- 接口是怎么被创建和由什么组成
- 动态分配是怎么工作，什么时候工作，会造成什么代价
- 空接口和其他特殊情况是如何与其他接口类型不同
- 接口组合是怎么工作的
- 类型声明是怎么工作的，有什么代价

随着我们的不断深入，我们会理解各种底层观点，例如一些现代cpu的实现细节和各种go编译器的优化技术

---

**目录**
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [函数和方法调用](#%E5%87%BD%E6%95%B0%E5%92%8C%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8)
  - [直接调用概述](#%E7%9B%B4%E6%8E%A5%E8%B0%83%E7%94%A8%E6%A6%82%E8%BF%B0)
    - [直接调用顶层函数](#%E7%9B%B4%E6%8E%A5%E8%B0%83%E7%94%A8%E9%A1%B6%E5%B1%82%E5%87%BD%E6%95%B0)
    - [带有指针接收器的直接方法调用](#%E5%B8%A6%E6%9C%89%E6%8C%87%E9%92%88%E6%8E%A5%E6%94%B6%E5%99%A8%E7%9A%84%E7%9B%B4%E6%8E%A5%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8)
    - [带有值接收器的方法调用](#%E5%B8%A6%E6%9C%89%E5%80%BC%E6%8E%A5%E6%94%B6%E5%99%A8%E7%9A%84%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8)
  - [隐式解引用](#%E9%9A%90%E5%BC%8F%E8%A7%A3%E5%BC%95%E7%94%A8)
    - [情况A:接收器在栈中](#%E6%83%85%E5%86%B5a%E6%8E%A5%E6%94%B6%E5%99%A8%E5%9C%A8%E6%A0%88%E4%B8%AD)
    - [情况B:接收器在堆上](#%E6%83%85%E5%86%B5b%E6%8E%A5%E6%94%B6%E5%99%A8%E5%9C%A8%E5%A0%86%E4%B8%8A)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

---

* 这章节假设你已经熟悉go的编译器（[章节01](https://github.com/ejunjsh/go-internals-zh/tree/master/chapter01)）
* 如果并且当遇到架构特定的问题时，总是假设linux/amd64。
* 我们将始终致力于启用编译器优化。
* 除非另有说明，否则引用的文字和/或注释始终来自官方文档和/或代码库。


## 函数和方法调用
正如Russ Cox在其关于函数调用的设计文档（本章最后列出）中指出的那样，go具有..：

..4种不同种类的函数..：
> - 顶层func
> - 带有值接收器的方法
> - 带有指针接收器的方法
> - 匿名func（译者注：英文原文叫这个func literal）

..和5个不同种类的调用:
> - 顶层func的直接调用（`func TopLevel(x int) {}`）
> - 带有值接收器的直接方法调用（`func (Value) M(int) {}`）
> - 带有指针接收器的直接方法调用（`func (*Pointer) M(int) {}`）
> - 接口方法的间接调用（`type Interface interface { M(int) }`）
> - func类型的值的间接调用（`var literal = func(x int) {}`）

混合在一起，这些组合了10种可能的函数和调用类型组合：
> - 顶层func的直接调用 /
> - 带有值接收器的直接方法调用/
> - 带有指针接收器的直接方法调用/
> - 接口方法的间接调用/包含value方法的方法
> - 接口方法的间接调用/包含带value方法的指针
> - 接口方法的间接调用/包含带指针方法的指针
> - func类型的值的间接调用/设置为顶层func
> - func类型的值的间接调用/设置为值方法
> - func类型的值的间接调用/设置为指针方法
> - func类型的值的间接调用/设置为匿名func

> （斜线将编译时已知的内容与运行时发现的内容区分开来。）

我们将首先花几分钟时间来回顾三种直接调用，然后我们将把焦点转向本章其余部分的接口和间接方法调用。

在本章中，我们不会涉及匿名函数，因为这样做首先需要我们熟悉闭包的机制，在适当的时候我们不可避免地会这样做。

### 直接调用概述
考虑下面的代码([direct_calls.go](https://github.com/ejunjsh/go-internals-zh/blob/master/chapter02/direct_calls.go))
````go
//go:noinline
func Add(a, b int32) int32 { return a + b }

type Adder struct{ id int32 }
//go:noinline
func (adder *Adder) AddPtr(a, b int32) int32 { return a + b }
//go:noinline
func (adder Adder) AddVal(a, b int32) int32 { return a + b }

func main() {
    Add(10, 32) // direct call of top-level function

    adder := Adder{id: 6754}
    adder.AddPtr(10, 32) // direct call of method with pointer receiver
    adder.AddVal(10, 32) // direct call of method with value receiver

    (&adder).AddVal(10, 32) // implicit dereferencing
}
````
让我们快速浏览一下为这4个调用生成的代码。

#### 直接调用顶层函数
看一下调用`Add(10, 32)`时的汇编：
````assembly
0x0000 TEXT	"".main(SB), $40-0
  ;; ...omitted everything but the actual function call...
  0x0021 MOVQ	$137438953482, AX
  0x002b MOVQ	AX, (SP)
  0x002f CALL	"".Add(SB)
  ;; ...omitted everything but the actual function call...
````
我们可以看到，正如我们在第一章中已经知道的那样，这将转化为在该`.text`部分中直接跳转到全局函数符号，参数和返回值存储在调用者的栈帧中。

它就像它一样简单。

Russ Cox 在他的文档上有提到
> 直接调用顶层func：一个顶层func调用是在栈中传递所有参数，并且结果也是放在接着之前都连续的栈上。

#### 带有指针接收器的直接方法调用
首先，接收器被通过`adder := Adder{id: 6754}`初始化:
````assembly
0x0034 MOVL	$6754, "".adder+28(SP)
````
（我们的栈帧上的额外空间是作为帧指针前导码的一部分预先分配的，这里我们列出来。）

然后来到实际的方法调用`adder.AddPtr(10, 32)`：
````assembly
0x0057 LEAQ	"".adder+28(SP), AX	;; move &adder to..
0x005c MOVQ	AX, (SP)		;; ..the top of the stack (argument #1)
0x0060 MOVQ	$137438953482, AX	;; move (32,10) to..
0x006a MOVQ	AX, 8(SP)		;; ..the top of the stack (arguments #3 & #2)
0x006f CALL	"".(*Adder).AddPtr(SB)
````
看一下汇编输出，我们能够清晰的看到方法的调用（无论是值接收器还是指针的）跟函数的调用几乎是一样的，唯一不同的是接收器会作为第一个参数传递。

在这个例子里，我们通过`LEAQ	"".adder+28(SP), AX`加载有效地址，并放到栈顶上。所以`&adder`就是参数#1（如果你对`LEA`和`MOV`搞不清楚，你可以看看文章最后的链接）

注意编译器怎么把接收器的类型编码进到符号名里面:`"".(*Adder).AddPtr`,无论接收器是值还是指针。

> Direct call of method: In order to use the same generated code for both an indirect call of a func value and for a direct call, the code generated for a method (both value and pointer receivers) is chosen to have the same calling convention as a top-level function with the receiver as a leading argument.

#### 带有值接收器的方法调用

正如我们想的，用值接收器产出跟上面类似的代码：

考虑一下`adder.AddVal(10, 32)`:
````assembly
0x003c MOVQ	$42949679714, AX	;; move (10,6754) to..
0x0046 MOVQ	AX, (SP)		;; ..the top of the stack (arguments #2 & #1)
0x004a MOVL	$32, 8(SP)		;; move 32 to the top of the stack (argument #3)
0x0052 CALL	"".Adder.AddVal(SB)
````
看上去有点怪，上面生成的汇编居然没有引用`"".adder+28(SP)`,这个是接收器的值放的地方。

究竟怎么回事呢？因为接收器是一个值类型，编译器可以推断它的值，而不用从它当前位置(`28(SP)`)拷贝已有的值。所以，编译器它会在栈中创建一个相同的`Adder`的值，然后跟第二个参数合并在一个整型（`42949679714`）中，这样可以节省更多的指令.

再一次注意下，一个方法的符号名显示地表示它期望一个值接收器。

### 隐式解引用
还有一个我们还没有看过的最后一个调用`(&adder).AddVal(10, 32)`。

在这个例子，我们用个指针变量去调用一个方法，虽然方法期望的是个值。Go用了某种方法自动的解引用这个指针，然后调用，怎么做到的呢？

编译器怎么处理这种情况依赖与接收器是在（逃逸到）堆还是栈里。

#### 情况A:接收器在栈中

如果接收器仍然在栈中，而它的大小足够小，并且能用很少指令来拷贝它，那么编译器只需将其值复制到堆栈的顶部，然后直接进行方法调用即可 `"".Adder.AddVal`(例如跟上面用值接收器一样)。

`(&adder).AddVal(10, 32)` 因此在这种情况下看起来像这样：

````assembly
0x0074 MOVL	"".adder+28(SP), AX	;; move (i.e. copy) adder (note the MOV instead of a LEA) to..
0x0078 MOVL	AX, (SP)		;; ..the top of the stack (argument #1)
0x007b MOVQ	$137438953482, AX	;; move (32,10) to..
0x0085 MOVQ	AX, 4(SP)		;; ..the top of the stack (arguments #3 & #2)
0x008a CALL	"".Adder.AddVal(SB)
````

无聊（尽管高效）。我们来看情况B.

#### 情况B:接收器在堆上

如果接收器逃脱到堆则编译器会采取更聪明的路线：它生成一个新方法（这一次用指针接收器），它包装了`"".Adder.AddVal`，并替换原来调用`"".Adder.AddVal`的地方，为调用自己`"".(*Adder).AddVal`。

这个包装方法的唯一任务就是确定接收器在传递给被包装的方法时候已经是解引用的了，同时保证返回值能够在调用者和被包装方法直接拷贝。

（注意：在汇编输出中，这些包装方法被标记为`<autogenerated>`）

下面是生成的包装器的一个注释列表，应该希望稍微清楚一点：

````assembly
0x0000 TEXT	"".(*Adder).AddVal(SB), DUPOK|WRAPPER, $32-24
  ;; ...omitted preambles...

  0x0026 MOVQ	""..this+40(SP), AX ;; check whether the receiver..
  0x002b TESTQ	AX, AX		    ;; ..is nil
  0x002e JEQ	92		    ;; if it is, jump to 0x005c (panic)

  0x0030 MOVL	(AX), AX            ;; dereference pointer receiver..
  0x0032 MOVL	AX, (SP)            ;; ..and move (i.e. copy) the resulting value to argument #1

  ;; forward (copy) arguments #2 & #3 then call the wrappee
  0x0035 MOVL	"".a+48(SP), AX
  0x0039 MOVL	AX, 4(SP)
  0x003d MOVL	"".b+52(SP), AX
  0x0041 MOVL	AX, 8(SP)
  0x0045 CALL	"".Adder.AddVal(SB) ;; call the wrapped method

  ;; copy return value from wrapped method then return
  0x004a MOVL	16(SP), AX
  0x004e MOVL	AX, "".~r2+56(SP)
  ;; ...omitted frame-pointer stuff...
  0x005b RET

  ;; throw a panic with a detailed error
  0x005c CALL	runtime.panicwrap(SB)

  ;; ...omitted epilogues...
````
显然，