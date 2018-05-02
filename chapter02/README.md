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
> - 顶级func的直接调用（`func TopLevel(x int) {}`）
> - 带有值接收器的直接方法调用（`func (Value) M(int) {}`）
> - 带有指针接收器的直接方法调用（`func (*Pointer) M(int) {}`）
> - 接口方法的间接调用（`type Interface interface { M(int) }`）
> - func类型的值的间接调用（`var literal = func(x int) {}`）

混合在一起，这些组合了10种可能的函数和调用类型组合：
> - 顶级func的直接调用 /
> - 带有值接收器的直接方法调用/
> - 带有指针接收器的直接方法调用/
> - 接口方法的间接调用/包含value方法的方法
> - 接口方法的间接调用/包含带value方法的指针
> - 接口方法的间接调用/包含带指针方法的指针
> - func类型的值的间接调用/设置为顶级func
> - func类型的值的间接调用/设置为value方法
> - func类型的值的间接调用/设置为指针方法
> - func类型的值的间接调用/设置为func文字

> （斜线将编译时已知的内容与运行时发现的内容区分开来。）
