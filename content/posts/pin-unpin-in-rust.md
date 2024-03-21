+++
title = "Rust 中的 Pin, Unpin 和 !Unpin"
date = 2024-03-19T11:54:00+08:00
lastmod = 2024-03-21T22:53:27+08:00
tags = ["rust"]
draft = false
+++

## 为什么需要 `Pin`? {#为什么需要-pin}

引入 `Pin` 的目的主要是为了支持 **自引用类型 (self-referential types)** 。下面我们以
`Future` 为例，解释一下自引用类型以及引入 `Pin` 的必要性。

由于异步块/异步函数中可能包含对局部变量的引用，例如下面的代码：

```rust
async {
    let mut x = [0; 128];
    let read_into_buf_fut = read_into_buf(&mut x);
    read_into_buf_fut.await;
    println!("{:?}", x);
}
```

这类代码在生成 Future 结构时，就会出现 **自引用类型 (self-referential types)** 。例如，上面的代码可能生成类似下面的 Future 结构：

```rust
struct ReadIntoBuf<'a> {
    buf: &'a mut [u8], // points to `x` below
}

struct AsyncFuture {
    x: [u8; 128],
    read_into_buf_fut: ReadIntoBuf<'what_lifetime?>,
}
```

这类结构如果不能保证 self 地址的稳定性，则会出现严重的安全隐患。例如，如果
AsyncFuture 发生了移动，则 x 的地址也会发生变化，从而导致 ReadIntoBuf 中存储的
buf 指针失效。

要防止该问题，我们需要引入 `Pin` 来确保 self 地址的稳定性，以便在 `async` 块中安全地创建引用。

更多细节可参考 [Pinning - Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/04_pinning/01_chapter.html)。


## `Pin` 是什么？ {#pin-是什么}

-   `Pin<P>` 是一个 `struct`, 可用于包装任意的指针类型 `P`, 而 `Unpin` 和 `!Unpin` 都是
    `trait` 。
-   `Pin<P>` 是一个智能指针，可保证其包装的指针 `P` 后面的值不会发生移动（即物理地址保持稳定且有效），前提是其目标类型没有实现 `Unpin` 。

    例如， `Pin<&mut T>`, `Pin<&T>`, `Pin<Box<T>>` 都能保证 `T` 不会发生移动，如果 `T:
        !Unpin` 。

-   大多数类型在移动时都没啥问题，这些类型实现了一个叫 `Unpin` 的 trait。
    -   对于这些类型来讲，Pin 前和 Pin 后在使用上基本一样，不受影响。例如 `Pin<&mut
                u8>` 的行为和 `&mut u8` 没啥区别。
    -   由于标准库为 `Unpin` 提供了 `DerefMut` 的一揽子实现，因此，如果 `T` 实现了 `Unpin`
        trait, 则可以通过 Deref Coercion 自动获得 `Pin` 对象的可变引用 `&mut T`, 也可以通过 `Pin::get_mut` 方法手动获取 `&mut T`, 这意味着可以安全地对 `T` 进行修改。

        详情可参考 [Pin 的不可移动性是通过编译器来保证的](#pin-的不可移动性是通过编译器来保证的) 。

-   如果 T 带有 `!Unpin` marker，一旦被 pin 则 `T` 不可移动。
    -   `Future` 就是一个 `!Unpin` 的例子。
    -   大多数类型默认实现了 `Unpin`, 如果要强制实现 `!Unpin`, 可以在结构体中添加一个
        `PhantomPinned` 字段。参考 [ `!Unpin` 和 `Pin<Box<T>>` 示例](#unpin-和-pin-box-t-示例) 。

-   在 `T: !Unpin` 的前提下，保证 `T` 不会发生移动意味着 `T` 的物理地址是稳定且有效的，我们可以始终依赖该地址。

    这点主要通过限制对 `T` 的修改来解决。因为对于 `T: !Unpin` 来说，我们拿不到 `Pin` 所指向的 `T` 的可变引用。详情可参考 [Pin 的不可移动性是通过编译器来保证的](#pin-的不可移动性是通过编译器来保证的) 。

    > 🌟 对 [ `Pin::set` ](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.set) 方法的理解。
    >
    > `Pin::set` 方法可以设置一个新的 `T` 值以替换旧值。请注意，这种替换永远是一个完整的、合法的 `T` 值替换另一个 `T` 值，这点和直接获取 `mut` 引用有所不同，因为直接获取 `mut` 引用可能会导致不安全的修改（例如对自引用类型的 `swap` 操作）。同时，
    > `set` 方法会引起旧的值的析构，因此是安全的，没有违反 `Pin` 协议。

-   Pin 可以发生在栈上，也可以发生在堆上。
    -   栈上的 Pin 依赖于 `unsafe` 代码，且需要由 我们自己提供被 Pin 值的生命周期的保证，否则可能违反 Pin 契约。（更新：Rust 1.68 引入了 [安全版本的栈上 pin 宏](https://doc.rust-lang.org/std/pin/macro.pin.html)
        `std::pin::pin!()` ）。参考 [ `!Unpin` 和 `Pin<Box<T>>` 示例](#unpin-和-pin-box-t-示例) 。

    -   堆上的 Pin 直接用 `Box::pin()` 即可。参考 [ `Unpin` 和 `Pin<Box<T>>` 示例](#unpin-和-pin-box-t-示例) 。


## `Pin` 的不可移动性是通过编译器来保证的 {#pin-的不可移动性是通过编译器来保证的}

`Pin<P>` 仅为 Target 为 `Unpin` 的可变引用实现了 `DerefMut`, 其他情况都只能获得 Target
的不可变引用。

参考标准库中的 `Pin` 实现：

```rust
impl<P: Deref> Deref for Pin<P> {
    type Target = P::Target;
    fn deref(&self) -> &P::Target {
        Pin::get_ref(Pin::as_ref(self))
    }
}

// 为 Target 为 Unpin 的可变引用提供的 DerefMut 实现
impl<P: DerefMut<Target: Unpin>> DerefMut for Pin<P> {
    fn deref_mut(&mut self) -> &mut P::Target {
        Pin::get_mut(Pin::as_mut(self))
    }
}
```


## `Pin` 的使用场景示例 {#pin-的使用场景示例}


### 需要通过 `Future` 的 `&mut _` 引用调用 `.await` 方法时 {#需要通过-future-的-and-mut-引用调用-dot-await-方法时}

-   `async fn` 返回的 Future 是 `!Unpin` 的
-   Future 在被 poll 之前，必须被 pin 住
-   直接在 Future 上调用 `.await` 会自动处理 pin 的逻辑，但是会将该 Future 消耗掉
-   要想不消耗掉 Future (例如在 loop 里面对 Future 进行 `select!`), 需要通过 mut 引用来调用 `.await` 方法。
-   通过 mut 引用 `.await` 前，需要我们自己手动先将 Future pin 住。
-   Pin 有两种方式：
    -   **在堆上 pin:** 使用 `Box::pin` 将数据分配到堆上并 pin 住。
    -   **在栈上 pin:** 使用 `tokio::pin!` (或者 `std::pin::pin!`) 宏，将数据 pin 在栈上。

参考 [tokio::pin](https://docs.rs/tokio/latest/tokio/macro.pin.html)。

示例：

```rust
use tokio::pin;

async fn my_async_fn() {
    // async logic here
}

#[tokio::main]
async fn main() {
    let future = my_async_fn();

    // 去掉下面这句，将导致编译失败‼️
    pin!(future);

    (&mut future).await;

    // 或者用标准库的 local pin 方法：
    // std::pin::pin!(future).await;
}
```


### 对 `stream!` / `try_stream!` 宏生成的 `Stream` 进行迭代时 {#对-stream-try-stream-宏生成的-stream-进行迭代时}

和 `Future` 类似，由于 `stream!` / `try_stream!` 生成的 Stream 同样是 `!Unpin` 的，因此，在对其进行迭代操作前，同样需要先 pin 住。

详细信息可参考 [Streams in Tokio](https://tokio.rs/tokio/tutorial/streams)。


## `!Unpin` 和 `Pin<Box<T>>` 示例 {#unpin-和-pin-box-t-示例}

```rust
use std::marker::PhantomPinned;
use std::pin::Pin;

#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,

    // 表明该类型未实现 Unpin, 去掉该字段则默认实现了 Unpin
    _marker: PhantomPinned,
}

impl Test {
    fn new(txt: &str) -> Pin<Box<Self>> {
        let t = Test {
            a: String::from(txt),
            b: std::ptr::null(),
            _marker: PhantomPinned,
        };
        let mut boxed = Box::pin(t);
        let self_ptr: *const String = &boxed.a;
        unsafe { boxed.as_mut().get_unchecked_mut().b = self_ptr };

        boxed
    }

    fn a(self: Pin<&Self>) -> &str {
        &self.get_ref().a
    }

    fn b(self: Pin<&Self>) -> &String {
        unsafe { &*(self.b) }
    }
}

pub fn main() {
    let mut test1 = Test::new("test1");
    let test2 = Test::new("test2");

    println!("a: {}, b: {}", test1.as_ref().a(), test1.as_ref().b());
    println!("a: {}, b: {}", test2.as_ref().a(), test2.as_ref().b());

    // 下面这行编译报错❗
    //   cannot borrow data in dereference of `Pin<Box<Test>>` as mutable rustc (E0596)
    // trait `DerefMut` is required to modify through a dereference,
    // but it is not implemented for `Pin<Box<Test>>`
    // test1.a.push_str("hello");
}
```


## `Unpin` 和 `Pin<Box<T>>` 示例 {#unpin-和-pin-box-t-示例}

```rust
use std::pin::Pin;

#[derive(Debug)]
struct Example {
    value: i32,
}

impl Example {
    fn increment(&mut self) {
        self.value += 1;
    }
}

// Example 类型自动实现了 Unpin。

fn main() {
    let mut example = Example { value: 10 };
    let mut pinned_example = Pin::new(&mut example);

    // 由于 Example 实现了 Unpin，我们可以获取可变引用来修改它
    pinned_example.as_mut().increment();

    println!("Updated value: {}", pinned_example.value);
}
```
