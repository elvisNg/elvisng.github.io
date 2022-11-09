---
layout: post
title: "Go-interface"
subtitle: 'Go接口'
author: "Elvis"
header-style: text
mermaid: true
tags:
  - Golang
---



### 什么是Interface?



在 Golang 中，interface 是一种抽象类型，相对于抽象类型的是具体类型（concrete type）：int，string。如下是 io 包里面的例子。

```go

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}
```

在 Golang 中，interface 是一组 method 的集合，是 duck-type programming 的一种体现。不关心属性（数据），只关心行为（方法）。具体使用中你可以自定义自己的 struct，并提供特定的 interface 里面的 method 就可以把它当成 interface 来使用。下面是一种 interface 的典型用法，定义函数的时候参数定义成 interface，调用函数的时候就可以做到非常的灵活。

```go
type MyInterface interface{
    Print()
}

func TestFunc(x MyInterface) {}
type MyStruct struct {}
func (me MyStruct) Print() {}

func main() {
    var me MyStruct
    TestFunc(me)
}
```



## 为什么要用Interface

Gopher China 上给出了下面上个理由：

- writing generic algorithm （泛型编程）
- hiding implementation detail （隐藏具体实现）
- providing interception points （拦截器锚点）



#### 泛型编程

> 其实有了解开源的都知道泛型的脚步可越来越近了，大概今年的1.17版本就可以试用了

严格来说，在 Golang 中并不支持泛型编程。在 C++ 等高级语言中使用泛型编程非常的简单，所以泛型编程一直是 Golang 诟病最多的地方。但是使用 interface 我们可以实现泛型编程，比如我们现在要写一个泛型算法，形参定义采用 interface 就可以了，以标准库的 sort 为例。

```go
package sort

// A type, typically a collection, that satisfies sort.Interface can be
// sorted by the routines in this package.  The methods require that the
// elements of the collection be enumerated by an integer index.
type Interface interface {
    // Len is the number of elements in the collection.
    Len() int
    // Less reports whether the element with
    // index i should sort before the element with index j.
    Less(i, j int) bool
    // Swap swaps the elements with indexes i and j.
    Swap(i, j int)
}

...

// Sort sorts data.
// It makes one call to data.Len to determine n, and O(n*log(n)) calls to
// data.Less and data.Swap. The sort is not guaranteed to be stable.
func Sort(data Interface) {
    // Switch to heapsort if depth of 2*ceil(lg(n+1)) is reached.
    n := data.Len()
    maxDepth := 0
    for i := n; i > 0; i >>= 1 {
        maxDepth++
    }
    maxDepth *= 2
    quickSort(data, 0, n, maxDepth)
}
```

Sort 函数的形参是一个 interface，包含了三个方法：`Len()`，`Less(i,j int)`，`Swap(i, j int)`。使用的时候不管数组的元素类型是什么类型（int, float, string…），只要我们实现了这三个方法就可以使用 Sort 函数，这样就实现了“泛型编程”。有一点比较麻烦的是，我们需要将数组自定义一下。下面是一个例子。

```go
type Person struct {
    Name string
    Age  int
}

func (p Person) String() string {
    return fmt.Sprintf("%s: %d", p.Name, p.Age)
}

// ByAge implements sort.Interface for []Person based on
// the Age field.
type ByAge []Person //自定义

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }

func main() {
    people := []Person{
        {"Bob", 31},
        {"John", 42},
        {"Michael", 17},
        {"Jenny", 26},
    }

    fmt.Println(people)
    sort.Sort(ByAge(people))
    fmt.Println(people)
}

//输出：
[Bob: 31 John: 42 Michael: 17 Jenny: 26]
[Michael: 17 Jenny: 26 Bob: 31 John: 42]

```

在我们设计函数的时候，下面是一个比较好的准则。

> Be ***conservative\*** in what you send, be ***liberal\*** in what you accept. — Robustness Principle

对应到 Golang 就是：

> Return ***concrete types\***, receive ***interfaces\*** as parameter. — Robustness Principle applied to Go

话说这么说，但是当我们翻阅 Golang 源码的时候，有些函数的返回值也是 interface。



#### 隐藏具体实现

隐藏具体实现，这个很好理解。比如我设计一个函数给你返回一个 interface，那么你只能通过 interface 里面的方法来做一些操作，但是内部的具体实现是完全不知道的。

例如我自己封装的微服务框架[broccoliv2](https://github.com/elvisNg/broccoliv2)在自定义prometheus的exporter

```go

type promClient interface {
	State(name string, v int64, extra ...string)
  ...
}
//内部的prometheus用来收集内部db的sql累加
type innerProm struct {
	timer   *prometheus.HistogramVec
	counter *prometheus.CounterVec
	state   *prometheus.GaugeVec
}

//外部的prometheus用来收集内部db的sql累加
type pubProm struct {
	timer   *prometheus.HistogramVec
	counter *prometheus.CounterVec
	state   *prometheus.GaugeVec
}

// Incr increments one stat gauge without sampling
func (p *pubProm) StateIncr(name string, extra ...string) {
	if p == nil {
		return
	}
	label := append([]string{name}, extra...)
	if p.state != nil {
		p.state.WithLabelValues(label...).Inc()
	}
}


//专门对于内部DB做的StateIncr封装函数
// Incr increments one db stat gauge without sampling
func (p *innerProm) StateIncr(name string, extra ...string) {
	if p == nil {
		return
	}
  //table name 和sql 需要用特殊字符切分成两段label进行incr
	label := append([]string{name}, extra...)
	if p.state != nil {
		p.state.WithLabelValues(label...).Inc()
	}
}


//外部只需要按需获取不同的PubClient，不需要关注PubClient实现
type Prometheus interface {
	GetPubCli() *promClient
	GetInnerCli() *promClient
	Enable()
	Disable()
}

```



尽管上面的两个函数`GetPubCliI()`和`GetInnerCli()`都返回了promClient对象，但他们对于**StateIncr()**的实现不同，对于外层框架消费者来说，无需关注其StateIncr的实现，只需要按文档的要求按需来获取不同的promClient做StateIncr即可，可以说是对于用户来说完全无感知的。



#### 拦截器锚点

Francesc 这里的 interception 想表达的意思我理解应该是 wrapper 或者装饰器，给一个例子如下：

```go
type header struct {
    rt  http.RoundTripper
    v   map[string]string
}

func (h header) RoundTrip(r *http.Request) *http.Response {
    for k, v := range h.v {
        r.Header.Set(k,v)
    }
    return h.rt.RoundTrip(r)
}
```

通过 interface，我们可以通过类似这种方式实现 dynamic dispatch。

当然也可以看[broccoliv2的mysql的Interceptor实现](https://github.com/elvisNg/broccoliv2/blob/main/mysql/client.go#L112)，比较复杂这里就不展开了。

### 非侵入式

Francesc 还提到 interface 的非侵入式特性。什么是侵入式呢？比如 Java 的 interface 实现需要显示的声明。

```
public class MyWriter implements io.Writer {}
```

这样就意味着如果要实现多个 interface 需要显示地写很多遍，同时 package 的依赖还需要进行管理。Dependency is evil。比如我要实现 io 包里面的 Reader，Writer，ReadWriter 接口，代码可以像下面这样写。

```go
type MyIO struct {}

func (io *MyIO) Read(p []byte) (n int, err error) {...}
func (io *MyIO) Write(p []byte) (n int, err error) {...}

// io package
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadWriter interface {
    Reader
    Writer
}
```

这种写法真的很方便，而且不用去显示的 import io package，interface 底层实现的时候会动态的检测。这样也会引入一些问题：

1. 性能下降。使用 interface 作为函数参数，runtime 的时候会动态的确定行为。而使用 struct 作为参数，编译期间就可以确定了。
2. 不知道 struct 实现哪些 interface。这个问题可以使用 guru 工具来解决。

综上，Golang interface 的这种非侵入实现真的很难说它是好，还是坏。但是可以肯定的一点是，对开发人员来说代码写起来更简单了。

### Interface的断言

我们常用的interface 是用来做入参，这是就需要其他类型转换；这里就会产生断言。

```
func do(v interface{}) {
    n := v.(int)    // might panic
}
```

这样写的坏处在于：一旦断言失败，程序将会 panic。一种避免 panic 的写法是使用 type assertion。

```
func do(v interface{}) {
    n, ok := v.(int)
    if !ok {
        // 断言失败处理
    }
}
```

对于 interface 的操作可以使用 reflect 包来处理。

> 后面会对于reflect写一篇来做介绍。



### 总结

* interface 是 Golang 的一种重要的特性，但是这是以 runtime 为代价的，也就意味着性能的损失（关于 interface 的底层实现之后有时间再写）。
* 抛开性能不谈（现实中使用 Golang 开发的程序 99% 性能都不是问题），interface 对于如何用优雅的设计通用的代码才是值得我们思考的方向。
* 如果泛型出来，可以马上去试试吧，毕竟Go的东西都是值得一用看个究竟的。