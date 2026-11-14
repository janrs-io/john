---
title: "Concurrent API Calls in Go with Goroutines"
author: "John Yang"
lang: en
pubDatetime: 2023-03-02T11:59:00Z
tags:
  - Go
  - Concurrency
description: "How Go's scheduler, goroutines, channels, and sync.WaitGroup work together to execute multiple HTTP API calls concurrently with minimal overhead."
---

Go is remarkably efficient, largely thanks to how it abstracts concurrency. Whereas languages like Java map their threads directly to OS threads, Go introduces another layer of abstraction: its runtime scheduler multiplexes lightweight **goroutines** onto a smaller pool of OS threads. Put simply, Go is frugal with OS threads—if one goroutine blocks (for instance, on network I/O), the Go scheduler swaps in another runnable goroutine on the same OS thread to keep CPU cores busy. Because OS threads are expensive to create and limited per core, keeping them productively occupied is a major win.

So how do we make multiple HTTP calls concurrently in Go? If you come from C# or modern JavaScript, you are probably used to `async`/`await`. Go takes a different, lower-level approach in the name of efficiency. Every Go program has at least one goroutine running `main()`. You spawn a new goroutine simply by prefixing a function call with the `go` keyword. The Go scheduler lets you spin up thousands of these lightweight goroutines and manages CPU time across them. As soon as a `go` statement executes, the calling goroutine continues immediately without waiting, until it hits a blocking operation (such as reading from a channel or waiting on a `sync.WaitGroup`).

***

## Sequential Calls (Baseline)

Let's start with a simple CLI program that checks reachability for several GitHub profiles. Initially, there are no goroutines—each HTTP request runs sequentially, one after another:

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

## Concurrent Calls with Goroutines, Channels, and `WaitGroup`

To run these requests concurrently and collect their results safely:

1. **Add a channel**: Because a function running in its own goroutine cannot return values directly to the caller, we use a channel (`c := make(chan string)`) to pass results back. The `<-` operator sends values into the channel and receives values out of it.
2. **Track completion with `sync.WaitGroup`**: We need a way to know when all spawned goroutines have finished so we can close the channel cleanly. `sync.WaitGroup` tracks pending operations via `Add`, `Done`, and `Wait`.

Putting both pieces together:

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
		wg.Add(1) // Register a pending goroutine with WaitGroup
		go checkUrl(link, c, &wg)
	}

	go func() {
		wg.Wait() // Block until the WaitGroup counter reaches zero
		close(c)  // Close the channel so the range loop below terminates
	}()

	// Range over the channel until it is closed
	for msg := range c {
		fmt.Println(msg)
	}
}

func checkUrl(url string, c chan string, wg *sync.WaitGroup) {
	defer (*wg).Done()
	_, err := http.Get(url)

	if err != nil {
		c <- "#We could not reach:" + url // Send result into the channel
	} else {
		c <- "Success reaching the website:" + url // Send result into the channel
	}
}
```

***
