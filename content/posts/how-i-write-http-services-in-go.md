
---
title: "经历八年后，我是如何用 Go 写 HTTP 服务的"
date: 2023-02-22T03:15:00.000Z
lastmod: 2023-02-22T08:49:00.000Z
tags: ['golang', 'server']
draft: false
---


今天读到一篇不错的文章，讲如何用 Go 写 HTTP 服务的，很有同感，翻译如下。

[原文链接](https://pace.dev/blog/2018/05/09/how-I-write-http-services-after-eight-years.html)


## A Server struct

一个 Server struct 是一个代表服务的对象，持有所有依赖。

每个组件都有一个唯一的 server struct，最后看起来通常类似这个样子：

```go
type server struct {
	db *someDatabase
	router *someRouter
	email EmailSender
}
```  
  
-   该结构的字段主要是各种需要共享的依赖


## routes.go

每个组件都有一个 ``routers.go`` 文件，包含所有的路由：

```go
package app

func (s *server) routes() {
	s.router.HandleFunc("/api/", s.handleAPI())
	s.router.HandleFunc("/about", s.handleAbout())
	s.router.HandleFunc("/", s.handleIndex())
}
```

由于大部分代码维护工作都是从一个 URL 和一个错误报告开始的，所以只需要看一眼 ``routes.go`` 文件，即可知道应该去那里查找问题。


## Handlers 挂着（hang off） server 对象

HTTP handlers 挂着 server 对象：

```go
func (s *server) handleSomething() http.HandlerFunc { ... }
```

Handlers 可以通过 server 对象访问依赖。


## 返回 handler

Handler 函数不直接处理请求，而是返回一个函数处理之。

这样我们就有一个闭包环境，在这里我们的 handler 可以这样操作：

```go
func (s *server) handleSomething() http.HandlerFunc {
	thing := prepareThing()
	return func(w http.ResponseWriter, r *http.Request) {
		// use the thing
	}
}
```

``prepareThing()`` 方法只会被调用一次，因此你可以用来执行一次性的 handler 初始化动作，然后在 handler 中使用初始化的结果（ ``thing`` ）。

在访问共享数据时，确保只执行读操作，否则需要加锁或者类似的保护措施。


## 通过参数传递 handler 的特定依赖

如果一个 handler 需要一个特殊依赖，可以通过参数来传递。

```go
func (s *server) handleGreeting(format string) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, format, "World")
	}
}
```


## 使用 HandlerFunc 而非 Handler

我几乎在所有情况下都使用 ``http.HandlerFunc`` ，而非 ``http.Handler`` 。

```go
func (s *server) handleSomething() http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// ...
	}
}
```

两者基本上是可互换的，只需要选一个可读性更强的就好。对我而言， ``http.HandlerFunc`` 会好点。


## 中间件就是普通的 Go 函数

中间件函数接受一个 ``http.HandlerFunc`` 参数，并返回一个新的 ``http.HandlerFunc`` ，新的这个 handler 可以在调用传入的 handler 之前或之后，执行任意代码，也可以选择完全不执行传入的 handler。

```go
func (s *server) adminOnly(h http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if !currentUser(r).IsAdmin {
			http.NotFound(w, r)
			return
		}
		h(w, r)
	}
}
```

上述例子中，如果 ``IsAdmin`` 为 false，则返回 404 并且终止处理。注意这种情况下，传入的 h handler 并未被调用。

如果 ``IsAdmin`` 为 true，则正常走传入的 h handler 逻辑。

中间件也可以列在 ``routes.go`` 中：

```go
package app
func (s *server) routes() {
	s.router.HandleFunc("/api/", s.handleAPI))
	s.router.HandleFunc("/about", s.handleAbout())
	s.router.HandleFunc("/", s.handleIndex())
	s.router.HandleFunc("/admin", s.adminOnly(s.handleAdminIndex()))
}
```


## 就地定义请求和响应类型

如果一个端点（endpoint）有自己的请求、响应类型，通常这些类型只对改 handler 有用。

如果确实如此，则可以直接在函数内部定义这些类型：

```go
func (s *server) handleSomething() http.HandlerFunc {
	type request struct {
		Name string
	}
	type response struct {
		Greeting string `json:"greeting"`
	}
	return func(w http.ResponseWriter, r *http.Request) {
		// ...
	}
}
```

这样不会污染你的包命名空间，允许你在不同的 handler 中使用相同的名字，而非为每个 handler 想一个不同的名字。

在测试代码中，也可以直接拷贝这些类型定义到测试函数中。


## 类型定义可以帮助人们构造测试用例，以及理解代码

如果你的请求、响应类型隐藏在 handler 内部，你可以在测试代码中直接定义新类型。

这是一个表达你的意图，方便后人理解你的代码的机会。

例如，假设有一个 ``Person`` 类型，在很多端点（endpoint）中被复用。其中有一个 ``/greet`` 端点，我们大概率只关心 ``Person.name`` 这个字段，因此我们可以在测试代码中表达这一点：

```go
func TestGreet(t *testing.T) {
	is := is.New(t)
	p := struct {
		Name string `json:"name"`
	}{
		Name: "Mat Ryer",
	}
	var buf bytes.Buffer
	err := json.NewEncoder(&buf).Encode(p)
	is.NoErr(err) // json.NewEncoder
	req, err := http.NewRequest(http.MethodPost, "/greet", &buf)
	is.NoErr(err)
	// ... more test code here
}
```

> 仅从功能测试角度来讲，这么做是 OK 的，被测代码的用法也表达的很清楚。但是从鲁棒性测试的角度，也许需要考虑到传递整个数据结构进去，会不会产生什么问题？  


## 利用 sync.Once 设置依赖

在准备 handler 的时候，如果需要执行一些成本比较高的初始化操作，可以考虑将该操作延迟到该 handler 第一次被调用的时候。

这可以改善应用的启动时间。

```go
func (s *server) handleTemplate(files string...) http.HandlerFunc {
	var (
		init    sync.Once
		tpl     *template.Template
		tplerr  error
	)
	return func(w http.ResponseWriter, r *http.Request) {
		init.Do(func() {
			tpl, tplerr = template.ParseFiles(files...)
		})
		if tplerr != nil {
			http.Error(w, tplerr.Error(), http.StatusInternalServerError)
			return
		}
		// use tpl
	}
}
```

``sync.Once`` 确保该代码只会被执行一次，而且其他调用（其他人发起同一个请求时）会一直阻塞直到执行结束。  
  
-   错误检查放在 init 函数外面，因此如果有错误发生，我们可以暴露出该错误，同时保留错误日志  
-   如果该 handler 未被调用，则该高成本操作永远不会被执行。有些情况下这样做有很大收益，取决于你的代码是如何部署的

> 这种方式实际上是将初始化时间从启动阶段转移到了运行时。如果使用 Google App Engine 则很有用，其他场景则需要单独考虑。  


## 服务可测性

上述的 server 类型是充分可测的。

```go
func TestHandleAbout(t *testing.T) {
	is := is.New(t)
	srv := server {
		db:     mockDatabase,
		email:  mockEmailSender,
	}
	srv.routes()
	req := httptest.NewRequest("GET", "/about", nil)
	w := httptest.NewRecorder()
	srv.ServeHTTP(w, req)
	is.Equal(w.StatusCode, http.StatusOK)
}
```  
  
-   在每个测试中创建一个 server 实例 —— 如果耗时操作是懒加载的，那么这么做不会耗费太多时间，即使对于大组件来说也适用  
-   通过调用 server 的 ``ServeHTTP`` 方法，包括路由、中间件等整个栈都可以被测到。当然，如果你不希望测试整个栈，也可以直接调用 handler 方法  
-   使用 ``httptest.NewRequest`` 和 ``httptest.NewRecorder`` 来记录 handlers 都做了什么  
-   代码中使用了 ``is`` 测试框架，Testify 的一个迷你替代版本：[is](https://github.com/matryer/is)
