+++
title = "理解 Rust 异步编程"
date = 2024-03-21T22:37:00+08:00
lastmod = 2024-03-21T22:53:31+08:00
tags = ["rust", "async"]
draft = false
+++

Rust 的异步特性很强大，相对也比较复杂。

为了更好的理解 Rust 的异步特性，本文分别从 Rust 异步的特点、与多线程的对比、异步的用法介绍及注意事项、内部实现机制、和其他语言的横向对比等多个方面进行阐述。


## Rust 异步的特点 {#rust-异步的特点}


### `Future` 是惰性的 (_inert_) {#future-是惰性的--inert}

-   Future 只有在轮询 (poll) 时才会取得进展
-   如果 Future 被 drop 了，则不会再取得更多进展


### Async 是零成本的 (_zero-cost_) {#async-是零成本的--zero-cost}

-   无需堆内存分配
-   无需动态分派（dynamic dispatch）

关于这一点，可以参考[Rust 的异步](#rust-的异步)中的进一步解释。


### 不提供内置运行时 {#不提供内置运行时}

运行时由社区维护的 crates 提供。具体来说，Rust 的异步编程环境由以下几部分组成：

标准库
: 提供最基本的异步相关的 traits, 类型和函数。例如 `Future` trait 就是标准库提供的。

编译器
: `async/await` 语法由 Rust 编译器直接提供支持。

[ `futures` crate](https://docs.rs/futures/)
: 提供通用工具类型、宏和函数，它们可以在任何 async 程序中使用。这些东西将来可能会成为标准库的一部分。

    事实上 `futures` crate 自带了一个 executor 可以用来[执行简单的异步任务](#用法介绍)，但是不包括 async I/O 以及 timer 的支持，可以看成是一个不完整的运行时环境，因此一般需要搭配其他运行时来使用。


运行时
: 异步代码的执行，IO 和任务生成 (task spawning) 由 async 运行时提供，例如 Tokio
    和 async-std。大部分异步程序以及一些异步 crates 会依赖于特定的运行时。关于这部分的更多信息可参考：[The Async Ecosystem](https://rust-lang.github.io/async-book/08_ecosystem/00_chapter.html) 。


### 单线程、多线程两种运行时可供选择 {#单线程-多线程两种运行时可供选择}

以 Tokio 为例， _rt_ 和 _rt-multi-thread_ 两个 feature flag 分别代表了单线程运行时和多线程运行时。


### 缺失部分语言功能 {#缺失部分语言功能}

一些同步 Rust 的语言功能在 async 中可能不可用，例如，不能在 trait 中定义 `async`
函数。


## 与多线程的对比 {#与多线程的对比}


### 线程适用于少量任务场景 {#线程适用于少量任务场景}


#### 缺点 {#缺点}

-   线程会带来 CPU 和内存开销
-   创建和切换线程的成本很高，即使是空线程也会消耗系统资源
-   使用线程池有一定缓解作用，但无法全部消除


#### 优点 {#优点}

-   由于不需要特殊的编程模型，因此在复用现有的同步代码时，无需太大的改造成本。
-   有些 OS 可以修改线程的优先级，在一些延迟敏感的应用中很有用（例如驱动程序）。


### Async 可显著降低 CPU 和内存开销 {#async-可显著降低-cpu-和内存开销}


#### 优点 {#优点}

-   Async 可显著降低 CPU 和内存开销，尤其是对于 IO 密集型的任务来说，例如服务器和数据库应用。
-   同等条件下，Async 可以比线程拥有多出几个数量级的任务。


#### 缺点 {#缺点}

-   会产生更大的二进制文件。

    原因如下：

    -   async 函数的执行过程是通过状态机来管理的，因此编译器会自动为每个 async 函数生成状态机代码。
    -   每个二进制文件都会捆绑一个 async 运行时。

-   开发过程可能会遇到更多问题。

    主要有以下几点：

    -   **可能会碰到更多的编译错误:** 由于涉及更复杂的语言功能，例如 lifetimes 和 pinning, 可能更容易碰到这类错误。
    -   **运行时的错误堆栈会更复杂:** 因为涉及编译器为 async 函数生成的状态机。
    -   **一些新的错误模式:** -   在异步上下文中调用一个阻塞函数
        -   没有正确实现 `Future` trait

        这些错误可以悄悄地通过编译器，有时甚至可以通过单元测试。


## 用法介绍 {#用法介绍}

以下示例参考自 [async/.await Primer](https://rust-lang.github.io/async-book/01_getting_started/04_async_await_primer.html) 。

```rust
// `block_on` blocks the current thread until the provided future has run to
// completion. Other executors provide more complex behavior, like scheduling
// multiple futures onto the same thread.
use futures::executor::block_on;

#[derive(Debug)]
struct Song {
    name: String,
}

async fn learn_song() -> Song {
    println!("learn song...");
    Song {name: "hello".to_string()}
}
async fn sing_song(song: Song) {
    println!("sing song: {:?}", song);
}
async fn dance() {
    println!("dance ...");
}

async fn learn_and_sing() {
    // Wait until the song has been learned before singing it.
    // We use `.await` here rather than `block_on` to prevent blocking the
    // thread, which makes it possible to `dance` at the same time.
    let song = learn_song().await;
    sing_song(song).await;
}

async fn async_main() {
    let f1 = learn_and_sing();
    let f2 = dance();

    // `join!` is like `.await` but can wait for multiple futures concurrently.
    // If we're temporarily blocked in the `learn_and_sing` future, the `dance`
    // future will take over the current thread. If `dance` becomes blocked,
    // `learn_and_sing` can take back over. If both futures are blocked, then
    // `async_main` is blocked and will yield to the executor.
    futures::join!(f1, f2);
}

fn main() {
    block_on(async_main());
}
```

```text
learn song...
sing song: Song { name: "hello" }
dance ...
```

下面是对该示例的几点说明：

-   `learn_song` 在 `sing_song` 之前，二者是顺序执行的。
-   `dance` 和 `learn_and_sing` 是并发执行的。
-   `.await` 调用会导致 async 函数在当前 `Future` 上等待直到完成，但是会在当前 `Future`
    阻塞时，出让当前线程的控制权并允许其他 async 函数继续执行。


## 使用异步时的注意事项（容易踩的坑） {#使用异步时的注意事项-容易踩的坑}


### `async` 的生命周期 {#async-的生命周期}

`async fn` 如果有 references 作为入参，则返回的 `Future` 会受到该引用的生命周期的约束。也就是说，返回的 future 必须在入参还有效时执行完 `.await` 。

```rust
use std::future::Future;
// This function:
async fn foo(x: &u8) -> u8 { *x }

// Is equivalent to this function:
fn foo_expanded<'a>(x: &'a u8) -> impl Future<Output = u8> + 'a {
    async move { *x }
}
```

在下面的例子中，通过将参数和对异步函数的调用打包到一个 `async` 块中，来解决
references-as-arguments 的生命周期问题。这种方式实际上将 `borrow_x` 返回的带生命周期约束的 future 转变成了一个 `'static` future。

```rust
fn bad() -> impl Future<Output = u8> {
    let x = 5;
    borrow_x(&x) // ERROR: `x` does not live long enough
}

fn good() -> impl Future<Output = u8> {
    async {
        let x = 5;
        borrow_x(&x).await
    }
}
```


### `async move` {#async-move}

-   async 块默认是以引用的方式捕获外部值的
-   async move 以 move 的方式捕获外部值，好处是生命周期可以超出该变量原来的作用域。

<!--listend-->

```rust
/// `async` block:
///
/// Multiple different `async` blocks can access the same local variable
/// so long as they're executed within the variable's scope
async fn blocks() {
    let my_string = "foo".to_string();

    let future_one = async {
        // ...
        println!("{my_string}");
    };

    let future_two = async {
        // ...
        println!("{my_string}");
    };

    // Run both futures to completion, printing "foo" twice:
    let ((), ()) = futures::join!(future_one, future_two);
}

/// `async move` block:
///
/// Only one `async move` block can access the same captured variable, since
/// captures are moved into the `Future` generated by the `async move` block.
/// However, this allows the `Future` to outlive the original scope of the
/// variable:
fn move_block() -> impl Future<Output = ()> {
    let my_string = "foo".to_string();
    async move {
        // ...
        println!("{my_string}");
    }
}
```


### `Future` 的跨线程移动 {#future-的跨线程移动}

当使用多线程执行器时， `Future` 可能会跨线程移动 (move), 这种移动发生在 `.await` 调用时。因此，当变量的作用域涉及跨 `.await` 调用时：

-   该变量类型要求实现 `Send` trait
-   如果涉及引用，则要求实现 `Sync` trait


### `Future` 和 `Pin` {#future-和-pin}

关于 `Pin` 的解释以及使用场景，可以参考我之前发布的 [Rust 中的 `Pin`, `Unpin` 和 `!Unpin`
]({{< relref "pin-unpin-in-rust" >}})一文。


### 传统互斥体的局限性 {#传统互斥体的局限性}

由于 `Future` 潜在的跨线程移动特性，在使用 `Mutex` 时也需要注意，不能在跨 `.await` 调用中持有传统的 non-futures-aware 锁，因为这样做可能会导致死锁的发生。

死锁案例分析（其中 task 的概念[在后文中有解释](#内部实现分析-task-executor-和-spawner)）：

1.  假设 task A 和 task B 共享同一把锁 L
2.  task A 先拿到锁 L
3.  task A 中执行了 `.await`, 并将当前线程让度给 task B
4.  task B 执行获取锁 L 的操作。由于此时 L 已经被 task A 所持有，且 task A 已经挂起没有机会再释放锁 L, 因此 task B 永远拿不到该锁，从而导致死锁发生。

Task B 要想顺利拿到该锁，应该要具备两个条件：

-   task A 和 task B 在同一个线程内调度执行
-   L 是一把可重入锁 (reentrant lock)

因此，这种情况下应该使用 `futures::lock::Mutex` (如果使用的是 tokio 运行时，则可以使用 `tokio::sync::Mutex`), 而不是 `std::sync::Mutex` 。

更多细节请参考：

-   [async/await](https://rust-lang.github.io/async-book/03_async_await/01_chapter.html#awaiting-on-a-multithreaded-executor)
-   [Shared state | Tokio](https://tokio.rs/tokio/tutorial/shared-state#holding-a-mutexguard-across-an-await)


## 内部实现分析：Task, Executor 和 Spawner {#内部实现分析-task-executor-和-spawner}

要进一步理解 Rust 异步编程，Task, Executor 和 Spawner 这几个概念不可避免。下面对这些概念进行逐一解释。


### Task {#task}

-   Task 是对一个或多个 Future 的封装，代表了一个可以被 Executor 执行的独立的异步工作单元。
-   一个 Task 通常会包含一个顶层 Future，这个 Future 可能会依赖其他更多的 Future。
-   Task 在被创建后会被提交给 Executor，由 Executor 负责调度和执行。


### Executor {#executor}

-   Executor 是一个负责调度和执行 Task 的组件。
-   它会不断轮询已经提交给它的 Task，通过调用 Task 内部 Future 的 poll 方法来驱动这些 Future 向完成状态前进。
-   Executor 可以是单线程的，也可以是多线程的，以支持不同的并发需求。


### Spawner {#spawner}

-   Spawner 是一个用于创建和提交 Task 到 Executor 的组件。
-   在一些异步运行时（如 tokio 或 async-std）中，Spawner 通常是与 Executor 紧密绑定的，提供了方便的接口来启动新的 Task。
-   在 [Build an Executor](https://rust-lang.github.io/async-book/02_execution/04_executor.html) 这个示例中，Spawner 内部持有一个 channel 的 Sender, 而
    Executor 则持有该 channel 对应的 Receiver, 用于接收 Spawner 发送过来的 task。
    Executor 会在一个循环中持续接收 task 并执行 poll 逻辑。


### 对 [Build an Executor](https://rust-lang.github.io/async-book/02_execution/04_executor.html) 示例的几点理解 {#对-build-an-executor-示例的几点理解}

-   Future 先被 Box 装箱, 然后存在 Task 中
-   Task 被包装在 Arc 中，以便跨线程共享所有权
-   Task 中除了 Future, 还包含一个 task_sender
    -   task_sender 是 channel 的 Sender 的克隆
    -   该 sender 用来在被唤醒时，重新将该 Task 发送到 channel 队列中，以便 executor
        重新调度执行该 Task。
-   Executor 中包含一个 channel 的 Receiver, 不断接收 task, 取出 Future 并执行
    poll。
-   poll 有一个 context 参数，是 Executor 创建的，context 中包含了 waker。
    -   waker 底层其实就是 task 的引用，只是以 `Waker` 接口的形式存在。当任务阻塞时会注册到某个触发器当中（例如 timer, 或者 epoll 事件等）。
    -   事件触发时，意味着任务可以继续执行。此时会调用 `waker.wake()` 方法，该方法会调到 Task 自身实现的 `ArcWake` trait 中的 `wake_by_ref` 方法，这里面就会调用
        `task_sender.send()` 将 task 的克隆重新发送到 channel 中。


## 和其他语言的横向对比 {#和其他语言的横向对比}

Rust 的异步和其他语言相比，最显著的特点就是 Future 的惰性。

下面分别对 Rust, Kotlin, Dart 和 Go 这几种语言的异步特性进行一个简单的概括性介绍，希望通过这种对比来加深对 Rust 异步特性的理解。


### Rust 的异步 {#rust-的异步}

懒执行（Lazy Execution）
: 在 Rust 中，当你定义一个 async 函数时，调用这个函数实际上并不会立即执行它的代码。相反，它返回一个未执行的 future。这个 future 必须被显式地轮询（poll），通常是在一个异步上下文中调用 .await ，或者使用某种执行器（executor）来驱动。


零成本抽象（Zero-Cost Abstractions）
: Rust 的异步实现旨在尽可能减少运行时开销。它通过状态机的转换来实现异步操作，并不直接依赖于线程或其他重量级的并发机制。

    Rust 通过在编译时将 async 函数转换成状态机来实现异步函数的运行、挂起和恢复。这种方法允许精细控制异步操作的执行，同时与 Rust 的零成本抽象原则相符。


明确的所有权和借用
: 由于 Rust 的所有权和借用规则，异步代码在编译时就能最大限度的避免数据竞争和并发相关的安全性问题。


### Kotlin 的异步 {#kotlin-的异步}

协程（Coroutines）
: Kotlin 使用协程来处理异步操作，这是一种轻量级的线程。Kotlin 的协程是立即执行的。

    Kotlin 中的协程也是通过编译时转换来实现的。当你在 Kotlin 中使用协程时，编译器会将协程代码转换为状态机（参考 [Kotlin language specification](https://kotlinlang.org/spec/asynchronous-programming-with-coroutines.html#coroutine-state-machine)）。这种转换类似于
    Rust 的处理方式，但在细节上有所不同，最大的区别是 Kotlin 的协程是基于 JVM 的，因此它们必须在 JVM 的限制和特性（如垃圾收集、JVM 线程模型等）下工作。


结构化并发（Structured Concurrency）
: Kotlin 强调在协程中使用[结构化并发](#结构化并发)，这有助于防止常见的并发相关错误。


上下文感知
: Kotlin 协程可以很容易地切换上下文，例如从后台线程切换到主线程。


#### 结构化并发 {#结构化并发}

在 Kotlin 中，当你启动一个协程，它总是与一个特定的作用域（协程作用域）相关联。这个作用域负责管理协程的生命周期，包括启动和取消。结构化并发的关键点在于：

作用域绑定
: 每个协程都运行在一个明确的作用域内，这个作用域定义了协程的生命周期。协程只在这个作用域内活动，一旦作用域结束，所有在这个作用域中启动的协程也会被自动取消。


父子关系
: 在结构化并发中，父协程会等待其所有子协程完成。如果父协程被取消，所有的子协程也会被取消。


异常传播
: 在子协程中发生的异常会被传播到父协程中，这使得异常处理更加一致和可预测。


### Dart 的异步 {#dart-的异步}

事件循环（Event Loop）
: Dart 使用单线程事件循环模型，所有的异步操作都是围绕这个事件循环来调度的。


Future 和 Stream
: Dart 中的异步模式主要是通过 Future 和 Stream 实现的。


### Go 的异步 {#go-的异步}

协程（Goroutines）
: Go 使用 Goroutines 来处理并发，这是一种非常轻量级的线程。Goroutines 在创建时就开始执行。


通道（Channels）
: Go 使用 `chan` 来在 Goroutines 之间进行通信，这是一种非常强大的并发模型。


简单直接
: Go 的并发模型非常简单直接，易于理解和使用。
