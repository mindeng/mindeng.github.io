+++
title = "Rust Trait+ 系列"
lastmod = 2024-07-03T17:52:03+08:00
draft = false
+++

这篇文章介绍了一系列 Rust 中利用 trait 实现的通用能力或惯用法，这些内容也是
Rust 编程中较常见的概念、方法和技巧，实用性很强，我称之为“Trait+ 系列”。

建议先阅读 [Rust 中的特征 (Trait)]({{< relref "traits-in-rust" >}}) 一文，配合食用效果更佳。


## Trait + 类型转换 {#trait-plus-类型转换}


### `From` 和 `Into` {#from-和-into}

`From` 和 `Into` trait 规定了一种惯用的类型转换方式。实现了 `From`, 就可以“免费”获得
`Into`:

```rust
use std::convert::From;

#[derive(Debug)]
struct Number {
    value: i32,
}

impl From<i32> for Number {
    fn from(item: i32) -> Self {
        Number { value: item }
    }
}

fn main() {
    let num = Number::from(30);
    println!("My number is {:?}", num);

    let num: Number = 40.into();
    println!("My number is {:?}", num);
}
```

```text
My number is Number { value: 30 }
My number is Number { value: 40 }
```


### `TryFrom` 和 `TryInto` {#tryfrom-和-tryinto}

和 `From`, `Into` 类似，只不过返回的是 `Result`:

```rust
use std::convert::TryFrom;
use std::convert::TryInto;

#[derive(Debug, PartialEq)]
struct EvenNumber(i32);

impl TryFrom<i32> for EvenNumber {
    type Error = ();

    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if value % 2 == 0 {
            Ok(EvenNumber(value))
        } else {
            Err(())
        }
    }
}

fn main() {
    // TryFrom

    assert_eq!(EvenNumber::try_from(8), Ok(EvenNumber(8)));
    assert_eq!(EvenNumber::try_from(5), Err(()));

    // TryInto

    let result: Result<EvenNumber, ()> = 8i32.try_into();
    assert_eq!(result, Ok(EvenNumber(8)));
    let result: Result<EvenNumber, ()> = 5i32.try_into();
    assert_eq!(result, Err(()));
}
```


### String 转换 {#string-转换}


#### 转换成 String: `fmt::Display` {#转换成-string-fmt-display}

一个类型要转换成 `String`, 一般会实现 `fmt::Display` trait, 而不是 `ToString` (参考“[一揽子实现]({{< relref "traits-in-rust#一揽子实现--blanket-implementations" >}})”中的说明):

```rust
use std::fmt;

struct Circle {
    radius: i32
}

impl fmt::Display for Circle {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Circle of radius {}", self.radius)
    }
}

fn main() {
    let circle = Circle { radius: 6 };
    println!("{}", circle.to_string());
}
```

```text
Circle of radius 6
```


#### 解析 String: `FromStr` {#解析-string-fromstr}

一个类型要支持从一个字符串中解析出来，需要实现 `FromStr` trait:

```rust
fn main() {
    let parsed: i32 = "5".parse().unwrap();
    let turbo_parsed = "10".parse::<i32>().unwrap();

    let sum = parsed + turbo_parsed;
    println!("Sum: {:?}", sum);
}
```

```text
Sum: 15
```


### `Box<dyn Trait>` 的 `downcast` {#box-dyn-trait-的-downcast}

我有一个 trait 的包装类型 `Box<dyn Trait>` 的变量，如何获得其底层的具体类型的引用呢？即如何获得该变量对应的实现该 trait 的 struct 的引用呢？

这是一个 `downcast` 的过程，类似于 C++ 中的 `dynamic_cast` 。

Rust 中应该怎么做？这就要用到一个叫做 `Any` 的 trait。示例如下：

```rust
use std::any::Any;

pub trait Counter {
    fn inc(&mut self);
    fn as_any_mut(&mut self) -> &mut dyn Any;
}

pub struct ConcreteCounter {
    pub count: u8,
}

impl ConcreteCounter {
    pub fn new() -> ConcreteCounter {
        ConcreteCounter { count: 1 }
    }
}

impl Counter for ConcreteCounter {
    fn inc(&mut self) {
        self.count += 1;
        println!("count: {}", self.count);
    }
    fn as_any_mut(&mut self) -> &mut dyn Any {
        self
    }
}

fn main() {
    let counter = ConcreteCounter::new();
    let mut dyn_counter: Box<dyn Counter> = Box::new(counter);

    // counter: &mut ConcreteCounter
    let counter = dyn_counter
        .as_any_mut()
        .downcast_mut::<ConcreteCounter>()
        .expect("dyn_counter is not a ConcreteCounter");

    counter.inc();
    counter.count = 0;
    counter.inc();
}
```

```text
count: 2
count: 1
```

大概步骤如下：

1.  `Box<dyn Counter>` &rarr; `&mut dyn Any` (或者 `&dyn Any`, 如果不需要可变引用)

    这里需要 `Counter` 包含一个 `as_any_mut` 方法，以便在 `ConcreteCounter` 中实现。

2.  `&mut dyn Any` &rarr; `&mut ConcreteCounter`

    这是通过调用 `Any` 的 `downcast_mut` (对应不可变引用是 `downcast_ref`) 方法实现的。

类似地， `&dyn Trait` 也可以通过上述方法来获取其底层具体的 struct 的引用。


## Trait + 闭包 (_Closure_) {#trait-plus-闭包--closure}

根据闭包 (closure) 处理参数的方式，闭包会自动实现以下三个 `Fn` traits 中的一个或多个：

1.  `FnOnce` 适用于可以调用一次的闭包。所有闭包都会至少实现该 trait, 因为所有的闭包都可以被调用。

    一个闭包如果将捕获的值 move 到闭包外部，则该闭包将仅实现该 `FnOnce` 而不会实现其他 `Fn` traits, 因为该闭包只能被调用一次。

2.  `FnMut` 适用于如下闭包：这类闭包不会将捕获的值 move 到闭包外部，但可能会修改捕获的值。这类闭包可以被调用多次。

3.  `Fn` 适用于如下闭包：这类闭包不会将捕获的值 move 到闭包外部，也不会修改捕获的值，或者根本不捕获任何值。

    这类闭包可以被调用多次，且不会修改环境（对环境无副作用），这在并发多次调用闭包等场景下非常重要。

以上 trait 对闭包的要求按照顺序逐渐递增： `FnOnce` 对闭包没有任何特殊要求，而 `Fn`
的要求最严格。


### `FnOnce` 的例子 {#fnonce-的例子}

`Option<T>.unwrap_or_else` 方法中的闭包参数就声明了 `FnOnce` 约束，意味着该方法可以接受任意类型的闭包：

```rust
impl<T> Option<T> {
    pub fn unwrap_or_else<F>(self, f: F) -> T
    where
        F: FnOnce() -> T
    {
        match self {
            Some(x) => x,
            None => f(),
        }
    }
}
```

> 💡 一个普通函数也可以实现全部三个 `Fn` traits。
>
> 如果不需要从环境中捕获值，我们可以在需要传入某个 `Fn` trait 的地方使用函数名而非闭包。例如：在一个 `Option<Vec<T>>` 上调用 `unwrap_or_else(Vec::new)`, 当该 option 为
> `None` 时，我们可以获得一个新的空 vector。


### `FnMut` 的例子 {#fnmut-的例子}

下面的示例演示了通过 `sort_by_key` 方法给数组排序。该方法接受 `FnMut` 闭包 (或者 `Fn`
闭包), 原因是它会调用该闭包多次，每个 item 一次。

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let mut list = [
        Rectangle { width: 10, height: 1 },
        Rectangle { width: 3, height: 5 },
        Rectangle { width: 7, height: 12 },
    ];

    list.sort_by_key(|r| r.width);
    println!("{:#?}", list);
}
```

```text
[
    Rectangle {
        width: 3,
        height: 5,
    },
    Rectangle {
        width: 7,
        height: 12,
    },
    Rectangle {
        width: 10,
        height: 1,
    },
]
```

如果你传入一个仅实现 `FnOnce` 的闭包，则会编译失败，例如：

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let mut list = [
        Rectangle { width: 10, height: 1 },
        Rectangle { width: 3, height: 5 },
        Rectangle { width: 7, height: 12 },
    ];

    let mut sort_operations = vec![];
    let value = String::from("by key called");

    list.sort_by_key(|r| {
        sort_operations.push(value); // value 被 move out 了，编译失败❗
        r.width
    });
    println!("{:#?}", list);
}
```

编译器报告的错误如下：

```text
error[E0507]: cannot move out of `value`, a captured variable in an `FnMut` closure
  --> src/main.rs:19:30
   |
16 |     let value = String::from("by key called");
   |         ----- captured outer variable
17 |
18 |     list.sort_by_key(|r| {
   |                      --- captured by this `FnMut` closure
19 |         sort_operations.push(value); // value 被 move out 了，编译失败❗
   |                              ^^^^^ move occurs because `value` has type `String`, which does not implement the `Copy` trait

For more information about this error, try `rustc --explain E0507`.
error: could not compile `cargoUnHfvy` (bin "cargoUnHfvy") due to previous error
```

由于该闭包将 `value` move 到闭包外部，该闭包仅实现了 `FnOnce` trait (只能被调用一次),
因此不符合 `FnMut` 的规范。

相反，下面的例子是合法的，因为该闭包仅捕获了 mutable 引用，没有对捕获的变量进行
`move` 操作，因此符合 `FnMut` 的规范（可以被多次调用）：

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let mut list = [
        Rectangle { width: 10, height: 1 },
        Rectangle { width: 3, height: 5 },
        Rectangle { width: 7, height: 12 },
    ];

    let mut num_sort_operations = 0;
    list.sort_by_key(|r| {
        num_sort_operations += 1;
        r.width
    });
    println!("{:#?}, sorted in {num_sort_operations} operations", list);
}
```

```text
[
    Rectangle {
        width: 3,
        height: 5,
    },
    Rectangle {
        width: 7,
        height: 12,
    },
    Rectangle {
        width: 10,
        height: 1,
    },
], sorted in 6 operations
```


## Trait + 所有权 (_Ownership_) {#trait-plus-所有权--ownership}

Rust 在处理 _ownership_ 规则时，会根据类型是否实现了 `Copy` trait 来区别对待。具体而言：

-   实现了 `Copy` trait 的类型，其值可以被存储在栈上。
-   实现了 `Copy` trait 的类型，在赋值和传参时，不会发生 `move`, 而是直接拷贝。
-   **未实现** `Copy` trait 的类型，在赋值和传参时，会发生 `move`, 之后不再有效。
-   实现了 `Drop` trait 的类型，在其 owner 超出作用域范围时，会调用其 `drop` 方法。

> 💡 如果一个类型（或者该类型的一部分）实现了 `Drop` trait, 则不能实现 `Copy` trait。这二者是互斥的，如果同时存在，会导致编译错误。


### `Copy` trait {#copy-trait}

存储在 stack 上的数据拷贝速度很快，而且深拷贝和浅拷贝没有任何区别，因此可以直接采用 copy 的方式处理。Rust 通过 `Copy` trait 来标识这类数据。

以下是一些常见的实现了 `Copy` trait 的类型：

-   [标量类型（Scalar types）](https://doc.rust-lang.org/book/ch03-02-data-types.html#scalar-types)由于 size 固定，可以直接存储在栈上。
-   元组（Tuple）如果只包含实现了 `Copy` trait 的类型，则也被视为实现了 `Copy` trait。
    -   例如 `(i32, i32)` 实现了 `Copy`, 但 `(i32, String)` 则未实现。


### `Drop` trait {#drop-trait}

存储在 heap 上的数据一般 size 不确定，且拷贝成本较高，因此采用 _move_ 的方式处理。

这类数据在超出作用域范围时，往往需要做一些特殊处理以便回收内存或释放资源，因此需要实现 `Drop` trait。

针对这类型数据，如果在某些场合确实需要进行“深拷贝”操作，可以通过显式调用对象的
`clone()` 方法手动进行深拷贝。

实现了 `Drop` trait 的类型示例：

-   `Box`
-   `Vec`
-   `String`
-   `File`
-   `Process`

`Drop` trait 使用示例：

```rust
struct Droppable {
    name: &'static str,
}

impl Drop for Droppable {
    fn drop(&mut self) {
        println!("> Dropping {}", self.name)
    }
}

fn main() {
    let _a = Droppable { name: "a" };

    // block A
    {
        let _b = Droppable { name: "b" };

        // block B
        {
            let _c = Droppable { name: "c" };
            let _d = Droppable { name: "d" };

            println!("Exiting block B");
        }
        println!("Just exited block B");

        println!("Exiting block A");
    }
    println!("Just exited block A");

    // 手动触发 drop
    drop(_a);

    println!("end of the main function");
}
```

```text
Exiting block B
> Dropping d
> Dropping c
Just exited block B
Exiting block A
> Dropping b
Just exited block A
> Dropping a
end of the main function
```


#### String 类型 {#string-类型}

执行 `let s2 = s1;` 时发生的事情（参考下方的 String 内存布局图）：

-   ptr, len, capacity 都是存储在 stack 上的，因此会直接拷贝。
-   ptr 指向的字符串数据存储在 heap 上，不会发生拷贝，而是被 _move_ 了。

`String s1` 的内存布局：

{{< figure src="/ox-hugo/2023-07-16_07-16-48_trpl04-01.svg" >}}


### 隐含的设计上的选择 {#隐含的设计上的选择}

> 🌟 _Tips_
>
> Rust 永远不会为你的数据自动创建“深拷贝”。
>
> 因此，任何自动发生的拷贝都可以认为是成本较低的（就运行时性能而言）。


## Trait + 解引用 (_Deref_) {#trait-plus-解引用--deref}

通过实现 `Deref` trait, 可以自定义类型的解引用操作符 (_dereference operator_) `*` 的行为。

不仅如此，Rust 还支持隐式 Deref 强制转换 (_Deref Coercion_)。下面重点解释一下这一概念。


#### 隐式 Deref 强制转换的特点 {#隐式-deref-强制转换的特点}

-   Deref coercion 作用在函数和方法的参数上，可以自动将一种类型的引用转换为另一种类型的引用。要求被转换的类型实现了对应的 `Deref` trait。
-   Deref coercion 可以按需连续转换多次，以获得参数所需类型的引用。
-   Deref coercion 发生在编译期，因此没有额外的运行时开销（符合零成本抽象原则 _Zero
    Cost Abstractions_ ）。


#### 隐式 Deref 强制转换示例 {#隐式-deref-强制转换示例}

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

fn need_a_ref(x: &i32) {
    println!("{}", x);
}

fn hello(name: &str) {
    println!("Hello, {name}!");
}

fn main() {
    let x = 5;
    let y = MyBox::new(x);
    assert_eq!(5, *y);

    // 这里传参时发生了 Deref coercion, 将 &MyBox<i32> 自动转换成 &i32
    need_a_ref(&y);

    let m = MyBox::new(String::from("Rust"));
    // 下面两行是等价的
    hello(&m);                  // 使用了隐式 Deref 强制转换
    hello(&(*m)[..]);           // 未使用隐式 Deref 强制转换
}
```

```text
5
Hello, Rust!
Hello, Rust!
```


#### Deref 和 DerefMut 强制转换规则 {#deref-和-derefmut-强制转换规则}

在隐式转换中，如果原参数是可变引用 (`&mut`), 需要转换的目标参数也是可变引用，则必须实现 `DerefMut` trait 才能支持。

具体规则如下：

`&T` &rarr; `&U`
: 当 `T: Deref<Target=U>`

`&mut T` &rarr; `&mut U`
: 当 `T: DerefMut<Target=U>`

`&mut T` &rarr; `&U`
: 当 `T: Deref<Target=U>`


## Trait + 迭代器 {#trait-plus-迭代器}

为了说明 trait 在迭代器中的作用，我们先思考一个开发过程中常遇到的问题：

-   当我们有一个 `Result` 数组/列表时，如何快速判断这个 `Result` 列表里面是否存在错误？

你会怎么做呢？

当然，你可以遍历这个列表，然后逐个判断。但是 `Iterator.collect` 方法可以帮助我们更加优雅的做到这一点，并且更加的符合 Rustaceans 的习惯:

```rust
let results = vec![Ok(1), Err("nope"), Ok(3), Err("bad")];
let result: Result<Vec<_>, &str> = results.into_iter().collect();

// gives us the first error
assert_eq!(Err("nope"), result);

let results = [Ok(1), Ok(3)];

let result: Result<Vec<_>, &str> = results.into_iter().collect();

// gives us the list of answers
assert_eq!(Ok(vec![1, 3]), result);
```

类似地，也可以利用 `collect` 方法将一个 `Option` 列表转换成一个 `Option`:

```rust
let results = vec![Some(1), None, Some(3), None];
let result: Option<Vec<_>> = results.iter().cloned().collect();

// gives us the first None
assert_eq!(None, result);

let results = [Some(1), Some(3)];

let result: Option<Vec<_>> = results.iter().cloned().collect();

// gives us the list of answers
assert_eq!(Some(vec![1, 3]), result);
```

`collect` 方法是如何做到的呢？答案就在 `Result` 和 `Option` 这两个类型的 `FromIterator`
trait 的实现上。

`collect` 的行为取决于它的目标类型，具体来说，取决于目标类型的 `FromIterator` trait
的实现。不同的类型会实现自己独有的 `FromIterator` 逻辑，据此来定义如何从一个迭代器中的元素构建自己。

对于 `Result` 类型， `FromIterator` 被实现为：

-   如果迭代器中的元素都是 `Ok`, 则返回一个 `Ok`, 其中包含迭代器中所有 `Ok` 值的集合。
-   如果迭代器中存在任何 `Err`, 则返回第一个 `Err` 。

对于 `Option` 类型， `FromIterator` 的实现也是类似的，这里不再赘述。


## Trait + 错误处理 {#trait-plus-错误处理}


### 传播错误 (Propagating Errors) {#传播错误--propagating-errors}

Rust 采用传播错误（即函数返回值）的形式来处理“可恢复性错误” (_recoverable_ errors)，而不是异常机制。

下面是一个例子：

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let username_file_result = File::open("hello.txt");

    let mut username_file = match username_file_result {
        Ok(file) => file,
        Err(e) => return Err(e),
    };

    let mut username = String::new();

    match username_file.read_to_string(&mut username) {
        Ok(_) => Ok(username),
        Err(e) => Err(e),
    }
}
```

我们可以使用问号运算符 (question mark operator) `?` 来简化错误传播。上述代码等价于：

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut username = String::new();
    // 使用问号运算符，表达式匹配 Err 时立即执行 return
    File::open("hello.txt")?.read_to_string(&mut username)?;
    Ok(username)
}
```

除了在遇到错误时执行 early return, `?` 运算符还额外提供错误类型的自动转换功能。具体而言，如果发生的错误和函数的返回值声明中的错误类型不同，只要该错误类型实现了相应的 `From` trait, 则会进行自动转换。例如：

1.  返回的 Result 类型声明为 `Result<String, OurError>` （ `OurError` 为自定义的错误类型），函数体中返回了 `io::Error` 类型的错误。
2.  `OurError` 实现了 `impl From<io::Error> for OurError` 。
3.  `?` 运算符会自动执行 `from` 转换，将 `io::Error` 转换为 `OurError` 并返回。


### `main` 函数的返回值 {#main-函数的返回值}

`main` 函数可以返回两类值：

`Result<T, E>`
: 返回 `Ok<T>` 表示成功， `Err<E>` 表示失败。

`Termination` trait
: 该 trait 包含一个 `report` 方法，用来返回一个 [ExitCode](https://doc.rust-lang.org/std/process/struct.ExitCode.html)。


## <span class="org-todo todo TODO">TODO</span> Trait + 并发 {#trait-plus-并发}

`Send`, `Sync` 相关，待补充。
