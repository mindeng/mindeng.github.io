+++
title = "Rust 中的特征 (Trait)"
date = 2023-12-19T20:46:00+08:00
lastmod = 2024-01-01T06:17:40+08:00
tags = ["rust"]
draft = false
+++

## 概述 {#概述}

简单来说， _trait_ 是 Rust 中用来定义共享行为的抽象机制，和 Java 的 interface、
Swift 的 protocol 有点类似：

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

单纯从提供的功能和灵活性角度来看，相比之下，trait 会比 Java 的 interface 更灵活和强大一些，可能和 Swift 的 protocol 更接近一点。例如：

-   trait 和 protocol 都支持关联类型，而 interface 不支持。
-   Rust/Swift 允许为外部类型增加 trait/protocol 实现, 可以很方便的为外部类型扩展一些额外的方法，并满足协议的要求。而 Java 并不支持这点。
    -   Java 可以通过[装饰器 (Decorator)](https://mincodes.com/posts/design-patterns-structural/#%E8%A3%85%E9%A5%B0%E5%99%A8--decorator)模式或者继承来实现类似的能力。
    -   Rust 在该功能上有额外限制（[孤儿规则](#孤儿规则--orphan-rule)），而 Swift 似乎并没有，这也导致 Swift
        中类型的行为一致性更难得到保证。

Trait 是 Rust 中比较有意思的语法特性，在标准库、第三方库中广泛使用（例如
[Derivable Traits](#可派生的特征--derivable-traits) 中所列的）。结合其语法特性及在库中的广泛性，让 trait 具有了非常强的灵活性和实用价值，因此值得我们深入探究。


## Trait 的语法特点 {#trait-的语法特点}


### 孤儿规则 (Orphan Rule) {#孤儿规则--orphan-rule}

为类型实现 trait 有一个限制：该类型和要实现的 trait 至少要有一个是在当前 crate
中定义的（crate 是 Rust 中的最小编译单元，参考[Packages and Crates](https://doc.rust-lang.org/book/ch07-01-packages-and-crates.html)）。

该限制是一致性 (_coherence_) 属性的一部分，叫做孤儿规则 (_orphan rule_)。该规则确保其他人的代码不会破坏你的代码，反之亦然。

例如，你无法为标准库中的 `IpAddr` 类型增加 `Iterator` trait 的实现（因为这二者都定义在外部 crate 中）：

```rust
use std::net::IpAddr;

// 编译失败❗
impl Iterator for IpAddr {
    type Item = u8;

    fn next(&mut self) -> Option<Self::Item> {
        return None
    }

}
```

编译器报告的错误如下：

```text
error[E0117]: only traits defined in the current crate can be implemented for types defined outside of the crate
 --> src/main.rs:5:1
  |
5 | impl Iterator for IpAddr {
  | ^^^^^^^^^^^^^^^^^^------
  | |                 |
  | |                 `IpAddr` is not defined in the current crate
  | impl doesn't use only types from inside the current crate
  |
  = note: define and implement a trait or new type instead

For more information about this error, try `rustc --explain E0117`.
error: could not compile `cargo0Pk8IQ` (bin "cargo0Pk8IQ") due to previous error
```

错误信息十分详尽，不仅解释了错误原因、指出了错误位置，还提供了解决方案和相关文档说明。

下面我们演示一下为标准库中的类型实现一个自己定义的 trait:

```rust
use std::net::{IpAddr, Ipv4Addr};

pub trait Openable {
    type Connection;

    fn open(&self) -> Option<Self::Connection>;
}

impl Openable for IpAddr {
    type Connection = String;

    fn open(&self) -> Option<Self::Connection> {
        Some(String::from("I'm connected!"))
    }
}

fn main() {
    if let Ok(localhost) = "127.0.0.1".parse::<IpAddr>() {
        if let Some(conn) = localhost.open() {
            println!("{conn}");
        }
    }
}
```

```text
I'm connected!
```

上面是程序运行的结果。


### Trait 中的默认实现 {#trait-中的默认实现}

和 Java 类似（Java 8 引入该特性），trait 在定义时可以提供方法的默认实现：

```rust
pub trait Summary {
    fn summarize_author(&self) -> String;

    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}
```

默认实现广泛存在于 Rust 标准库提供的 trait 中，为开发过程提供了很大的便利，并规范了一些编程的惯用法 (_idioms_)。


### 特征约束 (Trait Bound) {#特征约束--trait-bound}

Trait 可以和泛型编程很好的结合使用，可用于为泛型类型提供特征约束：

```rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

`impl Trait` 实际上是 _trait bound_ 的语法糖，上述代码和下面的代码等价：

```rust
fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
```


#### 多重特征约束 {#多重特征约束}

通过 `+` 操作符支持多重特征约束：

```rust
pub fn notify(item: &(impl Summary + Display)) {
}
```

等价于：

```rust
pub fn notify<T: Summary + Display>(item: &T) {
}
```


#### 在 `where` 子句中定义 Trait Bound {#在-where-子句中定义-trait-bound}

在泛型参数和约束较多时，这种方式相对会更加清晰一些：

```rust
fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{}
```


#### 使用特征约束有条件地实现方法 {#使用特征约束有条件地实现方法}

这个功能很有意思，可以为泛型的特定类型（实现了某些 trait 的类型）增加额外的方法定义：

```rust
use std::fmt::Display;

struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self { x, y }
    }
}

impl<T: Display + PartialOrd> Pair<T> {
    // 该方法仅在 T 实现了 Display + PartialOrd 时可用。
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("The largest member is x = {}", self.x);
        } else {
            println!("The largest member is y = {}", self.y);
        }
    }
}
```


#### 一揽子实现 (_Blanket Implementations_) {#一揽子实现--blanket-implementations}

这个功能很强大，在标准库中广泛使用，例如：

```rust
// 为实现了 Display trait 的任意类型实现 ToString trait
impl<T: Display> ToString for T {
    // --snip--
}
```

也就是说，一个类型只要实现了 `Display` trait, 便自动实现了 `ToString` trait（免费获得该接口）, 可以对其调用 `to_string` 方法（该方法由 `ToString` trait 定义）。因此，如果一个类型需要支持转换成 `String`, 我们一般实现 `Display` trait 即可。

Blanket implementations 也需要遵守[孤儿规则 (Orphan Rule)](#孤儿规则--orphan-rule)，而且情况会更加复杂一些。请参考下面这个错误示范：

```rust
struct Position {
    pub x: usize,
    pub y: usize,
}
enum Offset {
    Forward(usize),
    Backward(usize),
}
pub trait Document {
    // 获取一些文档的上下文信息
    // ...
}

// 尝试为实现了 Document trait 的任意类型实现 AddAssign trait
// 编译失败❗
impl<T: Document> std::ops::AddAssign<Offset> for T {
    fn add_assign(&mut self, rhs: Offset) {
    }
}
```

上述案例中， `AddAssign` 是标准库中定义的 trait, 属于外部类型，而 `T` 是一个泛型类型，意味着 `T` 可以是任意实现了 `Document` trait 的类型（包括定义在其他 crate 中的外部类型），因此，违反了孤儿规则的定义。


### 返回实现了特定 Trait 的类型 {#返回实现了特定-trait-的类型}

在返回类型中指定 trait 类型，可以对返回类型进行约束：

```rust
fn returns_summarizable() -> impl Summary {
    Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    }
}
```

由于泛型类型是编译时确定的，因此上述这种方式有个限制，就是不能在函数中的分支代码里，分别返回不同的具体类型。

如果需要支持返回多个不同的实现了某个 trait 的具体类型，需要使用 _trait object_
(`Box<dyn T>`, 参考
[Using Trait Objects
That Allow for Values of Different Types](https://doc.rust-lang.org/book/ch17-02-trait-objects.html)) 。


## 可派生的特征 (Derivable Traits) {#可派生的特征--derivable-traits}

_Derivable trait_ 指可以通过编译器自动实现的 trait。对于某些标准库中定义的 trait，
Rust 允许你在自定义类型上通过简单地添加一个属性（attribute）来自动实现这些 trait，而不需要手动编写实现代码。这个过程被称为 "派生"（deriving）。

使用可派生 trait 的主要优点是它减少了样板代码的数量，使得类型定义更加简洁。这对于提高代码的可读性和可维护性非常有帮助。

除了标准库提供的 derivable traits 外，第三方库也可以为自己的 traits 实现 `derive`,
因此，这个 derivable traits 列表是开放的。

下面列出了目前为止标准库提供的所有 derivable traits, 并对其使用要点进行简单说明。


### `Debug` {#debug}

用于支持字符串格式化中的 `{:?}` 占位符，主要用于 debug 打印。

特殊应用场景：

-   `assert_eq!` 宏要求其参数实现 `Debug` trait。

用法演示：

```rust
#[derive(Debug)]                // 自动派生 Debug trait 的实现
struct Position {
    x: u32,
    y: u32,
}

let p = Position{ x: 10, y: 20 };
println!("{:?}", p);
```

```text
Position { x: 10, y: 20 }
```

上面是程序的输出结果。


### `PartialEq`, `Eq` {#partialeq-eq}

-   用于支持 `==` 和 `!=` 操作符。
-   `Eq` 没有方法定义，只是一个指示，表明针对该类型的每个值，该值都等于其自身。
-   `Eq` trait 只能应用于实现了 `PartialEq` 的类型。

特殊应用场景：

-   `assert_eq!` 宏要求其参数实现 `PartialEq` trait。
-   `HashMap<K, V>` 要求 key 值实现 `Eq` trait。

用法演示：

```rust
#[derive(PartialEq, Eq, Debug)]
struct Rect {
    left: u32,
    right: u32,
    top: u32,
    bottom: u32,
}

let r1 = Rect {
    left: 0,
    right: 10,
    top: 0,
    bottom: 10,
};

let r2 = Rect {
    left: 0,
    right: 10,
    top: 0,
    bottom: 10,
};

dbg!(r1 == r2);
```

```text
[src/main.rs:25] r1 == r2 = true
```


### `PartialOrd`, `Ord` {#partialord-ord}

`PartialOrd` 和 `Ord` 用于支持类型的比较操作，可用于 `<`, `>`, `<=`, `>=` 这几个操作符。二者之间有如下区别：

-   `PartialOrd` 只能应用在实现了 `PartialEq` 的类型上
-   `Ord` 只能应用在实现了 `Eq` (从而也需要实现 `PartialEq` ) 的类型上
-   `PartialOrd` 返回的是 `Option<Ordering>` 类型，而 `Ord` 返回的是 `Ordering` 类型


#### `PartialOrd` 支持的可比较性是可选的 {#partialord-支持的可比较性是可选的}

比较有意思的是， `PartialOrd` 中定义的比较方法返回的是一个 `Option<Ordering>` 类型，也就是说，可以表达某些值之间不可比较的语意。

为了演示这个概念，下面杜撰了一个例子：

```rust
#[derive(PartialEq)]            // 自动派生 PartialEq trait 的实现
enum E {
    Man(u16),
    Dog(u16),
}

use E::{Dog, Man};

impl PartialOrd for E {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        return if let (Dog(me), Dog(other)) = (self, other) {
            // Dog 只和 Dog 相比较
            me.partial_cmp(other)
        } else if let (Man(me), Man(other)) = (self, other) {
            // Man 只和 Man 相比较
            me.partial_cmp(other)
        } else {
            // 其他情况不支持比较操作，比较时会直接返回 false
            None
        }
    }
}

let (man1, man2) = (Man(20), Man(30));
let (dog1, dog2) = (Dog(2), Dog(3));

dbg!(man2 > man1);
dbg!(dog2 > dog1);
dbg!(man1 > dog1);
dbg!(man1.partial_cmp(&dog1));
```

```text
[src/main.rs:29] man2 > man1 = true
[src/main.rs:30] dog2 > dog1 = true
[src/main.rs:31] man1 > dog1 = false
[src/main.rs:32] man1.partial_cmp(&dog1) = None
```

上面的输出说明：

-   `Man` 之间是可以进行比较的
-   `Man` 和 `Dog` 之间不可比，如果进行比较只会返回 `false` 或者 `None`, 取决于使用的是运算符还是方法调用。

总结一下，两个值之间的比较，遵循如下规则：

-   当且仅当 `partial_cmp(a, b) =` Some(Equal)= 时， `a == b`
-   当且仅当 `partial_cmp(a, b) =` Some(Less)= 时， `a < b`
-   当且仅当 `partial_cmp(a, b) =` Some(Greater)= 时， `a > b`
-   当且仅当 `a < b || a ​=`​= b= 时， `a <= b`
-   当且仅当 `a > b || a ​=`​= b= 时， `a >= b`
-   当且仅当 `!(a ​=`​= b)= 时， `a != b`


#### `Ord` 意味着任意两个值之间都存在有效的顺序 {#ord-意味着任意两个值之间都存在有效的顺序}

`Ord` 的用法和 `PartialOrd` 类似，只不过返回的直接就是一个 `Ordering`, 而非
`Option<Ordering>`, 这里不再举例说明。

特殊应用场景：

-   `BTreeSet<T>` 需要其存储的值实现 `Ord` trait。


### `Clone`, `Copy` {#clone-copy}

-   `Clone` 可用于实现值的深拷贝 (deep copy), 过程可能涉及对堆数据的拷贝。
-   实现 `Copy` trait 的类型​**必须同时实现 `Clone` trait** 。它们执行的是同样的任务，只是实现 `Copy` trait 意味着：
    1.  拷贝过程成本低，速度快（语意层面）。
    2.  赋值或传参时无需显式调用 `clone` 方法（语法层面）。
-   一个结构如果要 derive `Copy` trait, 要求其字段都必须实现了 `Copy` trait。

特殊应用场景：

-   调用 slice 的 `to_vec` 方法要求其存储的值实现 `Clone` trait。

在“[Trait 和生命周期](#trait-和所有权--ownership)”中我们会再次提到 `Copy` trait。


### `Hash` {#hash}

主要在哈希表中应用, `HashMap<K, V>` 要求 key 值实现 `Hash` trait。


### `Default` {#default}

`Default` trait 允许以一种惯用的方式创建类型的默认值。

特殊应用场景：

-   在结构体更新语法中使用： `..Default::default()`
-   `Option<T>.unwrap_or_default` 方法要求 `T` 类型实现 `Default` trait

下面演示了如何在结构体更新语法中使用 `Default` trait 提供的能力：

```rust
// 通过 derive 指令自动获得 Debug, Clone 和 Default trait
#[derive(Debug, Clone, Default)]
struct Rect {
    left: u32,
    right: u32,
    top: u32,
    bottom: u32,
}

let r1 = Rect {
    right: 10,
    bottom: 10,
    ..Default::default()
};

dbg!(r1);
```

```text
[src/main.rs:18] r1 = Rect {
    left: 0,
    right: 10,
    top: 0,
    bottom: 10,
}
```


## Trait 和类型转换 {#trait-和类型转换}


### `From` 和 `Into` {#from-和-into}

`From` 和 `Int` trait 规定了一种惯用的类型转换方式。实现了 `From`, 就可以“免费”获得
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

一个类型要转换成 `String`, 一般会实现 `fmt::Display` trait, 而不是 `ToString` (参考“[一揽子实现](#一揽子实现--blanket-implementations)”中的说明):

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


## Trait 和闭包 {#trait-和闭包}

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


## Trait 和所有权 (Ownership) {#trait-和所有权--ownership}

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

> Rust 永远不会为你的数据自动创建“深拷贝”。
>
> 因此，任何自动发生的拷贝都可以认为是成本较低的（就运行时性能而言）。


## Trait 和解引用 (Deref) {#trait-和解引用--deref}

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


## Trait 和迭代器 {#trait-和迭代器}

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


## Trait 和错误处理 {#trait-和错误处理}


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


### main 函数的返回值 {#main-函数的返回值}

main 函数可以返回两类值：

`Result<T, E>`
: 返回 `Ok<T>` 表示成功， `Err<E>` 表示失败。

`Termination` trait
: 该 trait 包含一个 `report` 方法，用来返回一个 [ExitCode](https://doc.rust-lang.org/std/process/struct.ExitCode.html)。


## 参考资料 {#参考资料}

-   [Traits: Defining Shared Behavior](https://doc.rust-lang.org/book/ch10-02-traits.html)
-   [Derivable Traits](https://doc.rust-lang.org/book/appendix-03-derivable-traits.html)
-   [Treating Smart Pointers Like Regular References with the Deref Trait](https://doc.rust-lang.org/book/ch15-02-deref.html)
-   [TryFrom and TryInto - Rust By Example](https://doc.rust-lang.org/rust-by-example/conversion/try_from_try_into.html)
