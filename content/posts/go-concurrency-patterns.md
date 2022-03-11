+++
title = "Go 并发模式总结"
date = 2022-03-11T00:00:00+08:00
tags = ["golang", "concurrency", "goroutine"]
draft = false
+++

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


### Fan-In: 多 goroutine 版本 {#fan-in-多-goroutine-版本}

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

```text
tick 0
tock 1
tick 2
timed out
tick 3
tick 4
You're too slow.
```


## <span class="org-todo todo TODO">TODO</span> Quit chan {#quit-chan}