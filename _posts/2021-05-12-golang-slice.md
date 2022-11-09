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



### slice是什么

`slice` 翻译成中文就是`切片`，它和`数组（array）`很类似，可以用下标的方式进行访问，如果越界，就会产生 panic。但是它比数组更灵活，可以自动地进行扩容。

 Go 中数组赋值和函数传参都是值复制的，这就引出了slice的特点；用切片传数组参数，既可以达到节约内存的目的，也可以达到合理处理好共享内存的问题。



### slice的数据结构

```go
// runtime/slice.go
type slice struct {
	array unsafe.Pointer // 元素指针
	len   int // 长度 
	cap   int // 容量
}
```

`指针`，指向底层数组；
`长度`，表示切片可用元素的个数，也就是说使用下标对 slice 的元素进行访问时，下标不能超过 slice 的长度；
`容量`，底层数组的元素个数，容量 >= 长度。在底层数组不进行扩容的情况下，容量也是 slice 可以扩张的最大限度。

![](https://raw.githubusercontent.com/elvisNg/elvisng.github.io/master/img/in-post/post-golang-slice/slice-struct-demo.png)

注意，底层数组是可以被多个 slice 同时指向的，因此对一个 slice 的元素进行操作是有可能影响到其他 slice 的。



# slice 的创建

创建 slice 的方式有以下几种：

| 序号 | 方式               | 代码示例                                             |
| ---- | ------------------ | ---------------------------------------------------- |
| 1    | 直接声明           | `var slice []int`                                    |
| 2    | new                | `slice := *new([]int)`                               |
| 3    | 字面量             | `slice := []int{1,2,3,4,5}`                          |
| 4    | make               | `slice := make([]int, 5, 10)`                        |
| 5    | 从切片或数组“截取” | `slice := array[1:5]` 或 `slice := sourceSlice[1:5]` |

#### 直接声明与new

第一种和第二种创建出来的 slice 其实是一个 `nil slice`。它的长度和容量都为0。和`nil`比较的结果为`true`。

这里比较混淆的是`empty slice`，它的长度和容量也都为0，但是所有的空切片的数据指针都指向同一个地址 `0xc42003bda0`。空切片和 `nil` 比较的结果为`false`。

它们的内部结构如下图：

![img](https://raw.githubusercontent.com/elvisNg/elvisng.github.io/master/img/in-post/post-golang-slice/nil-empty-slice.png)

| 创建方式      | nil切片              | 空切片                  |
| ------------- | -------------------- | ----------------------- |
| 方式一        | var s1 []int         | var s2 = []int{}        |
| 方式二        | var s4 = *new([]int) | var s3 = make([]int, 0) |
| 长度          | 0                    | 0                       |
| 容量          | 0                    | 0                       |
| 和 `nil` 比较 | `true`               | `false`                 |

`nil` 切片和空切片很相似，长度和容量都是0，官方建议尽量使用 `nil` 切片。

关于`nil slice`和`empty slice`的探索[深度解析 Go 语言中「切片」的三种特殊状态](https://juejin.im/post/5bea58df6fb9a049f153bca8)



#### 字面量和make

```go
func main(){
  slice := make([]int,4,6)
}
```

![image-20210513180551250](https://raw.githubusercontent.com/elvisNg/elvisng.github.io/master/img/in-post/post-golang-slice/make-slice.png)

```go
func main(){
  slice := []int{10,20,30,40,50,60}
}
```

![image-20210513214133148](https://raw.githubusercontent.com/elvisNg/elvisng.github.io/master/img/in-post/post-golang-slice/slice-array-demo.png)

```go
func main(){
  array := [6]intt{10,20,30,40,50,60}
  sliceA := array[2:5:5]
  sliceB := array[1:3:5]
}
```

![image-20210513214310317](https://raw.githubusercontent.com/elvisNg/elvisng.github.io/master/img/in-post/post-golang-slice/slice-array-demo2.png)



那么底层是怎么创建slice的呢？Go v1.16.2 /src/runtime/slice.go

```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
  // 比较切片的容量，容量值域应该在[len,maxAlloc]之间
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		//计算go内存中的class_to_size =======>如果不清楚什么class_to_size请回去看go内存那一篇文章
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
    //比较切片的长度，长度值域应该在[0,maxAlloc]之间
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}
	//申请内存并返回申请好的内存地址
  ===>注意：
  // 分配内存 
  // 小对象从当前P 的cache中空闲数据中分配
  // 大的对象 (size > 32KB) 直接从heap中分配  所以在使用切片的时候都要注意内存分配
	return mallocgc(mem, et, true)
}
```

#### 截取

```go
func main(){
  array := []intt{10,20,30,40,50,60}
  sliceA := array[2:5:5]
}
```

截取也是比较常见的一种创建 slice 的方法，可以从数组或者 slice 直接截取，当然需要指定起止索引位置。

基于已有 slice 创建新 slice 对象，被称为 `reslice`。新 slice 和老 slice 共用底层数组，新老 slice 对底层数组的更改都会影响到彼此。

基于数组创建的新 slice 对象也是同样的效果：对数组或 slice 元素作的更改都会影响到彼此。

值得注意的是，新老 slice 或者新 slice 老数组互相影响的前提是两者共用底层数组，如果因为执行 `append` 操作使得新 slice 底层数组扩容，移动到了新的位置，两者就不会相互影响了。所以，**问题的关键在于两者是否会共用底层数组**。



### slice的append操作

#### 什么是append

先来看看 `append` 函数的原型：

```golang
func append(slice []Type, elems ...Type) []Type
```

append 函数的参数长度可变，因此可以追加多个值到 slice 中，还可以用 `...` 传入 slice，直接追加一个切片。

```golang
slice = append(slice, elem1, elem2)
slice = append(slice, anotherSlice...)
```

`append`函数返回值是一个新的slice，Go编译器不允许调用了 append 函数后不使用返回值。

```golang
append(slice, elem1, elem2)
append(slice, anotherSlice...)
```

所以上面的用法是错的，不能编译通过。



#### append的扩容

网上的说法：

> **当原 slice 容量小于 `1024` 的时候，新 slice 容量变成原来的 `2` 倍；原 slice 容量超过 `1024`，新 slice 容量变成原来的`1.25`倍。**

这里先说结论是片面的。

举个例子：

```go
package main

import "fmt"

func main() {
	s := []int{1,2}
	s = append(s,4,5,6)
	fmt.Printf("len=%d, cap=%d",len(s),cap(s))
}

//输出：len=5, cap=6

```

两个例子中 `s` 原来只有 2 个元素，`len` 和 `cap` 都为 2，`append` 了[4,5,6]三个元素后的输出cap是6，那到底append的扩容逻辑是怎么样的呢

我们来仔细看看源码，为什么会这样：

```go
func growslice(et *_type, old slice, cap int) slice {
  //传入的cap现在是5
  ...
	
	newcap := old.cap
  //doublecap是4
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
    ...
			}
		}
	}
//newcap 是5

	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// Specialize for common values of et.size.
	// For 1 we don't need any division/multiplication.
	// For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
	// For powers of 2, use a variable shift.
	switch {
	case et.size == 1:
		...
	case et.size == sys.PtrSize:
    //运行在64位系统上ptrSize是8,一个指针的大小,分配的话还是可以看回内存那一篇文章，文末有链接
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
  	//计算内存空间
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)          <=====重点
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
    //内存对齐
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
	...
	default:
		...
	}
  ...
//一些校验
	//内存拷贝新旧切片
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}
```

计算内存

```go
//可以知道上面传入的size是5*8=40
//const _MaxSmallSize = 32768
//const smallSizeMax = 1024
//const smallSizeDiv = 8
func roundupsize(size uintptr) uintptr {
	if size < _MaxSmallSize {
		if size <= smallSizeMax-8 {
      //走到这里
      //divRoundUp(size, smallSizeDiv) =5
      //获取 size_to_class8 数组中索引为 5 的元素为 4
      //获取 class_to_size 中索引为 4 的元素为 48
			return uintptr(class_to_size[size_to_class8[divRoundUp(size, smallSizeDiv)]])
		} else {
			return uintptr(class_to_size[size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]])
		}
	}
	if size+_PageSize < size {
		return size
	}
	return alignUp(size, _PageSize)
}

```



所以最后内存对齐的结果`int(capmem / sys.PtrSize)`=48/8=6 ，最后newcap扩容大小为6。

这里需要用golang内存分配的知识，建议看回[go内存分配](https://elvisng.github.io/2020/11/27/golang-memory/)

##### Append小结：

- 扩容策略并不是简单的扩为原切片容量的 `2` 倍或 `1.25` 倍，还有内存对齐的操作。扩容后的容量 >=  `1.25` 倍。
- 不是单单网上所说的按1024跨分，大概go设计上是想节省内存空间吧



#### slice是值传递还是指针传递

先给结论：值传递

> Go 语言的函数参数传递，只有值传递，没有引用传递。　

后面会对此再写一篇文章说明



我们来看一段程序：

```go
package main

import "fmt"

func main() {
  slice := make([]int, 0, 10)
  slice = append(slice, 1)
  fmt.Println(slice, len(slice), cap(slice))
  fn(slice)
  fmt.Println(slice, len(slice), cap(slice))
}
func fn(in []int) {
  in = append(in, 5)
}

//输出：
//[1] 1 10
//[1] 1 10
```

可见fn内的append操作并未对slice产生影响，那我们再看一段代码：

```go
package main

import "fmt"

func main() {
  slice := make([]int, 0, 10)
  slice = append(slice, 1)
  fmt.Println(slice, len(slice), cap(slice))
  fn(slice)
  fmt.Println(slice, len(slice), cap(slice))
}
func fn(in []int) {
  in[0] = 100
}

//输出：
//[1] 1 10
//[100] 1 10
```

slice居然改变了，是不是有点混乱？前面我们说到slice底层其实是一个结构体，len、cap、array分别表示长度、容量、底层数组的地址，当slice作为函数的参数传递的时候，跟普通结构体的传递是没有区别的；如果直接传slice，实参slice是不会被函数中的操作改变的，但是如果传递的是slice的指针，是会改变原来的slice的；

另外，无论是传递slice还是slice的指针，如果改变了slice的底层数组，那么都是会影响slice的，这种通过数组下标的方式更新slice数据，是会对底层数组进行改变的，所以就会影响slice。

简单来说不是整体的slice是指针传递，而是结构体内部的底层数据是一个指针地址。



那么，讲到这里，在第一段程序中在fn函数内append的5到哪里去了，不可能凭空消失啊，我们再来看一段程序：

```go
package main

import "fmt"

func main() {
  slice := make([]int, 0, 10)
  slice = append(slice, 1)
  fmt.Println(slice, len(slice), cap(slice))
  fn(slice)
  fmt.Println(slice, len(slice), cap(slice))
  s1 := slice[0:9]//数组截取
  fmt.Println(s1, len(s1), cap(s1))
}
func fn(in []int) {
  in = append(in, 5)
}

//输出：
[1] 1 10
[1] 1 10
[1 5 0 0 0 0 0 0 0] 9 10
```



显然，虽然在append后，slice中并未展示出5，也无法通过slice[1]取到（会数组越界）,但是实际上底层数组已经有了5这个元素，但是由于slice的len未发生改变，所以我们在上层是无法获取到5这个元素的。那么，再问一个问题，我们是不是可以手动强制改变slice的len长度，让我们可以获取到5这个元素呢？是可以的，我们来看一段程序：

```go
package main

import (
  "fmt"
  "reflect"
  "unsafe"
)

func main() {
  slice := make([]int, 0, 10)
  slice = append(slice, 1)
  fmt.Println(slice, len(slice), cap(slice))
  fn(slice)
  fmt.Println(slice, len(slice), cap(slice))
  (*reflect.SliceHeader)(unsafe.Pointer(&slice)).Len = 2 //强制修改slice长度 这里用了unsafe.Pointer做强转，一般不太建议用go这种hacktech
  fmt.Println(slice, len(slice), cap(slice))
}

func fn(in []int) {
  in = append(in, 5)
}
//输出：
[1] 1 10
[1] 1 10
[1 5] 2 10
```

可以看出，通过强制修改slice的len，我们可以获取到了5这个元素。

所以再次回答一开始我们提出的问题，slice是值传递还是引用传递？答案是值传递！



### 总结

到此，关于 `slice` 的部分就讲完了，总结一下：

- 切片是对底层数组的一个抽象，描述了它的一个片段。
- 切片实际上是一个结构体，它有三个字段：长度，容量，底层数据的地址。
- 多个切片可能共享同一个底层数组，这种情况下，对其中一个切片或者底层数组的更改，会影响到其他切片。
- `append` 函数会在切片容量不够的情况下，调用 `growslice` 函数获取所需要的内存，这称为扩容，扩容会改变元素原来的位置。
- 扩容策略并不是简单的扩为原切片容量的 `2` 倍或 `1.25` 倍，还有内存对齐的操作。扩容后的容量 >= 原容量的 `2` 倍或 `1.25` 倍。
- 当直接用切片作为函数参数时，可以改变切片的元素，不能改变切片本身；想要改变切片本身，可以将改变后的切片返回，**（注意这样会造成内存逃逸，GC回收压力要考虑。**）函数调用者接收改变后的切片或者将切片指针作为函数参数。如：func add(* []int)