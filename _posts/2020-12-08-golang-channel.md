---
layout: post
title: "Golang-Routine"
subtitle: 'channel'
author: "Elvis"
header-style: text
tags:
  - golang
  - 技术分享

---

**Go**（又称**Golang**）是[Google](https://baike.baidu.com/item/Google)开发的一种[静态](https://baike.baidu.com/item/静态)[强类型](https://baike.baidu.com/item/强类型)、编译型、并发型，并具有垃圾回收功能的[编程语言](https://baike.baidu.com/item/编程语言)。Go于2009年正式推出，国内各大互联网公司都有使用，尤其是七牛云，基本都是golang写的，
传闻Go是为并发而生的语言，运行速度仅比c c++慢一点，内置协程（轻量级的线程），说白了协程还是运行在一个线程上，由调度器来调度线程该运行哪个协程，也就是类似于模拟了一个操作系统调度线程，我们也知道，***其实多线程说白了也是轮流占用cpu，其实还是顺序执行的\***，协程也是一样，他也是轮流获取执行机会，只不过他获取的是线程，***但是如果cpu是多核的话，多线程就能真正意义上的实现并发\***同时，如果GO执行过程中有多个线程的话，协程也能实现真正意义上的并发执行，所以，***最理想的情况，根据cpu核数开辟对应数量的线程，通过这些线程，来为协程提供执行环境\***
当我们在开发网络应用程序时，遇到的瓶颈总是在io上，由此出现了多进程，多线程，异步io的解决方案，其中异步io最为优秀，因为他们在不占用过多的资源情况下完成高性能io操作，但是异步io会导致一个问题，那就是回调地狱，node js之前深受诟病的地方就在于此，后来出现了async await这种方案，真正的实现了同步式的写异步，其实Go的协程也是这样，有人把goroutine叫做纤程，认为node js的async await才是真正的协程，对此我不做评价，关于goroutine的运行机制本文不讲，大家可以看[这篇](https://www.cnblogs.com/sunsky303/p/9115530.html)博文，讲的很生动，本文主要对goroutine的使用进行讲解，如果大家熟悉node js的async await或者c#的async（其实node js就是学习的c#的async await），可以来对比一下两者在使用上的不同，从而对协程纤程的概念产生进一步的了解
在golang中开辟一个协程非常简单，只需要一个go关键字

```
package main
  
import (
        "fmt"
        "time"
)


func main(){
        for i := 0;i<10;i++{
                go func(i int){
                        for{
                                fmt.Printf("%d",i);
                           }
                }(i)
        }
        time.Sleep(time.Millisecond);
}
```

打印结果

```
5551600088800499999991117777777742222220000044444444888888888999
9666665111177777777777777777777777777333333333333333399999999999
999999999999999999999999999999444442224444444488888888222222222
20888886666666655555555555444011111111111111000000000999999555555
5554444444000077777666666311111197777778888222277777753333444444
9999997777772222000077774444444444444444444
```

可以看到，完全是随机的，打印哪个取决于调度器对协程的调度，
goroutine相比于线程，有个特点，那就是非抢占式，如果一个协程占据了线程，不主动释放或者没有发生阻塞的话，那么永远不会交出线程的控制权，我们举个例子来验证下

```
package main
  
import (
       "time"
)
func main(){
        for i := 0;i<10;i++{
                go func(i int){
                        for{
                                i++                                
                          }
                }(i)
        }
        time.Sleep(time.Millisecond);
}
```

这段程序在执行后，永远不会退出，并且占满了cpu，原因就是goroutine中，一直在执行i++，没有释放，而一直占用线程，当四个线程占满之后，其他的所有goroutine都没有执行的机会了，所以本该一秒钟后就退出的程序一直没有退出，cpu满载再跑，但是为什么前面例子的Printf没有发生这种情况呢？是因为Printf其实是个io操作，io操作会阻塞，阻塞的时候goroutine就会自动的释放出对线程的占有，所以其他的goroutine才有执行的机会，除了io阻塞，golang还提供了一个api，让我们可以手动交出控制权，那就是Gosched()，当我们调用这个方法时，goroutine就会主动释放出对线程的控制权

```
package main
  
import (
       "time"
      "runtime"
)
func main(){
        for i := 0;i<10;i++{
                go func(i int){
                        for{
                                i++;
                                runtime.Gosched();                                
                          }
                }(i)
        }
        time.Sleep(time.Millisecond);
}
```

修改之后，一秒钟之后，代码正常退出
常见的触发goroutine切换，有一下几种情况

```
1、I/O,select

2、channel

3、等待锁

4、函数调用（是一个切换的机会，是否会切换由调度器决定）

5、runtime.Gosched()
```

说完了goroutine的基本用法，接下来我们说一下goroutine之间的通信，Go中通信的理念是“不要通过共享数据来通信，而是通过通信来共享数据“，Go中实现通信主要通过channel，它类似于unix shell中的双向管道，可以接受和发送数据，
我们来看个例子，

```
package main
  
import(
        "fmt"
        "time"
)

func main(){
        c := make(chan int)
        go func(){
           for{
                n := <-c;
                fmt.Printf("%d",n)
              }
        }()

        c <- 1;
        c <- 2;
        time.Sleep(time.Millisecond);


}
```

打印结果为`12`,我们通过make来创建channel类型，并指明存放的数据类型，通过 `<-`来接收和发送数据，`c <- 1`为向channel c发送数据1，`n := <-c;`表示从channel c接收数据，默认情况下，发送数据和接收数据都是阻塞的，这很容易让我们写出同步的代码，因为阻塞，所以会很容易发生goroutine的切换，并且，数据被发送后一定要被接收，不然会一直阻塞下去，程序会报错退出，
***本例中，首先向c发送数据1，main goroutine阻塞，执行开辟的协程，从而读到数据，打印数据，然后main协程阻塞完成，向c发送第二个数据2，开辟的协程还在阻塞读取数据，成功读取到数据2时，打印2，一秒钟后，主函数退出，所有goroutine销毁，程序退出\***









The buffer size is the number of elements that can be sent to the channel without the send blocking. By default, a channel has a buffer size of 0 (you get this with make(chan int)). This means that every single send will block until another goroutine receives from the channel. A channel of buffer size 1 can hold 1 element until sending blocks, so you'd get

 

```go
func main(){

c:=make(chan int)

c<-1*;*

c<-2*; //*fatal error: all goroutines are asleep - deadlock!

//Chan default size is zero,This means that every single send will block unitl another goroutine receives

gofunc(){

	for{

		n:=<-c*;*

		fmt.Printf("%d",n)

	}

}()

time.Sleep(time.Millisecond)

}

Output:

fatal error: all goroutines are asleep - deadlock!


//current
func main() {
	c := make(chan int)
	go func(){
		for{
			time.Sleep(time.Millisecond)
			n := <-c;
			fmt.Printf("%d",n)
		}
	}()
	c <- 1;
	c <- 2;

}

output:
12 
//using chan default size is zero,This means that every single send will block unitl another goroutine receives
```



