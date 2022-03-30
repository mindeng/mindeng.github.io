+++
title = "Go 并发模式总结"
date = 2022-03-11T00:00:00+08:00
tags = ["golang", "concurrency", "goroutine"]
draft = false
+++

## Go 的并发哲学 {#go-的并发哲学}

<style>.org-center { margin-left: auto; margin-right: auto; text-align: center; }</style>

<div class="org-center">

Don't Communicate by sharing memory, share memory by communicating.

</div>


## Generator 发生器 {#generator-发生器}

Generator 指返回一个 chan 的函数。这是一种十分常见的使用 goroutine +
chan 的方式，可以说是一种标准用法了。

采用这种方式使用 chan 十分的安全，不会出现一些 chan 误用导致的错误（例
如向已经关闭的 chan 写入数据等）。

例如下面的代码，会开一个 goroutine 递归遍历指定目录，并将目录下的所有
json 文件通过 chan 吐出去。

```go
func walkJsonFiles(dir string) <-chan string {
	out := make(chan string)

	go func() {
		defer close(out)

		err := filepath.WalkDir(dir,
			func(path string, info fs.DirEntry, err error) error {

				if err != nil {
					log.Printf("can't access path: %q: %v\n", path, err)
					return err
				}

				if !info.Type().IsDir() && filepath.Ext(path) == ".json" {
					out <- path
				}

				return nil
			})

		if err != nil {
			log.Printf("error walking the path %q: %v\n", trace_file_dir, err)
			return
		}
	}()

	return out
}
```


## Multiplexing 多路复用（Fan-In） {#multiplexing-多路复用-fan-in}

下面这张图可以形象的表达 Fan-In 的概念：

{{< figure src="/ox-hugo/2022-03-11_08-36-52_screenshot.png" >}}


## Fan-In: 多 goroutine 版本 {#fan-in-多-goroutine-版本}

```go
func fanIn(cs ...<-chan string) <-chan string {
	out := make(chan string)
	var wg sync.WaitGroup

	// 注意：这里不能直接 wg.Wait()，需要开一个 goroutine 来 Wait
	defer func() {
		go func() {
			wg.Wait()
			close(out)
		}()
	}()

	collect := func(in <-chan string) {
		defer wg.Done()
		for n := range in {
			out <- n
		}
	}

	wg.Add(len(cs))

	// Fan-In
	for _, c := range cs {
		go collect(c)
	}

	return out
}
```


### Fan-In: select 版本 {#fan-in-select-版本}

```go
func fanInUsingSelect(input1, input2 <-chan string) <-chan string {
	out := make(chan string)

	go func() {
		defer close(out)

		for {
			select {
			case x, ok := <-input1:
				if !ok {
					input1 = nil
				} else {
					out <- x
				}
			case x, ok := <-input2:
				if !ok {
					input2 = nil
				} else {
					out <- x
				}
			}

			if input1 == nil && input2 == nil {
				break
			}
		}
	}()

	return out
}
```


## Fan-Out {#fan-out}

Fan-Out 刚好和 Fan-In 相反，一般和 Fan-In 配合起来一起使用：

```go
func processJsonFiles(ch <-chan string) <-chan {
	out := make(chan string)

	go func() {
		defer close(out)

		for s := range ch {
			// do something useful ...
			out <- s + " done"
		}
	}()

	return out
}

func fanOut(in <-chan string) <-chan string {
	// 同时开 n 个 goroutine 来处理这些 json files
	n := 20
	cs := make([]<-chan string, n)
	for i := 0; i < n; i++ {
		cs[i] = processJsonFiles(ch)
	}

	out := fanIn(cs...)

	return out
}

func main() {
	ch := walkJsonFiles("./")
	out := fanOut(ch)

	for s := range out {
		fmt.Println(s)
	}
}
```


## select 实现 Timeout {#select-实现-timeout}

<a id="code-snippet--select-timeout"></a>
```go
package main
import (
	"fmt"
	"time"
)

// 为每次 select 设置超时
// 每次 select 都会重置超时
func selectTimeout(in <-chan string) {
	for {
		select {
		case s := <-in:
			fmt.Println("tick", s)
		case <- time.After(1 * time.Second):
			fmt.Println("You're too slow.")
			return
		}
	}
}

// 为整个 for 循环设置一个超时，时间结束即退出
func selectTimeout2(in <-chan string) {
	timeout := time.After(1 * time.Second)

	for {
		select {
		case s := <-in:
			fmt.Println("tock", s)
		case <- timeout:
			fmt.Println("timed out")
			return
		}
	}
}


func main() {
	in := make(chan string)

	go func() {
		for i := 0; i < 20; i++ {
			time.Sleep(time.Duration(i * 200) * time.Millisecond)
			in <- fmt.Sprintf("%d", i)
		}
	}()

	go selectTimeout2(in)

	selectTimeout(in)
}
```

输出如下：

```text
tick 0
tock 1
tick 2
timed out
tick 3
tick 4
You're too slow.
```


## 优雅的退出 (quit chan) {#优雅的退出--quit-chan}

由于 chan 可以进行双向通信（round-trip communication），因此，可以很方
便的实现退出前的清理工作。

例如：

-   A 通知 B 退出
-   B 收到退出指令时，执行清理动作，完成后再通知 A
-   A 收到通知后，结束完整的退出流程

代码示例：
\#+name graceful-quit

```go
import (
	"fmt"
	"io/fs"
	"path/filepath"
	"errors"
)

func walkAndGracefulQuit(dir string, quit chan string) <-chan string {
	out := make(chan string)

	go func() {
		defer func() {
			close(out)
			cleanup()
			quit <- "bye"
		}()

		filepath.WalkDir(dir,
			func(path string, info fs.DirEntry, err error) error {
				if err != nil {
					fmt.Printf("can't access path: %q %v\n", path, err)
					return err
				}
				if !info.Type().IsDir() && filepath.Ext(path) == ".md" {

					select {
					case out <- path:
						// do nothing
					case <-quit:
						return errors.New("quit")
					}
				}
				return nil
			})
	}()

	return out
}

func cleanup() {
}

func main() {
	quit := make(chan string)
	c := walkAndGracefulQuit(".", quit)
	i := 0
	for name := range c {
		fmt.Println(name)
		if i++; i == 1 {
			quit <- "bye"
		}
	}

	// 等待清理动作完毕，正式结束程序
	<-quit
}
```

```text
README.md
```


## 案例学习：Google Search {#案例学习-google-search}

Go talks 有一个很经典的案例，这里我们学习一下。

假设 Google Search 有三个后端搜索服务：

-   Web Search
-   Image Search
-   Video Search

这三个分别负责搜索网页，图片和视频。现在要整合这三个服务，对用户提供完
整的搜索服务。


### Search 1.0 {#search-1-dot-0}

最简单形态，轮流访问后端服务，无并发实现：

```go
func Google(query string) (results []Result) {
	results = append(results, Web(query))
	results = append(results, Image(query))
	results = append(results, Video(query))
	return
}
```


### Search 2.0 {#search-2-dot-0}

无需锁、条件变量、回调，即可实现并发搜索：

```go
func Google(query string) (results []Result) {
	c := make(chan Result)
	go func() { c <- Web(query) } ()
	go func() { c <- Image(query) } ()
	go func() {c <- Video(query) } ()

	for i := 0; i < 3; i++ {
		result := <-c
		results = append(results, result)
	}
	return
}
```


### Search 2.1 {#search-2-dot-1}

设置搜索服务的超时时间，避免被后端较慢的（或者出故障的）服务拖垮：

```go
func Google(query string) (results []Result) {
	c := make(chan Result)
	go func(){ c <- Web(query) }()
	go func(){ c <- Image(query) }()
	go func(){ c <- Video(query) }()

	timeout := time.After(80 * time.Millisecond)
	for i := 0; i < 3; i++ {
		select {
		case result := <-c:
			results = append(results, result)
		case <-timeout:
			fmt.Println("timed out")
			return
		}
	}
}
```


### Search 3.0 {#search-3-dot-0}

通过后端服务多开（replicated），降低尾部延迟（tail latency）：

```go
func First(query string, replicas ...Search) Result {
	c := make(chan Result)
	searchReplica := func(i int) { c <- replicas[i](query) }

	for i := range replicas {
		go searchReplica(i)
	}
	return <-c
}

func Google(query string) (results []Result) {
	c := make(chan Result)
	go func() { c <- First(query, Web1, Web2) } ()
	go func() { c <- First(query, Image1, Image2) } ()
	go func() { c <- First(query, Video1, Video2) } ()
	timeout := time.After(80 * time.Millisecond)
	for i := 0; i < 3; i++ {
		select {
		case result := <-c:
			results = append(results, result)
		case <-timeout:
			fmt.Println("timed out")
			return
		}
	}
	return
}
```


## 参考资料 {#参考资料}

-   [Go Concurrency Patterns - slide](https://talks.golang.org/2012/concurrency.slide#1)
-   [Go Concurrency Patterns - source code of slide](https://cs.opensource.google/go/x/website/+/master:_content/talks/2012/concurrency.slide)