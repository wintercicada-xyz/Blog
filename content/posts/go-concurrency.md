---
title: "Go 并发"
date: 2022-03-20T17:07:09+08:00
tags: ["Golang", "多线程"]
categories: ["编程"]
draft: false
---

Go 作为一个相对较新的语言，能够被许多人接受并大量应用于公司项目中，说明它在某些方面是优于传统的 C++ 与 Java 的，能吸引开发者去应用并学习 Go 语言。而我认为其中最能吸引开发者的特性就是易上手的并发。

<!--more-->
并发是 Go 语言中的一等公民，内置于语言中而不是像许多其他语言一样通过外部库实现。这就使得在 Go 中可以用一些特定的关键字与操作符来实现并发，更为简洁优雅，也便于理解学习。现在多核处理器大量普及，且后端服务要处理的任务一般具有数量大，相互独立的特征，这种情况下相较于优化单线程的算法，应用并发能以更低的成本获得更好的性能。

<br />

## 前置知识
### 并发与并行
并发（Concurrency）是**表面上**看起来同时执行多个任务，注意这里是**表面上**，这可以通过在同一个处理单元上快速在任务间切换来实现，也可以通过分配多个处理单元来同时执行多个任务实现。而并行（Parallelism）是实际上同时执行多个任务，一定要有与任务数相匹配的处理单元来一一对应地处理这些任务，相对于并发来说要求更苛刻。但是我们在编程时一般只会考虑并发，并行是由系统决定的，如果有充足的处理单元，程序就并行运行，否则程序并发运行。

### 进程与线程
上面讲到了系统分配的处理单元会影响程序是并发执行还是并行执行，而系统是如何分配的呢？就是通过进程与线程。进程（Process）是指一个分配给程序的内存块以及若干线程。线程（Thread）从属于进程，并共享线程的内存，且线程是计算资源分配的最小单元，即 CPU 有空闲核心时，它会以线程为单位去处理要计算的任务。

### Go 中的并发实现
Go 中的并发并不是线程实现的，而是通过 Goroutine —— 一个 Go 的协程（Coroutine）实现的。协程可以被看作是轻量级的线程，由编程语言本身实现，而区别于线程是由系统实现的。一般我们实现多线程之间的通信都是通过共享内存，但是比较繁琐不直观，容易在调用数据时忘了上锁解锁而造成难以排查的 Bug。而 Goroutine 支持通过 Channel 来直接进行线程间通信，这样同步更为直观不易出错。

<br />

## 并发语法
### Goroutine
Go 中的并发非常简单，要另起一个 Goroutine 来执行一个函数只需在调用函数前加上 `go` 就可以了。
```Golang
func work() {
    // ...Do something
}

func main() {
    // starts a new goroutine running work()
    go work()
}
```

### Channel
在 Go 中，Channel 的类型为 `chan <Channel 传输的数据类型>`，使用 `make` 函数来创建：
```Golang
channel := make(chan int)
```

还可以创建具有缓存的 Channel：
```Golang
bufferChannel := make(chan int, 10)
```

Channel 也是有长度的，可以通过 `len` 函数获取。对于没有缓存的 Channel 其长度永远为 0，而对于有缓存的 Channel，其长度为缓存中的数据量。

而向 Channel 中发送信息与接收信息也非常直观：
```Golang
channel := make(chan int)
go func() {
    // 向 Channel 发送数据
    channel <- 10
}
go func() {
    // 从 Channel 中取出数据
    get := <- channel
}
```
注意这两个操作都是会阻塞的。当 Channel 内数据缓存用完没及时取出数据时，向其中发送数据会阻塞，当 Channel 中没有数据时，取出数据也会阻塞。这个问题可用通过使用具有缓存的 Channel 部分缓解。

如果我们要不断取出同一个 Channel 中的数据，可以应用到 `for` 语句：
```Golang
for num := range channel {
    fmt.Println(nums)
}
```
那么何时结束循环呢？可以通过 `close` 函数关闭 Channel， 这样循环就会停止了。
```Golang
close(channel)
```

对于一个关闭的 Channel，它有以下特性：
* 向其发送数据时会引发 `panic`
* 取出数据时不再阻塞，而是一直取出对应数据类型的零值。
* `num, ok := <-channel` 取出数据得到的第二个返回值 `ok` 为 `false`，不同于 Channel 未关闭情况下为 `true`

如果我们同时有若干 Channel，要做发送数据或是取出数据的处理，如果一个阻塞了，其他的就算是没有阻塞也要等它，这明显不是我们想要的。我们想要的是去处理不阻塞的 Channel，如果全都阻塞时才阻塞，这时就可以用到 `select` 语句来实现：
```Golang
select {
    // 给 ch1 发送数据
    case ch1 <- 1:
        fmt.Println("Send 1 to ch1")

    // 取出 ch2 中的数据并放到 x 中
    case x := <-ch2
        fmt.Printf("Receive %v from ch2", x)

    // 取出 ch3 中的数据
    case <-ch3:
        fmt.Println("Receive from ch3")
}
```

但是这个语句还是有可能阻塞，如果三个 Channel 都阻塞了，它就会阻塞。我们有可能想着在这种情况时又去做一些无关 Channel，一定不会阻塞的事情就可以加上 `default` 来实现。我们还可以在这个 `select` 语句外套上 `for` 语句，这样就可以实现有 Channel 不阻塞时处理对应的 Channel，不然的话就循环执行 `default` 语句。这样可以实现一些很有趣的内容，例如 *A Tour of Go* 中的[默认选择](https://tour.go-zh.org/concurrency/6)。

还有一个使用 Channel 的小技巧，这是在 *A Tour of Go* 里没有提及的。可以在函数参数或是返回数据类型处使用 `<-chan <数据类型>` 来限定对应的 Channel 只能用于接收数据，使用 `chan<- <数据类型>` 来限定对应的 Channel 只能用于发送数据。下面是个例子：
```Golang
func relay(ch1 <-chan int, ch2 chan<- int) {
    // 在该函数内的 ch1 只可用于接收数据
    a := <-ch1

    // ch2 只可以用于发送数据
    ch2 <- a

    // 下面的注释取消后会报错
    // ch1 <- 1
    // b := <-ch2
}
```

<br />

## 参考
* [Go 并发](https://go.dev/tour/concurrency/1)
* [Go 取出操作符](https://go.dev/ref/spec#Receive_operator)
* [Go Channel 类型](https://go.dev/ref/spec#Channel_types)