---
title: "Go 并发实战"
date: 2022-03-22T11:09:00+08:00
tags: ["Golang", "多线程"]
categories: ["编程"]
draft: false
---

Go 语言中推荐在使用并发时使用 Channel 通信而不是共享内存方式来实现各个 Goroutine 相互之间的沟通，这使得它的并发编程会不同于大多数用共享内存方式来实现线程间通信的编程语言。下面将会使用一个驱动案例来介绍 Go 的并发编程常用的一些模式。如果你还没有了解过 Go 的并发，可以读一下[这篇文章](https://blog.wintercicada.xyz/post/go-concurrency/)。

<!--more-->
<br />

## 工厂 v0.0
想象一个这样的场景：有一个工厂，里面有若干个工人。工厂不断有订单流入，工人完成这些订单后就把产品运出去。让我们来实现这个场景：
```Golang
// 产品
type Product struct{}

// 订单
type Order struct{}

func (order Order) Process() Product {
	// 假装处理订单
	time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
	return Product{}
}

// 工人
type Worker struct{}

func (worker Worker) Work(order Order) Product {
	return order.Process()
}

// 工厂
type Factory struct {
	workers []Worker
}
```

可以看到我们这里是没有订单输入以及产品输出的流程的，我们可以用 Channel 来模拟这两个流程：
```Golang
// 改动 Factory 结构体
type Factory struct {
	orderIn    <-chan Order
	productOut chan<- Product
	workers    []Worker
}

func main() {
    // 初始化订单输入与产品输出的 Channel
    orderIn := make(chan Order, 10)
    productOut := make(chan Product, 10)
	factory := Factory {
		orderIn,
		productOut,
		[]Worker{Worker{}, Worker{}, Worker{}},
	}
    factory.Work()
	for {
		select{}
	}
}
```
写完后你会发现 `factory.Work()` 会报错，因为我们还没有实现这个方法。先用串行的方法实现一个初始版本：
```Golang
func (factory Factory) Work() {
	for order := range factory.orderIn {
		factory.productOut <- factory.workers[0].Work(order)
	}
}
```
这个初始版本非常简陋，我们姑且将其称为*工厂 v0.0*。

<br />

## 工厂 v1.0
上面的*工厂 v0.0* 有两个问题：
1. 必须要前一个订单完成了才会去接下一个订单
2. 编号为 0 的工人要累死了，自己一个人干，其他的工人都看着他

那么我们先用并发来解决第一个问题。作为一个自律的工人，怎么能等工厂来分配订单呢？当然是要自己去接订单再自己把成品送出去啦。这样的话就可以解决第一个问题：
```Golang
// 自律工人的工作方式
func (worker Worker) Work(orderIn <-chan Order, productOut chan<- Product) {
    // 工人是独立工作的，当接到订单后就各自干各的，所以要新开一个 Goroutine
    go func() {
        for order := range orderIn {
            productOut <- order.Process()
        }
    }()
}

// 拥有一个自律工人的先进工厂
func (factory Factory) Work() {
	Worker[0].Work(factory.orderIn, factory.productOut)
}
```


工厂老板看到了自律工人带来的效率提升，于是乎把所有工人都换成自律工人，再也没有偷懒的工人在一旁看其他人工作了。那么问题二也解决了：
```Golang
// 拥有许多自律工人的先进工厂
func (factory Factory) Work() {
    for _, worker := range factory.workers {
		worker.Work(factory.orderIn, factory.productOut)
	}
}
```
完成了这些改造之后，工厂效率大大提升，可以称之为*工厂 v1.0*了。

<br />

## 工厂 v2.0
*工厂 v1.0* 只考虑到了工厂的利益，没有考虑工人的利益。工人相当于和工厂签了卖身契，要一直为这一个工厂工作，不能转工厂，更不能打两个工厂的工。于是工人们决定起义，将获取订单与交付成本的通道掌握在自己的手里：
```Golang
// 自由工人
type Worker struct {
	orderIn    chan Order
	productOut chan Product
}

func NewWorker() Worker {
	orderIn := make(chan Order)
	productOut := make(chan Product)
	go func() {
		for order := range orderIn {
			productOut <- order.Process()
		}
	}()
	return Worker{orderIn, productOut}
}
```

工人们转变了，工厂也只有适应他们才能生存。每个工人自己掌控自己的订单接受和成品交付通道，工厂就要想办法将自己的订单分配给工人们，并收集工人们生产的产品。于是工厂也进行了改造，加入了任务分配与成品收集两个功能：
```Golang
// 工厂
type Factory struct {
	orderIn    <-chan Order
	productOut chan<- Product
	workers    []Worker
}

func (factory Factory) Work() {
	for _, worker := range factory.workers {
		// 将订单分配给工人
		go func(orderIn chan<- Order) {
			for order := range factory.orderIn {
				orderIn <- order
			}
		}(worker.orderIn)

		// 收集工人生产的产品
		go func(productOut <-chan Product) {
			for product := range productOut {
				factory.productOut <- product
			}
		}(worker.productOut)
	}
}
```
最终工人们争取到了自己的权利，工厂也复工了。这个我们称之为 *工厂 v2.0*。

<br />

## 工厂 v3.0
经过了多次改造，工厂也是非常成功，接到了许多订单。但是有一家公司，它给工厂下了大量的订单，却在工厂交付成品之前因经营不善倒闭了。工厂没有遇到过这样的事，没能及时通知工人，工人都把产品生产出来了，但是这些产品没人要，造成了工厂的亏损。于是工厂决定建立一个专门应对上述情况的机制，可以紧急停止生产，避免亏损。

一开始，工厂领导方想通过新建一个类似于下发订单的渠道（Channel），让工人们能够在接受到这个渠道的消息后停止工作。这个方案执行后工人改动如下：
```Golang
func NewWorker(stop <-chan struct{}) Worker {
	orderIn := make(chan Order)
	productOut := make(chan Product)
	go func() {
		for {
			select {
			case <-stop:
				return
			case order := <-orderIn:
				productOut <- order.Process()
			}
		}
	}()
	return Worker{orderIn, productOut, stop}
}
```

工厂也加入了紧急停止渠道 `stop`， 这里的 `stop` 和下属工人的 `stop` 是同一个。
```Golang
// 工厂
type Factory struct {
	orderIn    <-chan Order
	productOut chan<- Product
	stop       chan struct{}
	workers    []Worker
}
```

然后提供工厂的紧急停止执行入口：
```Golang
func (factory Factory) Stop() {
	factory.stop <- struct{}{}
}
```
但是这里有个问题，我们的渠道的消息只能由一个工人接受，其他的工人就接受不到了。当然我们可以按工人个数发送相应个数的消息，但是工人的数量可能是动态调整的，这样的话就不好管理了。

这里正确的用法是关闭这个渠道：
```Golang
func (factory Factory) Stop() {
	close(factory.stop)
}
```
还记得在之前那篇介绍 Golang 并发的文章中关于关闭 Channel 的描述吗？如果 Channel 被关闭了，对于接受方而言，这就是个固定为 Channel 对应数据类型零值的变量。这样对于所有工人来说，它都是一个立刻可读的变量，不会堵塞，所有工人都能立马接受到停止工作的信号。这就是 *工厂 v3.0*。

<br />

## 总结
以上各个版本的工厂涉及了许多 Golang 的并发模式，如将并发执行的内容封装为函数，通过 Channel 来输入参数及输出结果；Channel 的一对多，多对一通信；以及通过关闭 Channel 来给并发执行中的任务发送中止信号。各个版本的工厂完整代码可以在[我的 Github](https://github.com/wintercicada-xyz/go-concurrency) 上找到。这些代码都是为了方便解释 Golang 的并发模式，没有经过测试。如有错漏，欢迎指出。

<br />

## 参考
[Golang 并发模式](https://go.dev/blog/pipelines)