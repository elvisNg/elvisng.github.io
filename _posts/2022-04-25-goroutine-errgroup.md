---
layout: post
title: "GoRoutine-Errgroup"
subtitle: 'Errgroup'
author: "Elvis"
header-style: text
mermaid: true
tags:
  - Golang
---



## ErrGroup

如果您不需要对错误进行任何进一步的处理，可以尝试使用[ErrGroup](https://pkg.go.dev/golang.org/x/sync/errgroup)！

ErrGroup 本质上是一个包装好的[ sync.WaitGroup](https://pkg.go.dev/sync#WaitGroup)，用于从启动的 goroutine 中捕获错误。

### WaitGroup

这是一个普通使用 WaitGroup 没有错误的示例（来自 godoc,并加了一些改造）：

```go
package main

import (
	"sync"
)

type httpPkg struct{}

func (httpPkg) Get(url string) {}

var http httpPkg

func main() {
	var wg sync.WaitGroup
	var urls = []string{
		"http://www.golang.org/",
		"http://www.google.com/",
		"http://www.somename.com/",
	}
  errs := make (error,3)
  defer close(errs)
	for _, url := range urls {
		// Increment the WaitGroup counter.
		wg.Add(1)
		// Launch a goroutine to fetch the URL.
		go func(url string) {
			// Decrement the counter when the goroutine completes.
			defer wg.Done()
			// Fetch the URL.
      if _,err := http.Get(url);err!=nil{
        errs <- err
      }
		}(url)
	}
	// Wait for all HTTP fetches to complete.
	wg.Wait()
  fmt.Println("Successfully fetched all URLs.")
}
```

要使用 WaitGroup，首先创建group：

```go
var wg sync.WaitGroup
```

接下来，对于每个 goroutine，将该数字添加到group中：

```go
wg.Add(1)
```

然后每当一个 goroutine 完成时，告诉group：

```go
defer wg.Done()
```

它延迟关键字后面的语句的执行，直到周围的函数返回。

最后，等待group完成：

```go
wg.Wait()
```

在这种情况下，不会发生任何错误。让我们看看如果我们需要使用 ErrGroup 来捕获错误，它会如何变化。

## WaitGroup

这是与上面相同的示例，但使用了 ErrGroup（来自 godoc）：

```go
package main

import (
	"fmt"
	"net/http"

	"golang.org/x/sync/errgroup"
)

func main() {
	g := new(errgroup.Group)
	var urls = []string{
		"http://www.golang.org/",
		"http://www.google.com/",
		"http://www.somename.com/",
	}
	for _, url := range urls {
		// Launch a goroutine to fetch the URL.
		url := url // https://golang.org/doc/faq#closures_and_goroutines
		g.Go(func() error {
			// Fetch the URL.
			resp, err := http.Get(url)
			if err == nil {
				resp.Body.Close()
			}
			return err
		})
	}
	// Wait for all HTTP fetches to complete.
	if err := g.Wait(); err == nil {
		fmt.Println("Successfully fetched all URLs.")
	}
}
```

它看起来非常相似，以下是不同之处：

首先，创建组：

```go
var wg sync.WaitGroup

// VVV BECOMES VVV

g := new(errgroup.Group)
```

接下来，不是将每个 goroutine 添加到group中，而是将`g.Go`函数调用为 goroutine。唯一的要求是它必须具有以下签名：`func() error`. 此外，由于 ErrGroup 将在 goroutine 完成时进行处理，因此无需调用`wg.Done()`.

```go
go func(arg string) {
			// Decrement the counter when the goroutine completes.
			defer wg.Done()
			// ... work that can return error here
}(arg)

// VVV BECOMES VVV

g.Go(func() error {
  // ... work that can return error here
})
```

最后，等待组完成并根据需要处理错误：

```go
wg.Wait()

// VVV BECOMES VVV

if err := g.Wait(); err == nil {
		fmt.Println("Successfully fetched all URLs.")
}
```

ErrGroups 为处理 goroutine 中的错误提供了很多机会。话虽如此，ErrGroup 只是工具箱中的另一个工具，应在适合用例时使用。如果需要根据错误做出一些更复杂的决策和工作，那么channel可能更适合。

当然ErrGroups还有提供可以自定义的cancel()函数对于error去做catch处理，详细的就先不展开了。

打完工，还挺累的，下次再叨。