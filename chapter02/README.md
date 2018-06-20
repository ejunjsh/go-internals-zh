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


