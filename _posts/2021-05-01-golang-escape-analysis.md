---
layout: post
title: "Go-escape-analysis"
subtitle: 'Go逃逸分析'
author: "Elvis"
header-style: text
mermaid: true
tags:
  - Golang
---



### 什么是逃逸分析？

这里引述周志明大大的原话：

> 在计算机语言编译器优化原理中，逃逸分析是指**分析指针动态范围**的方法，它同编译器优化原理的指针分析和外形分析相关联。当变量（或者对象）在方法中分配后，其**指针有可能被返回或者被全局引用**，这样就会**被其他过程或者线程所引用**，这种现象称作**指针（或者引用）的逃逸(Escape)**。



### 为什么要逃逸分析？

* 性能优化：逃逸分析的好处是为了减少GC的压力，不逃逸的对象分配在栈上，当函数返回时就回收了资源，不需要GC标记清除。

* 尘归尘土归土：逃逸分析完后可以确定哪些变量可以分配在栈上，栈的分配比堆快，可以没有发生逃逸的则有编译器在栈上分配。更有效使用内存。



### 怎么做逃逸分析？

验证某个函数的变量是否发生逃逸的方法有两个：

1. **go run -gcflags "-m -l" ./main.go** (-m打印逃逸分析信息，-l禁止内联编译)；例：

   ```go
    go run -gcflags "-m -l" ./demo.go                  
   # command-line-arguments
   ./demo.go:36:2: moved to heap: a
   ./demo.go:37:11: make([]*S, 2) does not escape
   ./demo.go:39:10: new(S) escapes to heap
   
   ```

2. **go tool compile -S main.go | grep runtime.newobject**（汇编代码中搜runtime.newobject指令，该指令用于生成堆对象）,例：

   ```go
   go tool compile -S demo.go | grep runtime.newobject
           0x0028 00040 (demo.go:36)       CALL    runtime.newobject(SB)
           0x0038 00056 (demo.go:39)       CALL    runtime.newobject(SB)
           rel 41+4 t=8 runtime.newobject+0
           rel 57+4 t=8 runtime.newobject+0
   ```



### 什么情况下变量会逃逸？

### 情况1

首先说一种最基本的情况：

> **在某个函数中new或字面量创建出的变量，将其指针作为函数返回值，则该变量一定发生逃逸。**

这是golang基础教程中经常举的：

```text
func test() *User{
    a := User{}
    return &a
}
```



### 情况2

当某个值取指针传给另一个函数，该值是否发生逃逸：

example1

```go
type User struct {
	Username string
	Age      int
}

func main() {
	u := &User{"elvis", 25}
	Older(u)
	Call(u)
}

func Call(u *User) {
	fmt.Printf("Info:%v",u)
}

func Older(u *User) {
	u.Age++
}
```

看一下逃逸情况：

```go
 go run -gcflags "-m -l" ./escape_demo.go
# command-line-arguments
./escape_demo.go:20:12: u does not escape       ==>OlderFunc 
./escape_demo.go:16:12: leaking param: u        ==>CallFunc
./escape_demo.go:17:12: ... argument does not escape  
./escape_demo.go:11:7: &User{...} escapes to heap  
```

可以看到，&User最后逃逸了，那么到底是OlderFunc的指针引用的值修改导致的逃逸呢，还是CallFunc的指针传递打印导致的逃逸？

那么修改一下程序：

> 只长大，不Call

example2

```go
func main() {
	u := &User{"elvis", 25}
	Older(u)
}

func Older(u *User) {
	u.Age++
}
```

看一下逃逸情况：

```go
 go run -gcflags "-m -l" ./escape_demo.go
# command-line-arguments
./escape_demo.go:14:12: u does not escape
./escape_demo.go:9:7: &User{...} does not escape
```

并没有发生逃逸。

> 只Call，不长大

```go
func main() {
	u := &User{"elvis", 25}
	Call(u)
}

func Call(u *User) {
	fmt.Printf("Info:%v",u)
}
```

看一下逃逸情况：

```go
 go run -gcflags "-m -l" ./escape_demo.go
# command-line-arguments
./escape_demo.go:15:11: leaking param: u
./escape_demo.go:16:12: ... argument does not escape
./escape_demo.go:11:7: &User{...} escapes to heap
Info:&{elvis 25}
```

&User再次逃逸了。那么这里可以确认两个问题。

1. 指针传给另一个函数，指针变量发生值传递，并不产生逃逸。
2. fmt.Printf打印的时候回让指针变量发生逃逸。



**那究竟为什么fmt.Printf打印方法会让指针变量发生逃逸呢？**

看看fmt.Printf的源码，最终找到了被传入的&User被赋值给了pp指针的一个成员变量：

```go
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
  p := newPrinter()       <=====   newPrinter()返回了p指针对象，可以根据情况1知道，p对象是逃逸的
	p.doPrintf(format, a)   <=====   这里的a就是&User
...
}

func (p *pp) doPrintf(format string, a []interface{}) {
...
p.printArg(a[argNum], rune(c))
...
}

func (p *pp) printArg(arg interface{}, verb rune) {
	p.arg = arg    <========  整个&User最后赋值给了p的arg变量
	p.value = reflect.Value{}
...
}
```



从fmt.Printf的源码可以看到，上面的arg 是一个interface{} 引用类型。其实相当于p.arg = &User，User本来没有逃逸的，被逃逸的指针对象变量引用了，所以也逃逸了。

简单总结：

> 被逃逸指针的成员变量所引用的指针变量也会变成逃逸。



### 情况3

我们再看上面备注中的代码例子：

```go
func main() {
	a := make([]*int,1)
	b := 12
	a[0] = &b
}
```

逃逸结果：

```go
go run -gcflags "-m -l" ./escape_demo.go
# command-line-arguments
./escape_demo.go:5:2: moved to heap: b
./escape_demo.go:4:11: make([]*int, 1) does not escapexxxxxxxxxx go run -gcflags "-m -l" ./escape_demo.go# command-line-arguments./escape_demo.go:5:2: moved to heap: b./escape_demo.go:4:11: make([]*int, 1) does not escape➜  testProj go run -gcflags "-m -l" main.go# command-line-arguments./main.go:7:2: moved to heap: b./main.go:6:11: main make([]*int, 1) does not escape
```

sliace a并没有发生逃逸，但是被a引用的b依然逃逸了。类似的情况同样发生在map和chan中：

```go
func main() {
	a := make([]*int,1)
	b := 20
	a[0] = &b

	c := make(map[string]*int)
	d := 21
	c["hello"]=&d

	e := make(chan *int,1)
	f := 22
	e <- &f
}
```

逃逸结果：

```go
go run -gcflags "-m -l" ./escape_demo.go
# command-line-arguments
./escape_demo.go:5:2: moved to heap: b
./escape_demo.go:9:2: moved to heap: d
./escape_demo.go:13:2: moved to heap: f
./escape_demo.go:4:11: make([]*int, 1) does not escape
./escape_demo.go:8:11: make(map[string]*int) does not escape
```

由此我们可以得出结论：

> **被指针类型的slice、map和chan引用的指针一定发生逃逸**



### 总结

我们得出了指针**必然发生逃逸**的三种情况：

- 将其指针作为函数返回值，则该变量一定发生逃逸（构造函数返回的指针变量一定逃逸）；
- 被逃逸指针的成员变量所引用的指针变量也会变成逃逸；
- 被指针类型的slice、map和chan引用的指针，一定发生逃逸；

同时我们也得出一些**必然不会逃逸**的情况：

- 指针被未发生逃逸的变量引用；
- 仅仅在函数内对变量做取址操作，而未将指针传出；



### 文章小结

在高并发的情况下，要注意逃逸分析的使用，因为当chan和slice，特别是map，如果变量发生逃逸了，GC的回收会变得高昂，从而进一步影响Go语言的并发效率。

