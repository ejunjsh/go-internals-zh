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
- [接口解剖](#%E6%8E%A5%E5%8F%A3%E8%A7%A3%E5%89%96)
  - [数据结构概述](#%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E6%A6%82%E8%BF%B0)
    - [`iface`结构](#iface%E7%BB%93%E6%9E%84)
    - [`itab`结构](#itab%E7%BB%93%E6%9E%84)
    - [`_type`结构](#_type%E7%BB%93%E6%9E%84)
    - [`interfacetype`结构](#interfacetype%E7%BB%93%E6%9E%84)
    - [结论](#%E7%BB%93%E8%AE%BA)
  - [创建一个接口](#%E5%88%9B%E5%BB%BA%E4%B8%80%E4%B8%AA%E6%8E%A5%E5%8F%A3)
    - [部分1:分配接收器](#%E9%83%A8%E5%88%861%E5%88%86%E9%85%8D%E6%8E%A5%E6%94%B6%E5%99%A8)
    - [部分2:组装itab](#%E9%83%A8%E5%88%862%E7%BB%84%E8%A3%85itab)
    - [部分3:设置数据](#%E9%83%A8%E5%88%863%E8%AE%BE%E7%BD%AE%E6%95%B0%E6%8D%AE)
  - [从可执行文件中重建`itab`](#%E4%BB%8E%E5%8F%AF%E6%89%A7%E8%A1%8C%E6%96%87%E4%BB%B6%E4%B8%AD%E9%87%8D%E5%BB%BAitab)
    - [第1步：查找`.rodata`](#%E7%AC%AC1%E6%AD%A5%E6%9F%A5%E6%89%BErodata)
    - [第二步：找到`.rodata`的虚拟内存地址(VMA)](#%E7%AC%AC%E4%BA%8C%E6%AD%A5%E6%89%BE%E5%88%B0rodata%E7%9A%84%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98%E5%9C%B0%E5%9D%80vma)
    - [第三步：查找`go.itab.main.Adder,main.Mather`的VMA和大小](#%E7%AC%AC%E4%B8%89%E6%AD%A5%E6%9F%A5%E6%89%BEgoitabmainaddermainmather%E7%9A%84vma%E5%92%8C%E5%A4%A7%E5%B0%8F)
    - [第四步：查找和抽取`go.itab.main.Adder,main.Mather`](#%E7%AC%AC%E5%9B%9B%E6%AD%A5%E6%9F%A5%E6%89%BE%E5%92%8C%E6%8A%BD%E5%8F%96goitabmainaddermainmather)
    - [小结](#%E5%B0%8F%E7%BB%93)
    - [奖励](#%E5%A5%96%E5%8A%B1)
- [动态分派](#%E5%8A%A8%E6%80%81%E5%88%86%E6%B4%BE)
  - [接口上的间接方法调用](#%E6%8E%A5%E5%8F%A3%E4%B8%8A%E7%9A%84%E9%97%B4%E6%8E%A5%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8)
  - [性能开销](#%E6%80%A7%E8%83%BD%E5%BC%80%E9%94%80)
    - [理论：快速浏览一下现代CPU](#%E7%90%86%E8%AE%BA%E5%BF%AB%E9%80%9F%E6%B5%8F%E8%A7%88%E4%B8%80%E4%B8%8B%E7%8E%B0%E4%BB%A3cpu)
    - [实践：基准测试](#%E5%AE%9E%E8%B7%B5%E5%9F%BA%E5%87%86%E6%B5%8B%E8%AF%95)
      - [基准测试A：一个实例，很多调用，内联和非内联](#%E5%9F%BA%E5%87%86%E6%B5%8B%E8%AF%95a%E4%B8%80%E4%B8%AA%E5%AE%9E%E4%BE%8B%E5%BE%88%E5%A4%9A%E8%B0%83%E7%94%A8%E5%86%85%E8%81%94%E5%92%8C%E9%9D%9E%E5%86%85%E8%81%94)
      - [基准测试B：许多实例，许多非内联调用，小/大/伪随机迭代](#%E5%9F%BA%E5%87%86%E6%B5%8B%E8%AF%95b%E8%AE%B8%E5%A4%9A%E5%AE%9E%E4%BE%8B%E8%AE%B8%E5%A4%9A%E9%9D%9E%E5%86%85%E8%81%94%E8%B0%83%E7%94%A8%E5%B0%8F%E5%A4%A7%E4%BC%AA%E9%9A%8F%E6%9C%BA%E8%BF%AD%E4%BB%A3)
      - [结论](#%E7%BB%93%E8%AE%BA-1)
- [特殊情况和编译器技巧](#%E7%89%B9%E6%AE%8A%E6%83%85%E5%86%B5%E5%92%8C%E7%BC%96%E8%AF%91%E5%99%A8%E6%8A%80%E5%B7%A7)
  - [空接口](#%E7%A9%BA%E6%8E%A5%E5%8F%A3)
  - [持有标量类型的接口](#%E6%8C%81%E6%9C%89%E6%A0%87%E9%87%8F%E7%B1%BB%E5%9E%8B%E7%9A%84%E6%8E%A5%E5%8F%A3)
    - [第一步：创建接口](#%E7%AC%AC%E4%B8%80%E6%AD%A5%E5%88%9B%E5%BB%BA%E6%8E%A5%E5%8F%A3)
    - [赋值（部分1）](#%E8%B5%8B%E5%80%BC%E9%83%A8%E5%88%861)
    - [赋值（部分2）或者要求垃圾回收帮我们赋值](#%E8%B5%8B%E5%80%BC%E9%83%A8%E5%88%862%E6%88%96%E8%80%85%E8%A6%81%E6%B1%82%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E5%B8%AE%E6%88%91%E4%BB%AC%E8%B5%8B%E5%80%BC)
    - [结论](#%E7%BB%93%E8%AE%BA-2)
    - [接口技巧1:字节大小的值](#%E6%8E%A5%E5%8F%A3%E6%8A%80%E5%B7%A71%E5%AD%97%E8%8A%82%E5%A4%A7%E5%B0%8F%E7%9A%84%E5%80%BC)
    - [接口技巧2:静态推断](#%E6%8E%A5%E5%8F%A3%E6%8A%80%E5%B7%A72%E9%9D%99%E6%80%81%E6%8E%A8%E6%96%AD)
    - [接口技巧3：零值](#%E6%8E%A5%E5%8F%A3%E6%8A%80%E5%B7%A73%E9%9B%B6%E5%80%BC)
  - [关于零值变量](#%E5%85%B3%E4%BA%8E%E9%9B%B6%E5%80%BC%E5%8F%98%E9%87%8F)
  - [关于零大小变量](#%E5%85%B3%E4%BA%8E%E9%9B%B6%E5%A4%A7%E5%B0%8F%E5%8F%98%E9%87%8F)
- [接口组合](#%E6%8E%A5%E5%8F%A3%E7%BB%84%E5%90%88)
- [断言](#%E6%96%AD%E8%A8%80)
  - [类型断言](#%E7%B1%BB%E5%9E%8B%E6%96%AD%E8%A8%80)
    - [性能如何？](#%E6%80%A7%E8%83%BD%E5%A6%82%E4%BD%95)
  - [类型switch(Type-switches)](#%E7%B1%BB%E5%9E%8Bswitchtype-switches)
    - [注1：布局](#%E6%B3%A81%E5%B8%83%E5%B1%80)
    - [注2：O（n）](#%E6%B3%A82on)
    - [注3：类型哈希&指针比较](#%E6%B3%A83%E7%B1%BB%E5%9E%8B%E5%93%88%E5%B8%8C%E6%8C%87%E9%92%88%E6%AF%94%E8%BE%83)
- [结论](#%E7%BB%93%E8%AE%BA-3)
- [链接](#%E9%93%BE%E6%8E%A5)

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
考虑下面的代码([direct_calls.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/direct_calls.go))
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

> 方法的直接调用： 为了能使用相同的生成代码处理对函数值的间接调用以及直接调用，针对方法生成的代码 (无论接收器是值还是指针)都使用了与顶层方法同样的调用约定，并把接收器作为第一个参数。

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

显然，考虑到需要做的所有拷贝以便来回传递参数，这种包装会引起相当多的开销; 特别是如果被包装的方法（wrappee）只是几个指令。幸运的是，实际上，编译器会直接将wrappee内联到包装器中，以分摊这些成本（至少在可行时）。

请注意`WRAPPER`符号定义中的指令,它指示此方法不应出现在回溯信息里面(这会混淆终端用户)，也不能从wrappee抛出的异常恢复(panics).

> WRAPPER：这是一个包装函数，不应该算作禁用恢复。

这个`runtime.panicwrap`函数，会在接收器是`nil`的时候抛出异常，而且不解自明的 这是完整的清单供参考（[src/runtime/error.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/error.go#L132-L157)）：

````go
// panicwrap generates a panic for a call to a wrapped value method
// with a nil pointer receiver.
//
// It is called from the generated wrapper code.
func panicwrap() {
    pc := getcallerpc()
    name := funcname(findfunc(pc))
    // name is something like "main.(*T).F".
    // We want to extract pkg ("main"), typ ("T"), and meth ("F").
    // Do it by finding the parens.
    i := stringsIndexByte(name, '(')
    if i < 0 {
        throw("panicwrap: no ( in " + name)
    }
    pkg := name[:i-1]
    if i+2 >= len(name) || name[i-1:i+2] != ".(*" {
        throw("panicwrap: unexpected string after package name: " + name)
    }
    name = name[i+2:]
    i = stringsIndexByte(name, ')')
    if i < 0 {
        throw("panicwrap: no ) in " + name)
    }
    if i+2 >= len(name) || name[i:i+2] != ")." {
        throw("panicwrap: unexpected string after type name: " + name)
    }
    typ := name[:i]
    meth := name[i+2:]
    panic(plainError("value method " + pkg + "." + typ + "." + meth + " called using nil *" + typ + " pointer"))
}
````

这就是函数和方法调用，我们现在将重点放在主要课程：接口。


## 接口解剖

### 数据结构概述

在我们理解它们如何工作之前，我们首先需要建立一个构成接口的数据结构的心智模型，以及它们如何在内存中进行布局。为此，我们将快速浏览运行时包，从Go实现的角度看看接口是怎么实现都。

#### `iface`结构

`iface`是表示运行时内的接口的根类型（[src/runtime/runtime2.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/runtime2.go#L143-L146)）。

其定义如下所示：

````go
type iface struct { // 16 bytes on a 64bit arch
    tab  *itab
    data unsafe.Pointer
}
````

因此一个接口是一个非常简单的结构，它保持2个指针：

* `tab`保存了`itab`对象的地址，这个地址指向的地方包含了接口类型和这个接口类型所指向的数据类型
* `data`是一个原始指针（`unsafe`）,它指向接口引用的值

虽然非常简单，但这个定义已经为我们提供了一些有价值的信息：因为接口只能包含指针，所以我们包装到接口中的任何具体值都必须具有地址。

多数情况下，这会导致堆分配，因为编译器采用保守的方法并强制接收方逃逸。

即使对于标量类型也是如此！

我们用几行代码证明（[https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/escape.go](escape.go)）

````go
type Addifier interface{ Add(a, b int32) int32 }

type Adder struct{ name string }
//go:noinline
func (adder Adder) Add(a, b int32) int32 { return a + b }

func main() {
    adder := Adder{name: "myAdder"}
    adder.Add(10, 32)	      // doesn't escape
    Addifier(adder).Add(10, 32) // escapes
}
````

````shell
$ GOOS=linux GOARCH=amd64 go tool compile -m escape.go
escape.go:13:10: Addifier(adder) escapes to heap
# ...
````

用一个简单benchmark（[escape_test.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/escape_test.go)）可视化这个堆分配

````go
func BenchmarkDirect(b *testing.B) {
    adder := Adder{id: 6754}
    for i := 0; i < b.N; i++ {
        adder.Add(10, 32)
    }
}

func BenchmarkInterface(b *testing.B) {
    adder := Adder{id: 6754}
    for i := 0; i < b.N; i++ {
        Addifier(adder).Add(10, 32)
    }
}
````

````go
$ GOOS=linux GOARCH=amd64 go tool compile -m escape_test.go 
# ...
escape_test.go:22:11: Addifier(adder) escapes to heap
# ...
````

````shell
$ GOOS=linux GOARCH=amd64 go test -bench=. -benchmem ./escape_test.go
BenchmarkDirect-8      	2000000000	         1.60 ns/op	       0 B/op	       0 allocs/op
BenchmarkInterface-8   	100000000	         15.0 ns/op	       4 B/op	       1 allocs/op
````

我们可以清楚地看到，每次我们创建一个新`Addifier`接口并用我们的`adder`变量对它进行初始化时，实际上会发生大小为`sizeof(Adder)`的堆分配。在本章的后面，我们将看到即使简单标量类型在与接口一起使用时也会导致堆分配。

让我们把注意力转向下一个数据结构：`itab` 。

#### `itab`结构

`itab`定义([src/runtime/runtime2.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/runtime2.go#L648-L658))

````go
type itab struct { // 40 bytes on a 64bit arch
    inter *interfacetype
    _type *_type
    hash  uint32 // copy of _type.hash. Used for type switches.
    _     [4]byte
    fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
````

一个`itab`是一个接口的心脏和大脑。

首先，它嵌入一个`_type`，它是运行时中任何Go类型的内部表示。

`_type`描述了一个类型的每个方面：它的名字，它的特征（例如大小，对齐...），甚至在某种程度上，甚至它的表现（例如比较，散列...）！

在这个情况，`_type`字段描述了接口值的类型，例如，`data`指针指向的就是这种类型的指针。

其次，我们发现`interfacetype`指针，它仅仅是`_type`的包装，而且带了些关于接口的额外信息。

正如您所期望的那样，该`inter`字段描述了接口本身的类型。

最后，`fun`数组保存组成接口的虚拟/调度表的函数指针。

注意注释`//variable sized`,这意味这个数组被声明的大小是不相关的。

我们将在本章的后面看到，编译器负责分配支持这个数组的内存，并且独立于这里指定的大小。同样，运行时总是使用原始指针访问此数组，因此边界检查不适用于此。


#### `_type`结构
正如我们上面所说，`_type`结构给出了Go类型的完整描述。

它的定义如下([src/runtime/type.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/type.go#L25-L43))

````go
type _type struct { // 48 bytes on a 64bit arch
    size       uintptr
    ptrdata    uintptr // size of memory prefix holding all pointers
    hash       uint32
    tflag      tflag
    align      uint8
    fieldalign uint8
    kind       uint8
    alg        *typeAlg
    // gcdata stores the GC type data for the garbage collector.
    // If the KindGCProg bit is set in kind, gcdata is a GC program.
    // Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
    gcdata    *byte
    str       nameOff
    ptrToThis typeOff
}
````

值得庆幸的是，这些领域大部分都是不言自明的。

`nameOff`&`typeOff`类型是`int32`，表示在元数据的位移量，而这元数据最后会由链接器嵌入到可执行的程序里。

元数据在运行期被加载进`runtime.moduledata`结构（[src/runtime/symtab.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/symtab.go#L352-L393)）,这个结构跟ELF文件的内容是类似的。

运行期提供了帮助类，这个帮助类通过`moduledata`结构来为下面的偏移量实现必要的逻辑。例如`resolveNameOff`[src/runtime/type.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/type.go#L168-L196)和`resolveTypeOff`[src/runtime/type.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/type.go#L202-L236)

````go
func resolveNameOff(ptrInModule unsafe.Pointer, off nameOff) name {}
func resolveTypeOff(ptrInModule unsafe.Pointer, off typeOff) *_type {}
````

例如，假设`t`是`_type`类型，调用`resolveTypeOff(t, t.ptrToThis)` ,返回`t`的一个副本。

#### `interfacetype`结构

最后，`interfacetype`结构([src/runtime/type.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/type.go#L342-L346)) :

````go
type interfacetype struct { // 80 bytes on a 64bit arch
    typ     _type
    pkgpath name
    mhdr    []imethod
}

type imethod struct {
    name nameOff
    ityp typeOff
}
````

之前说过，`interfacetype`只是一个`_type`类的包装，它添加了些额外的关于接口元数据的信息。

在当前的实现中，元数据大多数由一些偏移量组成，这些偏移量指向方法的名字(names)和类型(types)，而这些方法由接口暴露出来(`[]imethod`).

#### 结论

下面概述了`iface`,把它的子类型都内联起来，这样能帮你连接所有点：


````go
type iface struct { // `iface`
    tab *struct { // `itab`
        inter *struct { // `interfacetype`
            typ struct { // `_type`
                size       uintptr
                ptrdata    uintptr
                hash       uint32
                tflag      tflag
                align      uint8
                fieldalign uint8
                kind       uint8
                alg        *typeAlg
                gcdata     *byte
                str        nameOff
                ptrToThis  typeOff
            }
            pkgpath name
            mhdr    []struct { // `imethod`
                name nameOff
                ityp typeOff
            }
        }
        _type *struct { // `_type`
            size       uintptr
            ptrdata    uintptr
            hash       uint32
            tflag      tflag
            align      uint8
            fieldalign uint8
            kind       uint8
            alg        *typeAlg
            gcdata     *byte
            str        nameOff
            ptrToThis  typeOff
        }
        hash uint32
        _    [4]byte
        fun  [1]uintptr
    }
    data unsafe.Pointer
}
````

这个部分描述了各种不同的数据类型，它们组合起来就是一个接口的结构，这样能帮助我们初步理解了结构之间是怎么互相工作的，就好像机器里面的齿轮一样。

在下一个部分，我们将会学到这些数据结构是怎么运作的。


### 创建一个接口

现在我们快速过一下所有上面提及的数据结构，我们将把注意力放在它们怎么分配内存和初始化上。

考虑下下面这个程序([iface.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/iface.go)):

````go
type Mather interface {
    Add(a, b int32) int32
    Sub(a, b int64) int64
}

type Adder struct{ id int32 }
//go:noinline
func (adder Adder) Add(a, b int32) int32 { return a + b }
//go:noinline
func (adder Adder) Sub(a, b int64) int64 { return a - b }

func main() {
    m := Mather(Adder{id: 6754})

    // This call just makes sure that the interface is actually used.
    // Without this call, the linker would see that the interface defined above
    // is in fact never used, and thus would optimize it out of the final
    // executable.
    m.Add(10, 32)
}
````

注意： 对于本章的其余部分，我们将一个接口`I`和它持有的一个类型`T`表示为`<I,T>`。

例如`Mather(Adder{id: 6754})`实例化一个`iface<Mather, Adder>`。

让我们看看`iface<Mather, Adder>`的实例化过程：

````go
m := Mather(Adder{id: 6754})
````

这一行Go代码实际上引发了很多操作，因为编译器生成的汇编列表可以证明：

````assembly
;; part 1: allocate the receiver
0x001d MOVL	$6754, ""..autotmp_1+36(SP)
;; part 2: set up the itab
0x0025 LEAQ	go.itab."".Adder,"".Mather(SB), AX
0x002c MOVQ	AX, (SP)
;; part 3: set up the data
0x0030 LEAQ	""..autotmp_1+36(SP), AX
0x0035 MOVQ	AX, 8(SP)
0x003a CALL	runtime.convT2I32(SB)
0x003f MOVQ	16(SP), AX
0x0044 MOVQ	24(SP), CX
````

正如你看到的，我们分三个部分来讲解上面的汇编。

#### 部分1:分配接收器

````assembly
0x001d MOVL	$6754, ""..autotmp_1+36(SP)
````

一个常量十进制值`6754`，对应的是`Adder`的ID,存储在当前栈帧的开始位置。

它存储在那是为了编译器之后会用它的地址来引用它；我们在部分3再回来看看。


#### 部分2:组装itab

````go
0x0025 LEAQ	go.itab."".Adder,"".Mather(SB), AX
0x002c MOVQ	AX, (SP)
````

它看上去像编译器已经创建了一个`itab `来表示`iface<Mather, Adder>`接口，并且我们可以用一个全局符号`go.itab."".Adder,"".Mather`来让它可用。

我们在创建`iface<Mather, Adder>`接口的过程中，为了能做到这点，我们加载全局符号`go.itab."".Adder,"".Mather`的有效地址到当前栈桢的顶部。

再一次，我们在部分3能知道为什么。

语法上，能够用以下伪代码来表示:
````go
tab := getSymAddr(`go.itab.main.Adder,main.Mather`).(*itab)
````
这里是我们接口的一半了(译者注：不知道怎么翻译)

现在，我们都走到这里了，让我们深入的看看`go.itab."".Adder,"".Mather`符号。

像往常一样，编译器的`-S`标志能告诉我们一些东西：
````shell
$ GOOS=linux GOARCH=amd64 go tool compile -S iface.go | grep -A 7 '^go.itab."".Adder,"".Mather'
go.itab."".Adder,"".Mather SRODATA dupok size=40
    0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
    0x0010 8a 3d 5f 61 00 00 00 00 00 00 00 00 00 00 00 00  .=_a............
    0x0020 00 00 00 00 00 00 00 00                          ........
    rel 0+8 t=1 type."".Mather+0
    rel 8+8 t=1 type."".Adder+0
    rel 24+8 t=1 "".(*Adder).Add+0
    rel 32+8 t=1 "".(*Adder).Sub+0
````
我们一部分一部分的分析上面的输出。

第一部分声明符号及其属性：

````assembly
go.itab."".Adder,"".Mather SRODATA dupok size=40
````

像往常一样，由于我们直接关注由编译器生成的中间目标文件（即链接程序尚未运行），因此符号名称仍然缺少包名称。这方面没有新东西。

除此之外，我们在这里得到的是一个40字节的全局对象符号，它将被存储在`.rodata`我们的二进制文件的部分中。

注意这个`dupok`指令，告诉链接器这个符号在链接时多次出现是合法的：链接器将不得不随意选择其中的一个。

是什么让Go作者认为这个符号可能会重复，我不确定。随时提出问题，如果你知道更多。

_[更新：我们在这个issue中继续讨论这个问题[ issue #7: How you can get duplicated go.itab interface definitions.](https://github.com/teh-cmc/go-internals/issues/7)]_

第部分是与符号相关的40字节数据的十六进制转储。也就是说，这是一个`itab`结构的序列化表示：

````
0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
0x0010 8a 3d 5f 61 00 00 00 00 00 00 00 00 00 00 00 00  .=_a............
0x0020 00 00 00 00 00 00 00 00                          ........
````

正如你所看到的，在这一点上，大部分数据只是一堆零。链接器会处理它们，正如我们稍后会看到的。

请注意，在所有这些零中，实际上已经设置了4个字节，但是在偏移量处`0x10+4`。

如果我们回顾一下itab结构的声明并注释其字段的相应偏移量：

````go
type itab struct { // 40 bytes on a 64bit arch
    inter *interfacetype // offset 0x00 ($00)
    _type *_type	 // offset 0x08 ($08)
    hash  uint32	 // offset 0x10 ($16)
    _     [4]byte	 // offset 0x14 ($20)
    fun   [1]uintptr	 // offset 0x18 ($24)
			 // offset 0x20 ($32)
}
````

我们看到偏移量`0x10+4`与该`hash uint32`字段匹配：即，对应于我们`main.Adder`类型的哈希值已经在我们的目标文件中。

第三部分也是最后一部分列出了链接器的一系列重定位指令：

````
rel 0+8 t=1 type."".Mather+0
rel 8+8 t=1 type."".Adder+0
rel 24+8 t=1 "".(*Adder).Add+0
rel 32+8 t=1 "".(*Adder).Sub+0
````

`rel 0+8 t=1 type."".Mather+0`告诉链接器用全局符号`type."".Mather`的地址来填充目标目标文件的头8个字节(`0+8`)。

`rel 8+8 t=1 type."".Adder+0`告诉链接器用全局符号`type."".Adder`的地址来填充目标目标文件的下一个8个字节，等等等等。

一旦链接器完成工作并遵循所有这些指令，我们的40字节序列化`itab`将完成。

总的来说，我们现在正在寻找类似于以下伪代码的东西：

````go
tab := getSymAddr(`go.itab.main.Adder,main.Mather`).(*itab)

// NOTE: The linker strips the `type.` prefix from these symbols when building
// the executable, so the final symbol names in the .rodata section of the
// binary will actually be `main.Mather` and `main.Adder` rather than
// `type.main.Mather` and `type.main.Adder`.
// Don't get tripped up by this when toying around with objdump.
tab.inter = getSymAddr(`type.main.Mather`).(*interfacetype)
tab._type = getSymAddr(`type.main.Adder`).(*_type)

tab.fun[0] = getSymAddr(`main.(*Adder).Add`).(uintptr)
tab.fun[1] = getSymAddr(`main.(*Adder).Sub`).(uintptr)
````

我们已经准备好了`itab`，现在如果我们只需要一些数据，就可以创建一个完美的接口。

#### 部分3:设置数据

````assembly
0x0030 LEAQ	""..autotmp_1+36(SP), AX
0x0035 MOVQ	AX, 8(SP)
0x003a CALL	runtime.convT2I32(SB)
0x003f MOVQ	16(SP), AX
0x0044 MOVQ	24(SP), CX
````

从部分1里说过，栈顶(SP)当前保存了`go.itab."".Adder,"".Mather`地址(作为参数#1)。

从部分2里面说过，我们保存`$6754`十进制常量到`""..autotmp_1+36(SP)`:现在我们加载这个常量的地址到栈顶的下一个字节8(SP)(作为参数#2)。

这两个指针作为两个参数传递到`runtime.convT2I32`,这个函数是创建和返回一个完美接口的最后一步了。

让我们更靠近的看看这个函数吧([src/runtime/iface.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/iface.go#L433-L451))：

````go
func convT2I32(tab *itab, elem unsafe.Pointer) (i iface) {
    t := tab._type
    /* ...omitted debug stuff... */
    var x unsafe.Pointer
    if *(*uint32)(elem) == 0 {
        x = unsafe.Pointer(&zeroVal[0])
    } else {
        x = mallocgc(4, t, false)
        *(*uint32)(x) = *(*uint32)(elem)
    }
    i.tab = tab
    i.data = x
    return
}
````

`runtime.convT2I32`做了四件事：

1. 它创建了一个新的`iface`结构`i`（严格来说，是它的调用者创建它的，其实没什么不同）。
2. 把`itab`指针赋值给`i.tab`。
3. 它在堆中分配了一个类型为`i.tab._type`的对象，然后拷贝第二个参数所指向的值给这个对象。
4. 它返回最终的接口。

整个过程非常简单，尽管第三步在这个特定情况下涉及到一些棘手的实现细节，这是由于我们的`Adder`类型实际上是一种标量类型。

我们将在关于[接口的特殊情况]()的章节中更详细地讨论标量类型和接口的交互。

从概念上讲，我们现在完成了以下（伪代码）：

````go
tab := getSymAddr(`go.itab.main.Adder,main.Mather`).(*itab)
elem := getSymAddr(`""..autotmp_1+36(SP)`).(*int32)

i := runtime.convTI32(tab, unsafe.Pointer(elem))

assert(i.tab == tab)
assert(*(*int32)(i.data) == 6754) // same value..
assert((*int32)(i.data) != elem)  // ..but different (al)locations!
````

为了总结所有这些，下面是所有3个部分的汇编代码的完整注释版本：
````assembly
0x001d MOVL	$6754, ""..autotmp_1+36(SP)         ;; create an addressable $6754 value at 36(SP)
0x0025 LEAQ	go.itab."".Adder,"".Mather(SB), AX  ;; set up go.itab."".Adder,"".Mather..
0x002c MOVQ	AX, (SP)                            ;; ..as first argument (tab *itab)
0x0030 LEAQ	""..autotmp_1+36(SP), AX            ;; set up &36(SP)..
0x0035 MOVQ	AX, 8(SP)                           ;; ..as second argument (elem unsafe.Pointer)
0x003a CALL	runtime.convT2I32(SB)               ;; call convT2I32(go.itab."".Adder,"".Mather, &$6754)
0x003f MOVQ	16(SP), AX                          ;; AX now holds i.tab (go.itab."".Adder,"".Mather)
0x0044 MOVQ	24(SP), CX                          ;; CX now holds i.data (&$6754, somewhere on the heap)
````

请记住，所有这一切都始于一行：`m := Mather(Adder{id: 6754})`。

我们最后会得到一个完整的，工作的接口。

### 从可执行文件中重建`itab`

在之前的章节中，我们直接从编译器生成的目标文件里面拿到`go.itab."".Adder,"".Mather`的内容，发现除了`hash`值之外，几乎全是零的:

````
$ GOOS=linux GOARCH=amd64 go tool compile -S iface.go | grep -A 3 '^go.itab."".Adder,"".Mather'
go.itab."".Adder,"".Mather SRODATA dupok size=40
    0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
    0x0010 8a 3d 5f 61 00 00 00 00 00 00 00 00 00 00 00 00  .=_a............
    0x0020 00 00 00 00 00 00 00 00                          ........
````

为了更好地了解数据如何布置到链接器生成的最终可执行文件中，我们将遍历生成的ELF文件并手动重构组成`itab`的字节

希望这能让我们观察到在链接器链接完之后`itab`变成什么样了。

首先，我们来构建`iface`二进制文件：`GOOS=linux GOARCH=amd64 go build -o iface.bin iface.go`。

#### 第1步：查找`.rodata`

让我们打印搜索节标题`.rodata`，`readelf`可以帮助：

````shell
$ readelf -St -W iface.bin
There are 22 section headers, starting at offset 0x190:

Section Headers:
  [Nr] Name
       Type            Address          Off    Size   ES   Lk Inf Al
       Flags
  [ 0] 
       NULL            0000000000000000 000000 000000 00   0   0  0
       [0000000000000000]: 
  [ 1] .text
       PROGBITS        0000000000401000 001000 04b3cf 00   0   0 16
       [0000000000000006]: ALLOC, EXEC
  [ 2] .rodata
       PROGBITS        000000000044d000 04d000 028ac4 00   0   0 32
       [0000000000000002]: ALLOC
## ...omitted rest of output...
````

我们真正需要的是该部分的（十进制）偏移量，所以让我们应用一下pipe-foo：
````shell
$ readelf -St -W iface.bin | \
  grep -A 1 .rodata | \
  tail -n +2 | \
  awk '{print "ibase=16;"toupper($3)}' | \
  bc
315392
````

意思是`.rodata`区域在二进制文件的位置是在315392字节处。

现在我们需要做的是映射这个文件位置到一个虚拟内存地址。

#### 第二步：找到`.rodata`的虚拟内存地址(VMA)

VMA是虚拟地址，一旦二进制文件被OS加载到内存中，该部分将被映射到该虚拟地址。也就是说，这是我们将在运行时用来引用符号的地址。

为什么我们会关心VMA呢，因为我们不能通过`readelf`和`objdump`来拿到指定符号的偏移量。我们能做的是，从另外角度来说，就是获取指定符号的VMA。

加一些简单都数学运算，我们就能够建立VMA和偏移量之间的映射，最终找到我们需要的符号的编译量。

找到`.rodata`的VMA跟找它的偏移量没什么不同，因为它们在不同的列上而已：

````shell
$ readelf -St -W iface.bin | \
  grep -A 1 .rodata | \
  tail -n +2 | \
  awk '{print "ibase=16;"toupper($2)}' | \
  bc
4509696
````

至今为止，`.rodata`在ELF文件中的偏移量为`$315392`(=`0x04d000`)，在运行时映射的虚拟地址为`$4509696`(=`0x44d000`)

现在我们需要我们找的那个符号的VMA和大小：
* VMA（间接地）使我们能够定位这个符号在可执行文件的位置。
* 它的大小将会告诉我们多少数据是我们需要抽取出来的，一旦我们知道正确的偏移量。


#### 第三步：查找`go.itab.main.Adder,main.Mather`的VMA和大小

`objdump`可以帮到我们。

首先，找到那个符号

````shell
$ objdump -t -j .rodata iface.bin | grep "go.itab.main.Adder,main.Mather"
0000000000475140 g     O .rodata	0000000000000028 go.itab.main.Adder,main.Mather
````

然后获取VMA的十进制数:

````shell
$ objdump -t -j .rodata iface.bin | \
  grep "go.itab.main.Adder,main.Mather" | \
  awk '{print "ibase=16;"toupper($1)}' | \
  bc
4673856
````

最后它十进制大小:

````shell
$ objdump -t -j .rodata iface.bin | \
  grep "go.itab.main.Adder,main.Mather" | \
  awk '{print "ibase=16;"toupper($5)}' | \
  bc
40
````

所以`go.itab.main.Adder,main.Mather`运行时映射的虚拟地址是`$4673856`(=`0x475140`),大小是40个字节(这个值跟`itab`的结构大小一致)

#### 第四步：查找和抽取`go.itab.main.Adder,main.Mather`

现在我们集齐所有能够计算`go.itab.main.Adder,main.Mather`在二进制文件位置的元素了。

下面是我们现在知道的所有元素：

````
.rodata offset: 0x04d000 == $315392
.rodata VMA: 0x44d000 == $4509696

go.itab.main.Adder,main.Mather VMA: 0x475140 == $4673856
go.itab.main.Adder,main.Mather size: 0x24 = $40
````

如果`$315392`（`.rodata`的偏移量）映射到`$4509696`（`.rodata`的VMA）和`go.itab.main.Adder,main.Mather`的VMA是`$4673856`,然后我们就能得出`go.itab.main.Adder,main.Mather`在可执行文件的偏移量为:`sym.offset = sym.vma - section.vma + section.offset = $4673856 - $4509696 + $315392 = $479552`。

现在我们知道了偏移量和大小，就可以用`dd`命令从可执行文件里面抽取`go.itab.main.Adder,main.Mather`的原始字节了。

````shell
$ dd if=iface.bin of=/dev/stdout bs=1 count=40 skip=479552 2>/dev/null | hexdump
0000000 bd20 0045 0000 0000 ed40 0045 0000 0000
0000010 3d8a 615f 0000 0000 c2d0 0044 0000 0000
0000020 c350 0044 0000 0000                    
0000028
````

这看起来确实是一个明确的胜利..但是，真的吗？也许我们刚刚dump出来的只是40个完全随机的，无关的字节？谁知道？

这里有个方法能够确认：让我们用运行时的hash值（来自下面代码[iface_type_hash.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/iface_type_hash.go)）和我们在二进制文件中找到hash值（`0x10+4` -> `0x615f3d8a`）进行比较。

````go
// simplified definitions of runtime's iface & itab types
type iface struct {
    tab  *itab
    data unsafe.Pointer
}
type itab struct {
    inter uintptr
    _type uintptr
    hash  uint32
    _     [4]byte
    fun   [1]uintptr
}

func main() {
    m := Mather(Adder{id: 6754})

    iface := (*iface)(unsafe.Pointer(&m))
    fmt.Printf("iface.tab.hash = %#x\n", iface.tab.hash) // 0x615f3d8a
}
````

匹配上了! `fmt.Printf("iface.tab.hash = %#x\n", iface.tab.hash)`得到`0x615f3d8a`，跟我们在ELF文件里面抽出来的值是相对应的。

#### 小结
我们已经为`iface<Mather, Adder>`接口重建出一个完全的`itab`；它都在可执行文件里面，等着被执行，并且提供所有信息给运行时去令这个接口能按我们的意思去执行。

当然，自从`itab`大部分都是由一堆指向其他数据结构的指针组成，我们已经用命令`dd`抽取的虚拟地址来重建这个完整的图像。 

提起指针，我们现在能够有一个`iface<Mather, Adder>`虚拟表(virtual-table)的清晰视图了；下面是`go.itab.main.Adder,main.Mather`的注解版本：

````shell
$ dd if=iface.bin of=/dev/stdout bs=1 count=40 skip=479552 2>/dev/null | hexdump
0000000 bd20 0045 0000 0000 ed40 0045 0000 0000
0000010 3d8a 615f 0000 0000 c2d0 0044 0000 0000
#                           ^^^^^^^^^^^^^^^^^^^
#                       offset 0x18+8: itab.fun[0]
0000020 c350 0044 0000 0000                    
#       ^^^^^^^^^^^^^^^^^^^
# offset 0x20+8: itab.fun[1]
0000028
````

````shell
$ objdump -t -j .text iface.bin | grep 000000000044c2d0
000000000044c2d0 g     F .text	0000000000000079 main.(*Adder).Add
````

````shell
$ objdump -t -j .text iface.bin | grep 000000000044c350
000000000044c350 g     F .text	000000000000007f main.(*Adder).Sub
````

没有什么惊喜的，`iface<Mather, Adder>`的虚拟表保存了两个方法指针`main.(*Adder).add`和`main.(*Adder).sub`。

事实上，还是有点惊喜的：我们从来没有定义两个指针接收器的方法。

编译器已经为我们生成了这些包装器方法（正如我们在“[隐式解引用](#%E9%9A%90%E5%BC%8F%E8%A7%A3%E5%BC%95%E7%94%A8)”一节中所描述的那样），因为它知道我们需要它们：自从一个接口结构里面只含有指针，并且自从我们`Adder`实现上述接口只为方法提供值接收器，如果我们要通过接口的虚拟表调用这些方法中的任何一个，我们必须在某个时候通过一个包装器。

这应该已经给你一个关于如何在运行时处理动态分派的好主意; 这是我们将在下一节中看到的。

#### 奖励

我已经破解了一个通用的bash脚本，您可以使用它来转储ELF文件（[dump_sym.sh](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/dump_sym.sh)）的任何部分中的任何符号的内容：

````shell
# ./dump_sym.sh bin_path section_name sym_name
$ ./dump_sym.sh iface.bin .rodata go.itab.main.Adder,main.Mather
.rodata file-offset: 315392
.rodata VMA: 4509696
go.itab.main.Adder,main.Mather VMA: 4673856
go.itab.main.Adder,main.Mather SIZE: 40

0000000 bd20 0045 0000 0000 ed40 0045 0000 0000
0000010 3d8a 615f 0000 0000 c2d0 0044 0000 0000
0000020 c350 0044 0000 0000                    
0000028
````

我会想象一定有一种更简单的方法来实现这个脚本的功能，也许有一些神秘的标记或者隐秘的宝石(gem)隐藏在`binutils`发行版中的..谁知道。

如果你有一些提示，不要犹豫提个issue吧。


## 动态分派

在本节中，我们将最终介绍接口的主要功能：动态分派。

具体来说，我们将看看动态分派如何在底层工作，以及我们为此付出多少代价。

### 接口上的间接方法调用

让我们回顾一下前面的代码（[iface.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/iface.go)）：

````go
type Mather interface {
    Add(a, b int32) int32
    Sub(a, b int64) int64
}

type Adder struct{ id int32 }
//go:noinline
func (adder Adder) Add(a, b int32) int32 { return a + b }
//go:noinline
func (adder Adder) Sub(a, b int64) int64 { return a - b }

func main() {
    m := Mather(Adder{id: 6754})
    m.Add(10, 32)
}
````


我们已经深入了解了这段代码中发生的大部分事情：`iface<Mather, Adder>`接口是如何创建的，它如何在最终的可执行文件中进行布局，以及它如何最终被运行时加载。

我们只有一件事要看，那就是下面的实际间接方法调用：`m.Add(10, 32)`。

为了更新我们的记忆，我们将放大接口的创建以及方法调用本身：

````go
m := Mather(Adder{id: 6754})
m.Add(10, 32)
````

万幸的是，我们已经有了完整注释版本的汇编代码，这汇编代码的生成是为了实例化上面的第一行代码`m := Mather(Adder{id: 6754})`:

````assembly
;; m := Mather(Adder{id: 6754})
0x001d MOVL	$6754, ""..autotmp_1+36(SP)         ;; create an addressable $6754 value at 36(SP)
0x0025 LEAQ	go.itab."".Adder,"".Mather(SB), AX  ;; set up go.itab."".Adder,"".Mather..
0x002c MOVQ	AX, (SP)                            ;; ..as first argument (tab *itab)
0x0030 LEAQ	""..autotmp_1+36(SP), AX            ;; set up &36(SP)..
0x0035 MOVQ	AX, 8(SP)                           ;; ..as second argument (elem unsafe.Pointer)
0x003a CALL	runtime.convT2I32(SB)               ;; runtime.convT2I32(go.itab."".Adder,"".Mather, &$6754)
0x003f MOVQ	16(SP), AX                          ;; AX now holds i.tab (go.itab."".Adder,"".Mather)
0x0044 MOVQ	24(SP), CX                          ;; CX now holds i.data (&$6754, somewhere on the heap)
````

下面是间接方法调用`m.Add(10, 32)`的汇编代码:

````assembly
;; m.Add(10, 32)
0x0049 MOVQ	24(AX), AX
0x004d MOVQ	$137438953482, DX
0x0057 MOVQ	DX, 8(SP)
0x005c MOVQ	CX, (SP)
0x0060 CALL	AX
````

根据之前章节知识的积累，这些指令应该很好理解了。

````assembly
0x0049 MOVQ	24(AX), AX
````

一旦`runtime.convT2I32`返回，`AX`就保存了`i.tab`的值，`i.tab`就是我们知道的指向`itab`的指针；在这里特指指向`go.itab."".Adder,"".Mather`的指针。

通过解引用`AX`和向前偏移24个字节，我们到达了`i.tab.fun`，这个对应的是虚拟表的第一个方法。

下面是一个带有`itab`的偏移表的代码：
````go
type itab struct { // 32 bytes on a 64bit arch
    inter *interfacetype // offset 0x00 ($00)
    _type *_type	 // offset 0x08 ($08)
    hash  uint32	 // offset 0x10 ($16)
    _     [4]byte	 // offset 0x14 ($20)
    fun   [1]uintptr	 // offset 0x18 ($24)
			 // offset 0x20 ($32)
}
````

正如我们在之前从可执行文件中重建`itab`的章节中看到，`iface.tab.fun[0]`是一个指向`main.(*Adder).add`的指针，它是由编译器生成的包装方法，包装了我们原来的值接收方法`main.Adder.add`。

````assembly
0x004d MOVQ	$137438953482, DX
0x0057 MOVQ	DX, 8(SP)
````

我们保存`10`和`32`到栈顶，作为方法的第二和第三个参数。

````assembly
0x005c MOVQ	CX, (SP)
0x0060 CALL	AX
````

一旦`runtime.convT2I32`返回，`CX`保存了`i.data`的值，它是一个指向`Adder`实例的指针。

我们把这个指针放到栈顶，作为方法的第一个参数，用来满足调用的约定：方法的接收器应该一直被传递作为第一个参数。

最后，栈准备好了，我们可以开始调用了。

我们以一个完整注释的汇编来结束这一节，它很好的解释了整个过程:

````assembly
;; m := Mather(Adder{id: 6754})
0x001d MOVL	$6754, ""..autotmp_1+36(SP)         ;; create an addressable $6754 value at 36(SP)
0x0025 LEAQ	go.itab."".Adder,"".Mather(SB), AX  ;; set up go.itab."".Adder,"".Mather..
0x002c MOVQ	AX, (SP)                            ;; ..as first argument (tab *itab)
0x0030 LEAQ	""..autotmp_1+36(SP), AX            ;; set up &36(SP)..
0x0035 MOVQ	AX, 8(SP)                           ;; ..as second argument (elem unsafe.Pointer)
0x003a CALL	runtime.convT2I32(SB)               ;; runtime.convT2I32(go.itab."".Adder,"".Mather, &$6754)
0x003f MOVQ	16(SP), AX                          ;; AX now holds i.tab (go.itab."".Adder,"".Mather)
0x0044 MOVQ	24(SP), CX                          ;; CX now holds i.data (&$6754, somewhere on the heap)
;; m.Add(10, 32)
0x0049 MOVQ	24(AX), AX                          ;; AX now holds (*iface.tab)+0x18, i.e. iface.tab.fun[0]
0x004d MOVQ	$137438953482, DX                   ;; move (32,10) to..
0x0057 MOVQ	DX, 8(SP)                           ;; ..the top of the stack (arguments #3 & #2)
0x005c MOVQ	CX, (SP)                            ;; CX, which holds &$6754 (i.e., our receiver), gets moved to
                                                    ;; ..the top of stack (argument #1 -> receiver)
0x0060 CALL	AX                                  ;; you know the drill
````

我们现在对于接口的机制还是虚拟方法调用是怎么工作的已经有一个清晰印象了。

在下一节，我们会用理论和实践来衡量这种机制的开销。

### 性能开销

正如我们看到的，接口的实现代表了编译器和链接器的大多数工作。从性能角度来看，这显然是一个非常好的消息：我们实际上想要尽可能减轻运行时间。

初始化一个接口在某种特定情况可能需要占用一些运行时间(例如`runtime.convT2*`这类函数)，尽管它们在实践中没有那么普遍。

我们会在下面的章节学到接口的某种特定情况。在这个章节，我们只会专注于虚拟方法调用的性能开销和忽略相关初始化时的一次性开销。

一旦一个接口被合适地初始化，调用它的方法只不过是比通常的静态分派调用（即`itab.fun`在所需索引处解引用）多一层间接寻址。

因此，人们可以想象这个过程几乎是没有问题的，可能会是对的，但并不完全：理论有点棘手，现实更棘手。

#### 理论：快速浏览一下现代CPU

对于虚拟调用而言，固有的额外间接调用本身只要从CPU的角度来看是有点可预测的，是没问题的。

现代CPU非常具有侵略性：它们积极缓存，积极预取指令和数据，积极预先执行代码，甚至在他们认为合适的时候重新排序和并行化代码。所有这些额外的工作无论是否需要都要完成的，因此我们应该始终努力不妨碍CPU的这些额外聪明工作，这样所有这些宝贵的工作才不会浪费。

这是虚拟方法调用可能很快成为问题的地方。

在静态分派调用情况，CPU能够预测到程序接下来要执行的分支，相应的去获取必要的指令。从性能方面来说，这使得程序能从一个分支到另一个分支的平滑，透明的过渡。

另一方面，在动态分派的情况下，CPU不能预先知道程序接下来往哪走：这一切都取决于计算，根据定义，计算的结果在运行时间之前是未知的。为了平衡这一点，CPU应用各种算法和启发式，以便猜测程序该走哪条分支（即“分支预测”）。

如果处理器猜测正确，我们可以预期动态分支几乎和静态分支一样高效，因为接下来要运行的指令已经被预取到处理器的缓存中。

如果猜测错误，那就有点难了：首先，额外的间接调用和相应的从主存（慢）加载正确的指令到L1i缓存（CPU会暂停的），更严重的是CPU为了回退它自己的错误，刷新它的分支预测错误时的指令流水线。

动态分派的另一个重要缺点是它根据定义使内联变得不可能：不能内联他们不知道的内容。

总而言之，至少在理论上，内联的直接调用和不能内联的同样函数之间已经有很大的性能差距了，现在还要有额外的一层间接调用，甚至可能在分支机构错误预测中受到冲击。

这主要是因为理论。

但是，对于现代硬件来说，人们应该始终对这一理论持谨慎态度。

所以我们衡量一下这个东西吧

#### 实践：基准测试

首先，看一下我们cpu的信息：

````shell
$ lscpu | sed -nr '/Model name/ s/.*:\s*(.* @ .*)/\1/p'
Intel(R) Core(TM) i7-7700HQ CPU @ 2.80GHz
````

我们会定义一个接口[](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/iface_bench_test.go)来作为基准测试的例子：

````go
type identifier interface {
    idInline() int32
    idNoInline() int32
}

type id32 struct{ id int32 }

// NOTE: Use pointer receivers so we don't measure the extra overhead incurred by
// autogenerated wrappers as part of our results.

func (id *id32) idInline() int32 { return id.id }
//go:noinline
func (id *id32) idNoInline() int32 { return id.id }
````

##### 基准测试A：一个实例，很多调用，内联和非内联

对于我们第一个基准测试，我们会分别用`*Adder`值和`iface<Mather, *Adder>`接口在一个忙循环里面调用非内联方法：

````go
var escapeMePlease *id32
// escapeToHeap makes sure that `id` escapes to the heap.
//
// In simple situations such as some of the benchmarks present in this file,
// the compiler is able to statically infer the underlying type of the
// interface (or rather the type of the data it points to, to be pedantic) and
// ends up replacing what should have been a dynamic method call by a
// static call.
// This anti-optimization prevents this extra cleverness.
//
//go:noinline
func escapeToHeap(id *id32) identifier {
    escapeMePlease = id
    return escapeMePlease
}

var myID int32

func BenchmarkMethodCall_direct(b *testing.B) {
    b.Run("single/noinline", func(b *testing.B) {
        m := escapeToHeap(&id32{id: 6754}).(*id32)
        for i := 0; i < b.N; i++ {
            // CALL "".(*id32).idNoInline(SB)
            // MOVL 8(SP), AX
            // MOVQ "".&myID+40(SP), CX
            // MOVL AX, (CX)
            myID = m.idNoInline()
        }
    })
}

func BenchmarkMethodCall_interface(b *testing.B) {
    b.Run("single/noinline", func(b *testing.B) {
        m := escapeToHeap(&id32{id: 6754})
        for i := 0; i < b.N; i++ {
            // MOVQ 32(AX), CX
            // MOVQ "".m.data+40(SP), DX
            // MOVQ DX, (SP)
            // CALL CX
            // MOVL 8(SP), AX
            // MOVQ "".&myID+48(SP), CX
            // MOVL AX, (CX)
            myID = m.idNoInline()
        }
    })
}
````

我们期望上面两个测试，A)非常快，B)速度几乎一样

考虑到循环的紧凑性，我们能够预测到这两个测试它们的数据（接收器和虚拟表）和指令（`"".(*id32).idNoInline`）在每个循环迭代已经缓存到CPU的L1d/L1i缓存中，也就是说，性能应该纯粹是CPU限制的。

`BenchmarkMethodCall_interface`应该运行得慢一点（在纳秒级），因为它必须处理从虚拟表（尽管它已经在L1缓存中）中查找和复制正确指针的开销。

由于`CALL CX`指令很强依赖于额外一些指令，一些需要查询虚拟表的指令的输出，所以处理器没办法只好执行额外的一些逻辑序列流，错过了指令级在表上的并行化机会。

这最后为什么我们会觉得"接口"会慢一点的主要原因。

下面是"直接"版本的结果：

````shell
$ go test -run=NONE -o iface_bench_test.bin iface_bench_test.go && \
  perf stat --cpu=1 \
  taskset 2 \
  ./iface_bench_test.bin -test.cpu=1 -test.benchtime=1s -test.count=3 \
      -test.bench='BenchmarkMethodCall_direct/single/noinline'
BenchmarkMethodCall_direct/single/noinline         	2000000000	         1.81 ns/op
BenchmarkMethodCall_direct/single/noinline         	2000000000	         1.80 ns/op
BenchmarkMethodCall_direct/single/noinline         	2000000000	         1.80 ns/op

 Performance counter stats for 'CPU(s) 1':

      11702.303843      cpu-clock (msec)          #    1.000 CPUs utilized          
             2,481      context-switches          #    0.212 K/sec                  
                 1      cpu-migrations            #    0.000 K/sec                  
             7,349      page-faults               #    0.628 K/sec                  
    43,726,491,825      cycles                    #    3.737 GHz                    
   110,979,100,648      instructions              #    2.54  insn per cycle         
    19,646,440,556      branches                  # 1678.852 M/sec                  
           566,424      branch-misses             #    0.00% of all branches        

      11.702332281 seconds time elapsed
````

和下面是"接口"版本的结果：

````shell
$ go test -run=NONE -o iface_bench_test.bin iface_bench_test.go && \
  perf stat --cpu=1 \
  taskset 2 \
  ./iface_bench_test.bin -test.cpu=1 -test.benchtime=1s -test.count=3 \
      -test.bench='BenchmarkMethodCall_interface/single/noinline'
BenchmarkMethodCall_interface/single/noinline         	2000000000	         1.95 ns/op
BenchmarkMethodCall_interface/single/noinline         	2000000000	         1.96 ns/op
BenchmarkMethodCall_interface/single/noinline         	2000000000	         1.96 ns/op

 Performance counter stats for 'CPU(s) 1':

      12709.383862      cpu-clock (msec)          #    1.000 CPUs utilized          
             3,003      context-switches          #    0.236 K/sec                  
                 1      cpu-migrations            #    0.000 K/sec                  
            10,524      page-faults               #    0.828 K/sec                  
    47,301,533,147      cycles                    #    3.722 GHz                    
   124,467,105,161      instructions              #    2.63  insn per cycle         
    19,878,711,448      branches                  # 1564.097 M/sec                  
           761,899      branch-misses             #    0.00% of all branches        

      12.709412950 seconds time elapsed
````

这结果跟我们预期的一致："接口"版本确实慢了一点，每个迭代接近0.15的额外纳秒开销，或者性能下降了~8%

8%可能听起来像是一个明显的差异，但我们必须记住，A）这些是纳秒级的测量 B）被调用的方法做得很少，它放大了调用的开销。

看看每个基准测试的指令数量，我们发现基于接口的版本不得不执行与“直接”版本（`110,979,100,648`vs. `124,467,105,161`）相比更多的140亿条指令，尽管这两个基准测试都是针对`6,000,000,000`（`2,000,000,000\*3`）迭代运行的。

正如我们之前提到的，CPU不能并行化这些额外的指令，因为`CALL`依赖这些指令，这很明显的反映在每周期指令率(instruction-per-cycle ratio):这两个测试都有类似的IPC率（`2.54` vs. `2.63`）,尽管"接口"版本总的来说要做更多事。

"接口"版本并行化的缺乏累计起来就是额外大约35亿CPU周期，这应该是我们测量的0.15纳秒花费在的地方。

现在，我们让编译器内联方法调用，看看会发生什么？

````go
var myID int32

func BenchmarkMethodCall_direct(b *testing.B) {
    b.Run("single/inline", func(b *testing.B) {
        m := escapeToHeap(&id32{id: 6754}).(*id32)
        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            // MOVL (DX), SI
            // MOVL SI, (CX)
            myID = m.idInline()
        }
    })
}

func BenchmarkMethodCall_interface(b *testing.B) {
    b.Run("single/inline", func(b *testing.B) {
        m := escapeToHeap(&id32{id: 6754})
        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            // MOVQ 32(AX), CX
            // MOVQ "".m.data+40(SP), DX
            // MOVQ DX, (SP)
            // CALL CX
            // MOVL 8(SP), AX
            // MOVQ "".&myID+48(SP), CX
            // MOVL AX, (CX)
            myID = m.idNoInline()
        }
    })
}
````

两个事情：

* `BenchmarkMethodCall_direct`: 由于内联，调用已经减少到一对简单的内存移动。
* `BenchmarkMethodCall_interface`: 由于动态分派，编译器无法内联该调用，因此生成的程序集与以前完全相同。

由于`BenchmarkMethodCall_interface`代码没有改变，所以没必要看它的测试了

所以让我们看看"直接"版本的吧：

````shell
$ go test -run=NONE -o iface_bench_test.bin iface_bench_test.go && \
  perf stat --cpu=1 \
  taskset 2 \
  ./iface_bench_test.bin -test.cpu=1 -test.benchtime=1s -test.count=3 \
      -test.bench='BenchmarkMethodCall_direct/single/inline'
BenchmarkMethodCall_direct/single/inline         	2000000000	         0.35 ns/op
BenchmarkMethodCall_direct/single/inline         	2000000000	         0.34 ns/op
BenchmarkMethodCall_direct/single/inline         	2000000000	         0.34 ns/op

 Performance counter stats for 'CPU(s) 1':

       2464.353001      cpu-clock (msec)          #    1.000 CPUs utilized          
               629      context-switches          #    0.255 K/sec                  
                 1      cpu-migrations            #    0.000 K/sec                  
             7,322      page-faults               #    0.003 M/sec                  
     9,026,867,915      cycles                    #    3.663 GHz                    
    41,580,825,875      instructions              #    4.61  insn per cycle         
     7,027,066,264      branches                  # 2851.485 M/sec                  
         1,134,955      branch-misses             #    0.02% of all branches        

       2.464386341 seconds time elapsed
````

预料之中，非常快，调用的开销已经没有了。

对于"直接"版本0.34ns/op，"接口"版本现在是慢了约475%，与我们之前在禁用内联功能时测量的约8％的差异相比急剧下降。

注意，由于方法调用的固有分支已经消失，CPU能够更高效地并行化并推测性地执行其余指令，从而使IPC比率达到4.61。


##### 基准测试B：许多实例，许多非内联调用，小/大/伪随机迭代

对于这个第二个基准测试，我们将看到更类似现实世界的情况，其中迭代器遍历一组(slice)对象，这些对象都公开一个公用方法，并为每个对象调用它。

为了更好地模拟现实，我们将禁用内联，因为在真正的程序中调用这种方法的大多数方法很可能非常复杂，不会被编译器内联（YMMV;这是一个很好的反例 - 来自标准库接口`sort.Interface`）。

我们将定义3个类似的测试，它们访问这组对象的方式有所不同; 目标是模拟高速缓存友好性的下降级别：

1. 在第一种情况下，迭代器按顺序遍历数组，调用方法，然后在每次迭代时增加一个对象的大小。
2. 在第二种情况下，迭代器仍按顺序遍历数组，但这次会增加一个大于单个缓存行大小的值。
3. 最后，在第三种情况下，迭代器将伪随机遍历数组。

在这三种情况下，我们都要确保数组足够大，不能完全适合任何处理器的缓存，以便模拟（不太准确）非常繁忙的服务器，这种服务器会给CPU高速缓存和主内存很大压力。

下面简要回顾处理器的属性，我们会根据这个设计基准测试：

````shell
$ lscpu | sed -nr '/Model name/ s/.*:\s*(.* @ .*)/\1/p'
Intel(R) Core(TM) i7-7700HQ CPU @ 2.80GHz
$ lscpu | grep cache
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            6144K
$ getconf LEVEL1_DCACHE_LINESIZE
64
$ getconf LEVEL1_ICACHE_LINESIZE
64
$ find /sys/devices/system/cpu/cpu0/cache/index{1,2,3} -name "shared_cpu_list" -exec cat {} \;
# (annotations are mine)
0,4 # L1 (hyperthreading)
0,4 # L2 (hyperthreading)
0-7 # L3 (shared + hyperthreading)
````

以下是“直接”版本的基准测试套件的代码（基准标记为`baseline`计算单独检索接收器的成本，以便我们可以从最终测量结果中减去该成本）：

````go
const _maxSize = 2097152             // 2^21
const _maxSizeModMask = _maxSize - 1 // avoids a mod (%) in the hot path

var _randIndexes = [_maxSize]int{}
func init() {
    rand.Seed(42)
    for i := range _randIndexes {
        _randIndexes[i] = rand.Intn(_maxSize)
    }
}

func BenchmarkMethodCall_direct(b *testing.B) {
    adders := make([]*id32, _maxSize)
    for i := range adders {
        adders[i] = &id32{id: int32(i)}
    }
    runtime.GC()

    var myID int32

    b.Run("many/noinline/small_incr", func(b *testing.B) {
        var m *id32
        b.Run("baseline", func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                m = adders[i&_maxSizeModMask]
            }
        })
        b.Run("call", func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                m = adders[i&_maxSizeModMask]
                myID = m.idNoInline()
            }
        })
    })
    b.Run("many/noinline/big_incr", func(b *testing.B) {
        var m *id32
        b.Run("baseline", func(b *testing.B) {
            j := 0
            for i := 0; i < b.N; i++ {
                m = adders[j&_maxSizeModMask]
                j += 32
            }
        })
        b.Run("call", func(b *testing.B) {
            j := 0
            for i := 0; i < b.N; i++ {
                m = adders[j&_maxSizeModMask]
                myID = m.idNoInline()
                j += 32
            }
        })
    })
    b.Run("many/noinline/random_incr", func(b *testing.B) {
        var m *id32
        b.Run("baseline", func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                m = adders[_randIndexes[i&_maxSizeModMask]]
            }
        })
        b.Run("call", func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                m = adders[_randIndexes[i&_maxSizeModMask]]
                myID = m.idNoInline()
            }
        })
    })
}
````

“接口”版本的基准测试组是相同的，除了数组是用接口值而不是指向具体类型的指针进行初始化的，正如人们所期望的那样：

````go
func BenchmarkMethodCall_interface(b *testing.B) {
    adders := make([]identifier, _maxSize)
    for i := range adders {
        adders[i] = identifier(&id32{id: int32(i)})
    }
    runtime.GC()

    /* ... */
}
````

下面是"直接"版本的测试结果：

````shell
$ go test -run=NONE -o iface_bench_test.bin iface_bench_test.go && \
  benchstat <(
    taskset 2 ./iface_bench_test.bin -test.cpu=1 -test.benchtime=1s -test.count=3 \
      -test.bench='BenchmarkMethodCall_direct/many/noinline')
name                                                  time/op
MethodCall_direct/many/noinline/small_incr/baseline   0.99ns ± 3%
MethodCall_direct/many/noinline/small_incr/call       2.32ns ± 1% # 2.32 - 0.99 = 1.33ns
MethodCall_direct/many/noinline/big_incr/baseline     5.86ns ± 0%
MethodCall_direct/many/noinline/big_incr/call         17.1ns ± 1% # 17.1 - 5.86 = 11.24ns
MethodCall_direct/many/noinline/random_incr/baseline  8.80ns ± 0%
MethodCall_direct/many/noinline/random_incr/call      30.8ns ± 0% # 30.8 - 8.8 = 22ns
````

这里真的没有什么惊喜：

1. `small_incr`：通过对缓存非常友好，我们获得了与之前在单个实例上循环测试的类似结果。
2. `big_incr`：通过强制CPU在每次迭代时获取新的缓存行，我们确实看到了明显的延迟波动，这与调用的成本完全无关，但是约6ns属于基线，其余的包含了解引用获得`id`字段和拷贝返回值加在一起的成本。
3. `random_incr`：跟`big_incr`差不多，但是比它多了A)伪随机访问 B) 获取预先计算好的索引数组的下一个索引的成本（这会触发其缓存不命中和本身）

正如逻辑所指出的那样，CPU的d-cache的抖动似乎并没有以任何方式影响实际直接方法调用（内联或不内联）的延迟，尽管它确实使围绕它的一切都变得更慢。

动态分派的结果如何呢？

````shell
$ go test -run=NONE -o iface_bench_test.bin iface_bench_test.go && \
  benchstat <(
    taskset 2 ./iface_bench_test.bin -test.cpu=1 -test.benchtime=1s -test.count=3 \
      -test.bench='BenchmarkMethodCall_interface/many/inline')
name                                                     time/op
MethodCall_interface/many/noinline/small_incr/baseline   1.38ns ± 0%
MethodCall_interface/many/noinline/small_incr/call       3.48ns ± 0% # 3.48 - 1.38 = 2.1ns
MethodCall_interface/many/noinline/big_incr/baseline     6.86ns ± 0%
MethodCall_interface/many/noinline/big_incr/call         19.6ns ± 1% # 19.6 - 6.86 = 12.74ns
MethodCall_interface/many/noinline/random_incr/baseline  11.0ns ± 0%
MethodCall_interface/many/noinline/random_incr/call      34.7ns ± 0% # 34.7 - 11.0 = 23.7ns
````

结果非常相似，尽管总体上稍慢一点，这是因为我们`identifier`在每次迭代中将两个四字（即一个接口的两个字段）从数组中复制出来，而不是一个（指向`id32`的指针）。

这个运行的速度几乎与其“直接”对应的速度一样快，因为数组中的所有接口共享一个共同的`itab`（即它们都是`iface<Mather, Adder>`接口），它们相关的虚拟表永远不会离开L1d高速缓存，因此每次迭代提取正确的方法的指针几乎都是0成本的。

同样，组成`main.(*id32).idNoInline`方法主体的指令也不会离开L1i缓存。

有人可能会认为，在实践中，一组接口将包含许多不同的基础类型（和虚拟表），这将导致L1i和L1d高速缓存的抖动，这是由于各种虚拟表互相推挤造成的。

虽然这在理论上是成立的，但这些想法往往是多年来使用旧版OOP语言（如C++）的经验的结果，（以前，至少）鼓励将深度嵌套的继承类和虚拟调用层次用作他们的主要抽象工具。

在足够大的层次结构中，关联虚拟表的数量有时可能足够大，例如迭代一个持有各种虚拟类实现的数据结构的时候，会导致CPU缓存抖动，（可以考虑GUI框架，其中所有内容都是Widget存储在一个图形数据结构中）; 特别是至少在C++中，虚拟类倾向于指定相当复杂的行为，有时候会有十几种方法，从而导致相当大的虚拟表，并且给L1d缓存带来更大的压力。

Go,从另一方面来说，非常不同：OOP已经完全抛弃，类型系统变得平坦起来，并且接口通常被用来描述最小的，受限制的行为（一些方法最多是平均的，接口隐式满足的事实），而不是在更复杂的分层类型层次结构之上用作抽象。

在实践中，用Go来写代码，我发现很少去遍历一组包含许多不同基础类型的接口。YMMV，当然。

对于好奇心强的人来说，以下是启用内联的“直接”版本的结果：

````go
name                                                time/op
MethodCall_direct/many/inline/small_incr            0.97ns ± 1% # 0.97ns
MethodCall_direct/many/inline/big_incr/baseline     5.96ns ± 1%
MethodCall_direct/many/inline/big_incr/call         11.9ns ± 1% # 11.9 - 5.96 = 5.94ns
MethodCall_direct/many/inline/random_incr/baseline  9.20ns ± 1%
MethodCall_direct/many/inline/random_incr/call      16.9ns ± 1% # 16.9 - 9.2 = 7.7ns
````

在编译器能够内联该调用的情况下，这将使“直接”版本比“接口”版本快2至3倍。

再次，正如我们前面提到的，当前编译器在内联方面的有限能力意味着，在实践中，这些胜利很少被看到。当然，经常有些时候你真的没有选择，只能依靠虚拟调用。


##### 结论

有效地测量虚拟调用的延迟是一项相当复杂的工作，因为其中大部分是由于现代硬件的非常复杂的实施细节导致的许多交织副作用的直接后果。

在Go中，由于语言设计鼓励使用的习语，并考虑到编译器关于内联的（当前）局限性，所以可以有效地将动态分派视为几乎没有任何开销。

尽管如此，如果有疑问，人们应该总是测量他们的热门路径，并查看相关的绩效指标，以确定地断言动态分派是否最终成为问题。

（注意：我们将在本书后面的章节中看看编译器的内联功能。）


## 特殊情况和编译器技巧

本节将回顾我们在处理接口时每天遇到的一些最常见的特例。

现在，您应该对接口的工作方式有一个非常清晰的概念，所以我们会尽量在这里简明扼要地介绍一下。

### 空接口

空接口的数据结构是：`iface`没有`itab`，这应该跟你直觉一样吧。

有两个原因：

1. 由于空接口没有方法，因此与动态分配相关的所有内容都可以安全地从数据结构中删除。
2. 随着虚拟表不见了，空接口本身的类型，始终是相同的，不要被它保存的数据的类型迷惑，（即我们谈论的是这个空接口，而不是一个空接口）。

注意：类似于我们使用的符号`iface`，我们用`eface<T>`表示保存类型T的空接口。

`eface`是根类型,代表了运行时的空接口（[src/runtime/runtime2.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/runtime2.go#L148-L151)）。

其定义如下所示：

````go
type eface struct { // 16 bytes on a 64bit arch
    _type *_type
    data  unsafe.Pointer
}
````

其中`_type`保存了`data`指针所指向的值的类型信息。
正如预期的那样，`itab`完全被抛弃了。

虽然空的接口可以重用数据`iface`结构（它毕竟是`eface`的超集），但运行时选择区分这两个主要原因：空间效率和代码清晰度。

### 持有标量类型的接口

在前面的章节（[#接口解剖](#%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E6%A6%82%E8%BF%B0)）,我们已经提到，即使将一个简单的标量类型（如整数）存储到接口中也会导致堆分配。

是时候看看为什么会这样了。

考虑下这两个基准测试([eface_scalar_test.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/eface_scalar_test.go)):

````go
func BenchmarkEfaceScalar(b *testing.B) {
    var Uint uint32
    b.Run("uint32", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            Uint = uint32(i)
        }
    })
    var Eface interface{}
    b.Run("eface32", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            Eface = uint32(i)
        }
    })
}
````
````shell
$ go test -benchmem -bench=. ./eface_scalar_test.go
BenchmarkEfaceScalar/uint32-8         	2000000000	   0.54 ns/op	  0 B/op     0 allocs/op
BenchmarkEfaceScalar/eface32-8        	 100000000	   12.3 ns/op	  4 B/op     1 allocs/op
````

1. 对于一个简单的赋值操作，这是一个2个数量级的性能差异
2. 我们可以看到第二个基准测试必须在每次迭代中分配4个额外的字节。

显然，在第二种情况下，一些隐藏的重型机械正在起步：我们需要看看生成的汇编。

对于第一个基准测试，编译器完全按照您对赋值操作的期望进行生成：

````assembly
;; Uint = uint32(i)
0x000d MOVL	DX, (AX)
````

然而，在第二个基准中，事情变得更加复杂：

````assembly
;; Eface = uint32(i)
0x0050 MOVL	CX, ""..autotmp_3+36(SP)
0x0054 LEAQ	type.uint32(SB), AX
0x005b MOVQ	AX, (SP)
0x005f LEAQ	""..autotmp_3+36(SP), DX
0x0064 MOVQ	DX, 8(SP)
0x0069 CALL	runtime.convT2E32(SB)
0x006e MOVQ	24(SP), AX
0x0073 MOVQ	16(SP), CX
0x0078 MOVQ	"".&Eface+48(SP), DX
0x007d MOVQ	CX, (DX)
0x0080 MOVL	runtime.writeBarrier(SB), CX
0x0086 LEAQ	8(DX), DI
0x008a TESTL	CX, CX
0x008c JNE	148
0x008e MOVQ	AX, 8(DX)
0x0092 JMP	46
0x0094 CALL	runtime.gcWriteBarrier(SB)
0x0099 JMP	46
````

这只是赋值，不是完全的基准测试！

我们先一点点的去学这个代码吧。

#### 第一步：创建接口

````assembly
0x0050 MOVL	CX, ""..autotmp_3+36(SP)
0x0054 LEAQ	type.uint32(SB), AX
0x005b MOVQ	AX, (SP)
0x005f LEAQ	""..autotmp_3+36(SP), DX
0x0064 MOVQ	DX, 8(SP)
0x0069 CALL	runtime.convT2E32(SB)
0x006e MOVQ	24(SP), AX
0x0073 MOVQ	16(SP), CX
````

该列表的第一部分实例化了空接口`eface<uint32>`,我们稍后将它分配给`Eface`。

我们已经在关于创建接口的部分研究过类似的代码（[#创建一个接口](#%E5%88%9B%E5%BB%BA%E4%B8%80%E4%B8%AA%E6%8E%A5%E5%8F%A3)），除了这个代码在调用`runtime.convT2I32`而不是`runtime.convT2E32`; 尽管如此，这应该看起来很熟悉。

事实证明，`runtime.convT2I32`和`runtime.convT2E32`都属于的一个更大的方法集，他们的工作就是从标量（或字符串或数组，或某种情况）来实例化一个指定的接口或空接口。

这个方法集由10个符号组成，每个都是(`eface/iface, 16/32/64/string/slice`)的组合：

````go
// empty interface from scalar value
func convT2E16(t *_type, elem unsafe.Pointer) (e eface) {}
func convT2E32(t *_type, elem unsafe.Pointer) (e eface) {}
func convT2E64(t *_type, elem unsafe.Pointer) (e eface) {}
func convT2Estring(t *_type, elem unsafe.Pointer) (e eface) {}
func convT2Eslice(t *_type, elem unsafe.Pointer) (e eface) {}

// interface from scalar value
func convT2I16(tab *itab, elem unsafe.Pointer) (i iface) {}
func convT2I32(tab *itab, elem unsafe.Pointer) (i iface) {}
func convT2I64(tab *itab, elem unsafe.Pointer) (i iface) {}
func convT2Istring(tab *itab, elem unsafe.Pointer) (i iface) {}
func convT2Islice(tab *itab, elem unsafe.Pointer) (i iface) {}
````

你会注意到没有`convT2E8`和`convT2I8`函数：这是因为编译器优化了。我们会在这章节结束再来看这个问题。

所有这些函数都做几乎相同的事，不同的是他们的返回值(`iface`vs.`eface`)和分配到堆大内存大小。

让我们看看`runtime.convT2E32`([src/runtime/iface.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/iface.go#L308-L325))

````go
func convT2E32(t *_type, elem unsafe.Pointer) (e eface) {
    /* ...omitted debug stuff... */
    var x unsafe.Pointer
    if *(*uint32)(elem) == 0 {
        x = unsafe.Pointer(&zeroVal[0])
    } else {
        x = mallocgc(4, t, false)
        *(*uint32)(x) = *(*uint32)(elem)
    }
    e._type = t
    e.data = x
    return
}
````

这个函数初始化`eface`结构的`_type`字段，由调用者放在第一参数的位置"传递"进来的（记得：返回值都是在调用者它自己的栈帧中分配的）

对于`eface`结构的`data`字段，它的值依赖于传进来的第二个参数`elem`的值:

* 如果`elem`是零，`e.data`指向`runtime.zeroVal`,这是个特殊的全局变量，定义在运行时，代表了零值。在下一部分我们会讨论更多这个。
* 如果`elem`非零，函数在堆中分配四个字节(`x = mallocgc(4, t, false)`),然后用指向`elem`的值的内容来初始化(`*(*uint32)(x) = *(*uint32)(elem)`),然后赋值进`e.data`。

在这个情况，`e._type`保存`type.uint32`的地址（`LEAQ type.uint32(SB), AX`）,这个类型由标准库实现，所以类型地址只能在链接stdlib库时才能知道：

````shell
$ go tool nm eface_scalar_test.o | grep 'type\.uint32'
         U type.uint32
````

(`U`表示这个符号还没有定义在目标文件里面，将由其他目标文件（在这个例子是标准库）在链接时提供)

#### 赋值（部分1）

````assembly
0x0078 MOVQ	"".&Eface+48(SP), DX
0x007d MOVQ	CX, (DX)		;; Eface._type = ret._type
````

`runtime.convT2E32`的结果赋值给我们的`Eface`变量，是吗？

事实上，现在只有`_type`字段被赋值给`Eface._type`,`data`字段还没有赋值。

#### 赋值（部分2）或者要求垃圾回收帮我们赋值

````assembly
0x0080 MOVL	runtime.writeBarrier(SB), CX
0x0086 LEAQ	8(DX), DI	;; Eface.data = ret.data (indirectly via runtime.gcWriteBarrier)
0x008a TESTL	CX, CX
0x008c JNE	148
0x008e MOVQ	AX, 8(DX)	;; Eface.data = ret.data (direct)
0x0092 JMP	46
0x0094 CALL	runtime.gcWriteBarrier(SB)
0x0099 JMP	46
````

最后一块代码变得明显复杂了，是因为返回的`eface`的`data`指针分配到`Eface.data`的副作用：自从我们正在计算我们程序的内存图谱的时候（例如哪个部分的内存保存了哪个部分内存的引用），我们可能需要通知垃圾回收关于内存的改变，在这情况下，垃圾回收当前正在后台运行。

这被我们称之为写屏障，是Go并发垃圾回收的直接后果。

如果你有些模糊，不用担心，本书的下一章会提供一个Go的完整的垃圾回收讲解。

现在，我们只要记住当我们看到汇编代码调用`runtime.gcWriteBarrier`,表示正在处理指针计算和通知垃圾回收器。

总的来说，最后的一片代码只能做下面两件事之一：

* 如果写屏障当前处于非活动状态，它直接把`ret.data`赋值给`Eface.data`(`MOVQ AX, 8(DX)`)。
* 如果写屏障处于活动状态，它会礼貌地要求垃圾收集器帮我（`LEAQ 8(DX), DI`+ `CALL runtime.gcWriteBarrier(SB)`）进行赋值。

（再次，现在尽量不要太担心这个。）

瞧，我们有一个完整的接口，它拥有一个简单的标量类型（`uint32`）。

#### 结论

在实践中，绑定一个标量的值到一个接口是不常用的，基于各种理由，它可能是昂贵的操作，所以知道背后的原理是很重要的。

说到开销，我们已经提及过在某些特殊情况，编译器实现了各种优化去避免分配一个接口。我们快速看看3个优化，作为本节的结尾吧。

#### 接口技巧1:字节大小的值

考虑下面初始化一个`eface<uint8>`（[eface_scalar_test.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/eface_scalar_test.go)）的基准测试:
````go
func BenchmarkEfaceScalar(b *testing.B) {
    b.Run("eface8", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            // LEAQ    type.uint8(SB), BX
            // MOVQ    BX, (CX)
            // MOVBLZX AL, SI
            // LEAQ    runtime.staticbytes(SB), R8
            // ADDQ    R8, SI
            // MOVL    runtime.writeBarrier(SB), R9
            // LEAQ    8(CX), DI
            // TESTL   R9, R9
            // JNE     100
            // MOVQ    SI, 8(CX)
            // JMP     40
            // MOVQ    AX, R9
            // MOVQ    SI, AX
            // CALL    runtime.gcWriteBarrier(SB)
            // MOVQ    R9, AX
            // JMP     40
            Eface = uint8(i)
        }
    })
}
````

````shell
$ go test -benchmem -bench=BenchmarkEfaceScalar/eface8 ./eface_scalar_test.go
BenchmarkEfaceScalar/eface8-8         	2000000000	   0.88 ns/op	  0 B/op     0 allocs/op
````

我们注意到在这个情况，一个字节大小的值，编译没有为它调用`runtime.convT2E/runtime.convT2I`和在堆中分配内存，而是重用一个全局变量的地址，这个变量在运行时就已经保存了一个字节的值,这个值就是我们要找的了：`LEAQ runtime.staticbytes(SB), R8`。

`runtime.staticbytes`代码在[src/runtime/iface.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/iface.go#L619-L653),看上去像下面那样:

````go
// staticbytes is used to avoid convT2E for byte-sized values.
var staticbytes = [...]byte{
    0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08, 0x09, 0x0a, 0x0b, 0x0c, 0x0d, 0x0e, 0x0f,
    0x10, 0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17, 0x18, 0x19, 0x1a, 0x1b, 0x1c, 0x1d, 0x1e, 0x1f,
    0x20, 0x21, 0x22, 0x23, 0x24, 0x25, 0x26, 0x27, 0x28, 0x29, 0x2a, 0x2b, 0x2c, 0x2d, 0x2e, 0x2f,
    0x30, 0x31, 0x32, 0x33, 0x34, 0x35, 0x36, 0x37, 0x38, 0x39, 0x3a, 0x3b, 0x3c, 0x3d, 0x3e, 0x3f,
    0x40, 0x41, 0x42, 0x43, 0x44, 0x45, 0x46, 0x47, 0x48, 0x49, 0x4a, 0x4b, 0x4c, 0x4d, 0x4e, 0x4f,
    0x50, 0x51, 0x52, 0x53, 0x54, 0x55, 0x56, 0x57, 0x58, 0x59, 0x5a, 0x5b, 0x5c, 0x5d, 0x5e, 0x5f,
    0x60, 0x61, 0x62, 0x63, 0x64, 0x65, 0x66, 0x67, 0x68, 0x69, 0x6a, 0x6b, 0x6c, 0x6d, 0x6e, 0x6f,
    0x70, 0x71, 0x72, 0x73, 0x74, 0x75, 0x76, 0x77, 0x78, 0x79, 0x7a, 0x7b, 0x7c, 0x7d, 0x7e, 0x7f,
    0x80, 0x81, 0x82, 0x83, 0x84, 0x85, 0x86, 0x87, 0x88, 0x89, 0x8a, 0x8b, 0x8c, 0x8d, 0x8e, 0x8f,
    0x90, 0x91, 0x92, 0x93, 0x94, 0x95, 0x96, 0x97, 0x98, 0x99, 0x9a, 0x9b, 0x9c, 0x9d, 0x9e, 0x9f,
    0xa0, 0xa1, 0xa2, 0xa3, 0xa4, 0xa5, 0xa6, 0xa7, 0xa8, 0xa9, 0xaa, 0xab, 0xac, 0xad, 0xae, 0xaf,
    0xb0, 0xb1, 0xb2, 0xb3, 0xb4, 0xb5, 0xb6, 0xb7, 0xb8, 0xb9, 0xba, 0xbb, 0xbc, 0xbd, 0xbe, 0xbf,
    0xc0, 0xc1, 0xc2, 0xc3, 0xc4, 0xc5, 0xc6, 0xc7, 0xc8, 0xc9, 0xca, 0xcb, 0xcc, 0xcd, 0xce, 0xcf,
    0xd0, 0xd1, 0xd2, 0xd3, 0xd4, 0xd5, 0xd6, 0xd7, 0xd8, 0xd9, 0xda, 0xdb, 0xdc, 0xdd, 0xde, 0xdf,
    0xe0, 0xe1, 0xe2, 0xe3, 0xe4, 0xe5, 0xe6, 0xe7, 0xe8, 0xe9, 0xea, 0xeb, 0xec, 0xed, 0xee, 0xef,
    0xf0, 0xf1, 0xf2, 0xf3, 0xf4, 0xf5, 0xf6, 0xf7, 0xf8, 0xf9, 0xfa, 0xfb, 0xfc, 0xfd, 0xfe, 0xff,
}
````

在上面这个数组里面，用对偏移，编译器就能有效的避免分配一个额外的堆，并且仍然引用任何可表示为单个字节的值。

这里有什么感觉不对劲，但是......你能告诉吗？

尽管我们正在操作的指针保存了一个全局变量的地址，但它的生存期与整个程序的相同，所生成的代码仍然嵌入了与写屏障相关的代码。
也就是说`runtime.staticbytes`，永远不会被垃圾收集，无论内存的哪一部分都引用它，所以在这种情况下，我们不应该为写入障碍的开销付出代价。

#### 接口技巧2:静态推断

考虑这个基准测试，它用一个编译期就能确定的值来实例化一个`eface<uint64>`([eface_scalar_test.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/eface_scalar_test.go)):

````go
func BenchmarkEfaceScalar(b *testing.B) {
    b.Run("eface-static", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            // LEAQ  type.uint64(SB), BX
            // MOVQ  BX, (CX)
            // MOVL  runtime.writeBarrier(SB), SI
            // LEAQ  8(CX), DI
            // TESTL SI, SI
            // JNE   92
            // LEAQ  "".statictmp_0(SB), SI
            // MOVQ  SI, 8(CX)
            // JMP   40
            // MOVQ  AX, SI
            // LEAQ  "".statictmp_0(SB), AX
            // CALL  runtime.gcWriteBarrier(SB)
            // MOVQ  SI, AX
            // LEAQ  "".statictmp_0(SB), SI
            // JMP   40
            Eface = uint64(42)
        }
    })
}
````

````shell
$ go test -benchmem -bench=BenchmarkEfaceScalar/eface-static ./eface_scalar_test.go
BenchmarkEfaceScalar/eface-static-8    	2000000000	   0.81 ns/op	  0 B/op     0 allocs/op
````

从汇编代码可以看到，编译器完全优化掉了`runtime.convT2E64`的调用，而是通过加载自动生成的全局变量的地址来构建一个空接口，而这个全局变量的值就是我们要找的：`LEAQ "".statictmp_0(SB), SI`(注意`SB`，表示一个全局变量)。

我们可以更好地使用我们之前破解的脚本直观地看到发生了什么：`dump_sym.sh`。

````shell
$ GOOS=linux GOARCH=amd64 go tool compile eface_scalar_test.go
$ GOOS=linux GOARCH=amd64 go tool link -o eface_scalar_test.bin eface_scalar_test.o
$ ./dump_sym.sh eface_scalar_test.bin .rodata main.statictmp_0
.rodata file-offset: 655360
.rodata VMA: 4849664
main.statictmp_0 VMA: 5145768
main.statictmp_0 SIZE: 8

0000000 002a 0000 0000 0000                    
0000008
````

正如所料，`main.statictmp_0`是一个八字节变量，它的值是`0x000000000000002a`,即`$42`。

#### 接口技巧3：零值

对于这个最后的技巧,下面的基准测试是用一个零值来初始化`eface<uint32>`[eface_scalar_test.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/eface_scalar_test.go):

````go
func BenchmarkEfaceScalar(b *testing.B) {
    b.Run("eface-zeroval", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            // MOVL  $0, ""..autotmp_3+36(SP)
            // LEAQ  type.uint32(SB), AX
            // MOVQ  AX, (SP)
            // LEAQ  ""..autotmp_3+36(SP), CX
            // MOVQ  CX, 8(SP)
            // CALL  runtime.convT2E32(SB)
            // MOVQ  16(SP), AX
            // MOVQ  24(SP), CX
            // MOVQ  "".&Eface+48(SP), DX
            // MOVQ  AX, (DX)
            // MOVL  runtime.writeBarrier(SB), AX
            // LEAQ  8(DX), DI
            // TESTL AX, AX
            // JNE   152
            // MOVQ  CX, 8(DX)
            // JMP   46
            // MOVQ  CX, AX
            // CALL  runtime.gcWriteBarrier(SB)
            // JMP   46
            Eface = uint32(i - i) // outsmart the compiler (avoid static inference)
        }
    })
}
````

````shell
$ go test -benchmem -bench=BenchmarkEfaceScalar/eface-zero ./eface_scalar_test.go
BenchmarkEfaceScalar/eface-zeroval-8  	 500000000	   3.14 ns/op	  0 B/op     0 allocs/op
````

首先，注意我们是如何利用`uint32(i - i)`而不是`uint32(0)`阻止编译器回退到优化＃2（静态推断）。

(确实，我们可以定义一个全局的零值变量而编译器就会被迫采取保守的路径，但是我们是有趣的人，不要做无趣的人)。

这个生成的代码现在看上去跟平常没什么两样，但是没有任何内存分配，发生了什么。

我们之前分析`runtime.convT2E32`的时候提过，分配内存可能会被优化掉，而且用的是类似#1 (字节大小的值)的方法：当一些代码需要引用一个变量，而这个变量是个零值，编译器就会返回一个全局变量的地址给它，并且这个全局变量在运行时一直是零的。

类似于`runtime.staticbytes`,我们能在运行时代码找到这个变量([src/runtime/hashmap.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/hashmap.go#L1248-L1249)):

````go
const maxZero = 1024 // must match value in ../cmd/compile/internal/gc/walk.go
var zeroVal [maxZero]byte
````
我们的优化之旅就到这里了。

我们汇总了之前看过的测试结果来结束这节。

````shell
$ go test -benchmem -bench=. ./eface_scalar_test.go
BenchmarkEfaceScalar/uint32-8         	2000000000	   0.54 ns/op	  0 B/op     0 allocs/op
BenchmarkEfaceScalar/eface32-8        	 100000000	   12.3 ns/op	  4 B/op     1 allocs/op
BenchmarkEfaceScalar/eface8-8         	2000000000	   0.88 ns/op	  0 B/op     0 allocs/op
BenchmarkEfaceScalar/eface-zeroval-8  	 500000000	   3.14 ns/op	  0 B/op     0 allocs/op
BenchmarkEfaceScalar/eface-static-8    	2000000000	   0.81 ns/op	  0 B/op     0 allocs/op
````

### 关于零值变量

正如我们刚刚看到的，当由接口生成的数据碰巧引用一个零值时，`runtime.convT2*`函数族避免了堆分配。

这种优化不是特定于接口的，它实际上是Go运行时所做的更广泛工作的一部分，以确保在需要指向零值的指针时，通过采用特殊的，始终不变的运行时零值变量地址来避免不必要的分配。

我们可以用一个简单的程序（[zeroval.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/zeroval.go)）来证实这一点：

````go
//go:linkname zeroVal runtime.zeroVal
var zeroVal uintptr

type eface struct{ _type, data unsafe.Pointer }

func main() {
    x := 42
    var i interface{} = x - x // outsmart the compiler (avoid static inference)

    fmt.Printf("zeroVal = %p\n", &zeroVal)
    fmt.Printf("      i = %p\n", ((*eface)(unsafe.Pointer(&i))).data)
}
````

````shell
$ go run zeroval.go
zeroVal = 0x5458e0
      i = 0x5458e0
````

如预期。

请注意`//go:linkname`允许我们引用外部符号的指令：

> // go：linkname指令指示编译器使用“importpath.name”作为源代码中声明为“localname”的变量或函数的目标文件符号名称。由于此指令可以破坏类型系统和程序包模块性，因此只能在导入“不安全(unsafe)”的文件中启用。

### 关于零大小变量

与零值类似，Go程序中的一个非常常见的技巧是依靠实例化大小为0的对象（例如`struct{}{}`）不会导致分配。

正式的Go规格文档（本章结尾部分）以一个注释说明了这一点：

> 如果一个结构或数组类型不包含大小大于零的字段（或元素），则它的大小为零。内存中两个不同的零大小的变量可能具有相同的地址。

“可能”在“可能在内存中具有相同的地址”意味着编译器并不能保证这个事实是真实的，尽管在官方Go编译器（gc）的当前实现中它一直并且仍然是这种情况。

像往常一样，我们可以通过一个简单的程序（[zerobase.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/zerobase.go)）来确认：

````go
func main() {
    var s struct{}
    var a [42]struct{}

    fmt.Printf("s = % p\n", &s)
    fmt.Printf("a = % p\n", &a)
}
````

````shell
$ go run zerobase.go
s = 0x546fa8
a = 0x546fa8
````

如果我们想知道这个地址背后隐藏着什么，我们可以简单地在这个二进制文件中窥视一下：

````shell
$ go build -o zerobase.bin zerobase.go && objdump -t zerobase.bin | grep 546fa8 
0000000000546fa8 g O .noptrbss 0000000000000008 runtime.zerobase
````

然后在运行时源代码（[src/runtime/malloc.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/malloc.go#L516-L517)）中找到这个变量`runtime.zerobase`

````go
// base address for all 0-byte allocations
var zerobase uintptr
````

确实如此：

````go
//go:linkname zerobase runtime.zerobase
var zerobase uintptr

func main() {
    var s struct{}
    var a [42]struct{}

    fmt.Printf("zerobase = %p\n", &zerobase)
    fmt.Printf("       s = %p\n", &s)
    fmt.Printf("       a = %p\n", &a)
}
````

````shell
$ go run zerobase.go
zerobase = 0x546fa8
       s = 0x546fa8
       a = 0x546fa8
````

## 接口组合

接口组合确实没有什么特别之处，它只是编译器暴露的语法糖。

考虑下面的程序（[compound_interface.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/compound_interface.go)）：

````go
type Adder interface{ Add(a, b int32) int32 }
type Subber interface{ Sub(a, b int32) int32 }
type Mather interface {
    Adder
    Subber
}

type Calculator struct{ id int32 }
func (c *Calculator) Add(a, b int32) int32 { return a + b }
func (c *Calculator) Sub(a, b int32) int32 { return a - b }

func main() {
    calc := Calculator{id: 6754}
    var m Mather = &calc
    m.Sub(10, 32)
}
````

像往常一样，编译器生成相应`itab`的`iface<Mather, *Calculator>`：

````shell
$ GOOS=linux GOARCH=amd64 go tool compile -S compound_interface.go | \
  grep -A 7 '^go.itab.\*"".Calculator,"".Mather'
go.itab.*"".Calculator,"".Mather SRODATA dupok size=40
    0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
    0x0010 5e 33 ca c8 00 00 00 00 00 00 00 00 00 00 00 00  ^3..............
    0x0020 00 00 00 00 00 00 00 00                          ........
    rel 0+8 t=1 type."".Mather+0
    rel 8+8 t=1 type.*"".Calculator+0
    rel 24+8 t=1 "".(*Calculator).Add+0
    rel 32+8 t=1 "".(*Calculator).Sub+0
````

我们可以从重定位指令中看到，编译器生成的虚拟表既包含Adder的方法也包含属于Subber的方法：

````
rel 24+8 t=1 "".(*Calculator).Add+0
rel 32+8 t=1 "".(*Calculator).Sub+0
````

就像我们所说的，接口组合，没有什么秘诀。

一个不相关提示，这个小程序演示了一些直到现在才能看到的东西：生成的`itab`特指一个指针指向一个`构造器`,与之对应的是具体的值，而这两个东西都反映在符号名上(`go.itab.*"".Calculator,"".Mather`)，同时也反映在`_type`,如(`type.*"".Calculator`)。

这与用于命名方法符号的语义是一致的，就像我们在本章开始时看到的那样。

## 断言

从实现和成本的角度来看类型断言，并以此来结束本章。

### 类型断言

考虑这个简短的程序（[eface_to_type.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/eface_to_type.go)）：

````go
var j uint32
var Eface interface{} // outsmart compiler (avoid static inference)

func assertion() {
    i := uint64(42)
    Eface = i
    j = Eface.(uint32)
}
````

下面是带注解的`j = Eface.(uint32)`的汇编代码：

````assembly
0x0065 00101 MOVQ	"".Eface(SB), AX		;; AX = Eface._type
0x006c 00108 MOVQ	"".Eface+8(SB), CX		;; CX = Eface.data
0x0073 00115 LEAQ	type.uint32(SB), DX		;; DX = type.uint32
0x007a 00122 CMPQ	AX, DX				;; Eface._type == type.uint32 ?
0x007d 00125 JNE	162				;; no? panic our way outta here
0x007f 00127 MOVL	(CX), AX			;; AX = *Eface.data
0x0081 00129 MOVL	AX, "".j(SB)			;; j = AX = *Eface.data
;; exit
0x0087 00135 MOVQ	40(SP), BP
0x008c 00140 ADDQ	$48, SP
0x0090 00144 RET
;; panic: interface conversion: <iface> is <have>, not <want>
0x00a2 00162 MOVQ	AX, (SP)			;; have: Eface._type
0x00a6 00166 MOVQ	DX, 8(SP)			;; want: type.uint32
0x00ab 00171 LEAQ	type.interface {}(SB), AX	;; AX = type.interface{} (eface)
0x00b2 00178 MOVQ	AX, 16(SP)			;; iface: AX
0x00b7 00183 CALL	runtime.panicdottypeE(SB)	;; func panicdottypeE(have, want, iface *_type)
0x00bc 00188 UNDEF
0x00be 00190 NOP
````

这里没有什么让人意外的地方：代码将持有`Eface._type`的地址与`type.uint32`地址进行比较，正如我们以前所见，`type.uint32`是由标准库公开的全局符号，持有`_type`结构的内容，是描述`uint32`的结构。

如果`_type`指针匹配，那么一切都很好，我们可以自由地分配`*Eface.data`到`j`; 否则，我们调用`runtime.panicdottypeE`来发出描述这不匹配的异常。

`runtime.panicdottypeE`是一个非常简单的函数，它不会超出你的预期（[src/runtime/iface.go](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/iface.go#L235-L245)）：

````go
// panicdottypeE is called when doing an e.(T) conversion and the conversion fails.
// have = the dynamic type we have.
// want = the static type we're trying to convert to.
// iface = the static type we're converting from.
func panicdottypeE(have, want, iface *_type) {
    haveString := ""
    if have != nil {
        haveString = have.string()
    }
    panic(&TypeAssertionError{iface.string(), haveString, want.string(), ""})
}
````

#### 性能如何？

让我们看看在这里得到什么了：一堆`MOV`指令，一条很容易预测的分支，最后但同样重要的一个指针解引用(`j = *Eface.data`)(因为我们之前用具体的值初始化了这个接口，所以它只能在那里，否则我们只能直接拷贝`Eface.data`指针)。

真的，这甚至不值得进行微基准测试。

类似于我们之前测量的动态分配开销，理论上它本身几乎是没有开销的。在实践中，真正的开销是取决于你的代码路径如何设计缓存友好性等问题。

无论如何，一个简单的微基准测试可能会太偏向于告诉我们任何有用的东西。

总而言之，我们最终会得到与往常一样的旧建议：测量您的特定用例，检查处理器的性能计数器，并确定这是否对您的热门路径有明显影响。

它可能。它可能不会。它很可能不会。

### 类型switch(Type-switches)

当然，类型判断有点棘手。考虑下面的代码（[eface_to_type.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/eface_to_type.go)）：

````go
var j uint32
var Eface interface{} // outsmart compiler (avoid static inference)

func typeSwitch() {
    i := uint32(42)
    Eface = i
    switch v := Eface.(type) {
    case uint16:
        j = uint32(v)
    case uint32:
        j = v
    }
}
````

下面是十分简单的类型判断汇编代码(有注解的)：

````assembly
;; switch v := Eface.(type)
0x0065 00101 MOVQ	"".Eface(SB), AX	;; AX = Eface._type
0x006c 00108 MOVQ	"".Eface+8(SB), CX	;; CX = Eface.data
0x0073 00115 TESTQ	AX, AX			;; Eface._type == nil ?
0x0076 00118 JEQ	153			;; yes? exit the switch
0x0078 00120 MOVL	16(AX), DX		;; DX = Eface.type._hash
;; case uint32
0x007b 00123 CMPL	DX, $-800397251		;; Eface.type._hash == type.uint32.hash ?
0x0081 00129 JNE	163			;; no? go to next case (uint16)
0x0083 00131 LEAQ	type.uint32(SB), BX	;; BX = type.uint32
0x008a 00138 CMPQ	BX, AX			;; type.uint32 == Eface._type ? (hash collision?)
0x008d 00141 JNE	206			;; no? clear BX and go to next case (uint16)
0x008f 00143 MOVL	(CX), BX		;; BX = *Eface.data
0x0091 00145 JNE	163			;; landsite for indirect jump starting at 0x00d3
0x0093 00147 MOVL	BX, "".j(SB)		;; j = BX = *Eface.data
;; exit
0x0099 00153 MOVQ	40(SP), BP
0x009e 00158 ADDQ	$48, SP
0x00a2 00162 RET
;; case uint16
0x00a3 00163 CMPL	DX, $-269349216		;; Eface.type._hash == type.uint16.hash ?
0x00a9 00169 JNE	153			;; no? exit the switch
0x00ab 00171 LEAQ	type.uint16(SB), DX	;; DX = type.uint16
0x00b2 00178 CMPQ	DX, AX			;; type.uint16 == Eface._type ? (hash collision?)
0x00b5 00181 JNE	199			;; no? clear AX and exit the switch
0x00b7 00183 MOVWLZX	(CX), AX		;; AX = uint16(*Eface.data)
0x00ba 00186 JNE	153			;; landsite for indirect jump starting at 0x00cc
0x00bc 00188 MOVWLZX	AX, AX			;; AX = uint16(AX) (redundant)
0x00bf 00191 MOVL	AX, "".j(SB)		;; j = AX = *Eface.data
0x00c5 00197 JMP	153			;; we're done, exit the switch
;; indirect jump table
0x00c7 00199 MOVL	$0, AX			;; AX = $0
0x00cc 00204 JMP	186			;; indirect jump to 153 (exit)
0x00ce 00206 MOVL	$0, BX			;; BX = $0
0x00d3 00211 JMP	145			;; indirect jump to 163 (case uint16)
````

再一次，如果你仔细地逐步看生成的代码并仔细阅读相应的注释，你会发现那里没有黑魔法。

控制流程起初可能看起来有点复杂，因为它会跳来跳去，但除了这个之外，就是原始Go代码的忠实再现了。

虽然有不少有趣的事情需要注意。

#### 注1：布局

首先，注意生成的代码的高级布局，它与原始的switch语句非常接近：

1. 我们找到一个初始指令块，它加载我们感兴趣的`_type`变量，并检查nil指针，以防万一。
2. 然后，我们得到N个逻辑块，每个逻辑块对应于原始switch语句中描述的一种情况。
3. 最后，最后一个块定义了一种间接跳转表，允许控制流从一种情况跳转到另一种情况，同时确保在路上正确地重置脏寄存器。

虽然事后看来很明显，但第二点非常重要，因为它意味着type-switch语句生成的指令数量纯粹是它所描述的case数量的一个因素。

实际上，这可能会导致令人惊讶的性能问题，例如，如果在错误的路径上使用大量类型切换语句，则会产生大量指令并最终导致L1i缓存抖动。

关于上面简单的switch语句的布局的另一个有趣的事实是在生成的代码中case的顺序。在我们原来的Go代码中，`case uint16`先出现，然后是`case uint32`。然而，在编译器生成的汇编中，它们的顺序已经颠倒过来，`case uint32`现在是第一，`case uint16`是第二个。

在这个特殊情况下，这种重新排序对我们来说是一个净胜，不过是单纯的运气，AFAICT。事实上，如果你花时间用type-switch进行实验，特别是那些有两个以上case的switch，你会发现编译器总是使用某种确定性启发式来改变case顺序。

这些启发式算法是什么，我不知道（但一如既往，如果你愿意，我很乐意）。

#### 注2：O（n）

其次，注意控制流程是如何从一种case向另一种case盲目跳转，直到它落在一个评估为真或最终达到switch语句结尾的状态。

再一次，当一个人真的停下来思考它时（“它还能如何工作？”），当在更高层次推理时，这很容易被忽视。在实践中，这意味着评估类型switch语句的成本，跟case数呈线性增长：它的O(n)。

同样，有效地评估具有N个case的type-switch语句具有与评估N类型断言相同的时间复杂度。正如我们所说，这里没有魔力。

用一堆基准测试（[eface_to_type_test.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/eface_to_type_test.go)）很容易确认：

````go
var j uint32
var eface interface{} = uint32(42)

func BenchmarkEfaceToType(b *testing.B) {
    b.Run("switch-small", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            switch v := eface.(type) {
            case int8:
                j = uint32(v)
            case int16:
                j = uint32(v)
            default:
                j = v.(uint32)
            }
        }
    })
    b.Run("switch-big", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            switch v := eface.(type) {
            case int8:
                j = uint32(v)
            case int16:
                j = uint32(v)
            case int32:
                j = uint32(v)
            case int64:
                j = uint32(v)
            case uint8:
                j = uint32(v)
            case uint16:
                j = uint32(v)
            case uint64:
                j = uint32(v)
            default:
                j = v.(uint32)
            }
        }
    })
}
````

````shell
benchstat <(go test -benchtime=1s -bench=. -count=3 ./eface_to_type_test.go)
name                        time/op
EfaceToType/switch-small-8  1.91ns ± 2%
EfaceToType/switch-big-8    3.52ns ± 1%
````

在所有额外的case下，第二种类型switch的确在每个迭代中需要两倍长的时间。

作为读者的一个有趣的练习，尝试在上面的任何一个基准中（任何地方）添加一个`case uint32`，你会看到他们的性能大幅提升：

````shell
benchstat <(go test -benchtime=1s -bench=. -count=3 ./eface_to_type_test.go)
name                        time/op
EfaceToType/switch-small-8  1.63ns ± 1%
EfaceToType/switch-big-8    2.17ns ± 1%
````

使用我们在本章中收集的所有工具和知识，您应该能够解释数字背后的基本原理。祝玩的开心！

#### 注3：类型哈希&指针比较

最后，请注意每种情况下的类型比较总是分两个阶段进行：

1. 比较类型的哈希（`_type.hash`）
2. 如果它们匹配，则直接比较每个`_type`指针的相应存储器地址。

由于每个`_type`结构都由编译器生成一次，并存储在该`.rodata`节中的全局变量中，因此我们保证每个类型在程序的整个生命周期都被分配一个唯一的地址。

在这种情况下，进行额外的指针比较是有意义的，以确保成功匹配不仅仅是哈希冲突的结果。但是这提出了一个明显的问题：为什么不直接比较指针首先，完全放弃类型哈希的概念？特别是当我们在前面看到简单类型断言时，根本不使用类型散列。

答案是我没有丝毫的线索，并且肯定会对此有所启发。与往常一样，如果您知道更多信息，请随时开issue。

说起类型散列，是什么让我们知道，`$-800397251`对应于`type.uint32.hash`和`$-269349216`于`type.uint16.hash`，你可能想知道？当然，这很难（[eface_type_hash.go](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/eface_type_hash.go)）：

````go
// simplified definitions of runtime's eface & _type types
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
type _type struct {
    size    uintptr
    ptrdata uintptr
    hash    uint32
    /* omitted lotta fields */
}

var Eface interface{}
func main() {
    Eface = uint32(42)
    fmt.Printf("eface<uint32>._type.hash = %d\n",
        int32((*eface)(unsafe.Pointer(&Eface))._type.hash))

    Eface = uint16(42)
    fmt.Printf("eface<uint16>._type.hash = %d\n",
        int32((*eface)(unsafe.Pointer(&Eface))._type.hash))
}
````

````shell
$ go run eface_type_hash.go
eface<uint32>._type.hash = -800397251
eface<uint16>._type.hash = -269349216
````

## 结论

这就是接口。

我希望本章给出了大部分关于接口及其内部的答案。最重要的是，它提供了所有必要工具和技能，方便你需要挖掘更多细节的时候能用到。

如果您有任何问题或建议，请不要犹豫，在`chapter2`开issue吧: prefix!

## 链接

- [[Official] Go 1.1 Function Calls](https://docs.google.com/document/d/1bMwCey-gmqZVTpRax-ESeVuZGmjwbocYs1iHplK-cjo/pub)
- [[Official] The Go Programming Language Specification](https://golang.org/ref/spec)
- [The Gold linker by Ian Lance Taylor](https://lwn.net/Articles/276782/)
- [ELF: a linux executable walkthrough](https://i.imgur.com/EL7lT1i.png)
- [VMA vs LMA?](https://www.embeddedrelated.com/showthread/comp.arch.embedded/77071-1.php)
- [In C++ why and how are virtual functions slower?](https://softwareengineering.stackexchange.com/questions/191637/in-c-why-and-how-are-virtual-functions-slower)
- [The cost of dynamic (virtual calls) vs. static (CRTP) dispatch in C++](https://eli.thegreenplace.net/2013/12/05/the-cost-of-dynamic-virtual-calls-vs-static-crtp-dispatch-in-c)
- [Why is it faster to process a sorted array than an unsorted array?](https://stackoverflow.com/a/11227902)
- [Is accessing data in the heap faster than from the stack?](https://stackoverflow.com/a/24057744)
- [CPU cache](https://en.wikipedia.org/wiki/CPU_cache)
- [CppCon 2014: Mike Acton "Data-Oriented Design and C++"](https://www.youtube.com/watch?v=rX0ItVEVjHc)
- [CppCon 2017: Chandler Carruth "Going Nowhere Faster"](https://www.youtube.com/watch?v=2EWejmkKlxs)
- [What is the difference between MOV and LEA?](https://stackoverflow.com/a/1699778)