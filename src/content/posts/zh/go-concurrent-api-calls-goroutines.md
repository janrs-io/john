---
title: "用GoRoutines高性能同时进行多个Api调用"
author: "John Yang"
lang: zh
pubDatetime: 2023-03-02T11:59:00Z
tags:
  - Go
  - Concurrency
description: "Golang是高效的，非常高效。这种效率在很大程度上要归功于它在处理并发性问题时的独特抽象。例如，Java将其线程映射为操作系统线程，而Go使用自己的goroutines调度器将其轻量级goroutines从操作系统线程中进一步抽象出来。简而言之，Golang在使用操作系统线程方面非常节俭；如果一个go..."
---

Golang是高效的，非常高效。这种效率在很大程度上要归功于它在处理并发性问题时的独特抽象。例如，Java将其线程映射为操作系统线程，而Go使用自己的goroutines调度器将其轻量级goroutines从操作系统线程中进一步抽象出来。简而言之，Golang在使用操作系统线程方面非常节俭；如果一个goroutine被阻塞了，Go的调度器会在它的位置上切换另一个goroutine，以尽可能地保持线程的忙碌。由于每个CPU核心处理的线程数量有限（而且产生新的线程是很昂贵的），保持这些线程的工作是一件很好的事情。

那么，我们如何使用Golang来并发地进行多个http调用呢？如果你使用过C#或现代JavaScript，你可能使用过async/await来进行多个api调用。Golang并不那么容易，但这都是以效率为名义的。Go总是至少有一个goroutine在运行，它负责运行main()。我们可以在函数调用前用关键字go催生新的例程。如果你从事过Java/C#的异步调用，那么goroutine可能会让你想起上下文的概念。 Go Scheduler允许开发者制作成千上万个这种轻量级的goroutines，并为我们管理每个goroutines所花费的CPU时间。每当一个以go为前缀的函数被执行时，就会创建一个新的goroutine来运行该函数，主goroutine在生成一个新的goroutine后立即继续前进，直到它遇到一个阻塞操作符（类似于C#或Js中的await）。

***

## 原始调用

让我们从一个简单的控制台应用开始，它调用了几个GitHub配置文件，并检查连接是否成功。起初，这里没有goroutines，所有的调用都是连续进行的，效率不高。

```go

package main

import "fmt"
import "net/http"

func main() {
	links := []string{
		"https://github.com/fabpot",
		"https://github.com/andrew",
		"https://github.com/taylorotwell",
		"https://github.com/egoist",
		"https://github.com/HugoGiraudel",
	}

	checkUrls(links)
}

func checkUrls(urls []string) {
	for _, link := range urls {
		checkUrl(link)
	}
}

func checkUrl(url string) {
	_, err := http.Get(url)

	if err != nil {
		fmt.Println("We could not reach:", url)
	} else {
		fmt.Println("Success reaching the website:", url)
	}
}
```

***

## 高性能调用

首先，我们需要添加一个叫做通道的东西。由于在自己的goroutine中运行的Golang函数只是简单的函数，我们需要一种方法，通过它内部的goroutine可以把它们的结果告诉外部的goroutine；这就是使用通道来实现的。我们通过以下方式初始化它们： c := make(chan string) 我们能够使用<- 箭头将结果值发送到我们的通道，我们也可以使用这个箭头将通道的值分配出去。

第二，我们需要添加一个跟踪器，来跟踪我们应该期待多少个值从这个通道出来。这可以通过使用sync.WaitGroup.WaitGroup的类型来完成。

落实这两个想法，代码如下。
```go
import (
	"fmt"
	"net/http"
	"sync"
)

func main() {
	links := []string{
		"https://github.com/fabpot",
		"https://github.com/andrew",
		"https://github.com/taylorotwell",
		"https://github.com/egoist",
		"https://github.com/HugoGiraudel",
	}

	checkUrls(links)
}

func checkUrls(urls []string) {
	c := make(chan string)
	var wg sync.WaitGroup

	for _, link := range urls {
		wg.Add(1)   // 这告诉wg，现在这里有一个待处理的操作。
		go checkUrl(link, c, &wg)
	}

	go func() {
		wg.Wait()	// 这将阻止Goroutine，直到WaitGroup计数器为零。
		close(c)    // 通道需要被关闭，否则下面的循环将永远持续下去
	}()

    // 这个简略的循环是一个无休止的循环的语法糖，它只是在等待结果通过'c'通道进入。
	for msg := range c {
		fmt.Println(msg)
	}
}

func checkUrl(url string, c chan string, wg *sync.WaitGroup) {
	defer (*wg).Done()
	_, err := http.Get(url)

	if err != nil {
		c <- "#We could not reach:" + url    // 将结果输入通道
	} else {
		c <- "Success reaching the website:" + url    // 将结果输入通道
	}
}
```

***
