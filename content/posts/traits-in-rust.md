+++
title = "Rust 中的特征 (Trait)"
date = 2023-12-19T20:46:00+08:00
lastmod = 2024-07-03T17:54:27+08:00
tags = ["rust"]
draft = false
+++

## Trait 初探 {#trait-初探}

_trait_ 是 Rust 中用来定义共享行为的抽象机制，和 Java 的 _interface_, Swift 的
_protocol_ 等接口抽象机制有点类似。

定义一个 trait 很简单：

<a id="code-snippet--callable"></a>
```rust
trait Callable {
    fn call(&self);
}
```

为 Rust 的 `str` 类型实现该 trait (**impl**​ements `Callable` **for** `str`):

```rust
impl Callable for str {
    fn call(&self) {
        println!("call on {self}");
    }
}

"job-1".call();
```

```text
call on job-1
```

上面的代码为基本类型 `str` 扩展了一个 `call` 方法，语法上还是挺简洁、直观的。

这种为现有类型扩展 trait 实现的能力，除了可以应用在 Rust 的基本类型上，也可以应用在标准库、外部第三方库以及自定义的各种类型上，前提只要不违反 [孤儿规则 (Orphan
Rule)](#孤儿规则--orphan-rule) 即可。

Java 不支持这种能力，Kotlin 通过 _extension function_ 可以为现有类型扩展新方法（仅限于增加方法，不支持增加新的 interface 实现），而 Swift 是支持的。

当然，trait 的能力远不止于此，远比 interface/protocol 强大和复杂得多。下面我们来逐一探析 trait 的这些强大功能。


## Trait 的基本用法 {#trait-的基本用法}


### Rust 中的操作符定义 {#rust-中的操作符定义}

前面介绍了如何为外部类型扩展方法，当然也可以反过来，为自定义类型实现标准库中定义的 trait, 或者实现外部库中定义的 trait。

为自定义类型 `Offset` 扩展操作符 `+` 的实现（附带 += ​​实现）：

```rust
use std::ops::AddAssign;

#[derive(Debug, Copy, Clone, PartialEq)]
struct Offset {
    x: i32,
    y: i32,
}

impl AddAssign<i32> for Offset {
    fn add_assign(&mut self, v: i32) {
        *self = Self {
            x: self.x + v,
            y: self.y + v,
        };
    }
}

let mut offset = Offset { x: 1, y: 0 };
offset += 2;
assert_eq!(offset, Offset { x: 3, y: 2 });
```

> 🌟 _Tips_
>
> 上述代码说明了一个事实，即 Rust 中的操作符也是通过 trait 来定义的。因此，我们可以轻松通过实现 trait 来为自定义类型增加操作符的支持。用法也是标准的 trait 用法，并没有引入新的『操作符重载』的概念。


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


#### 使用特征约束有条件地实现方法 (Conditional APIs) {#使用特征约束有条件地实现方法--conditional-apis}

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
(`Box<dyn Trait>` 或 `&dyn Trait`, 参考 [Using Trait Objects That Allow for Values
of Different Types](https://doc.rust-lang.org/book/ch17-02-trait-objects.html))。


### 孤儿规则 (Orphan Rule) {#孤儿规则--orphan-rule}

为类型实现 trait 有一个限制：该类型和要实现的 trait 至少要有一个是在当前 crate
中定义的（crate 是 Rust 中的最小编译单元，参考 [Packages and Crates](https://doc.rust-lang.org/book/ch07-01-packages-and-crates.html)）。

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


## 静态分派 &amp; 动态分派 (Static dispatch &amp; Dynamic dispatch) {#静态分派-and-动态分派--static-dispatch-and-dynamic-dispatch}

Trait 支持两种分派方式，一种是静态的，即在编译期确定的分派方式；第二种是动态的，即在运行时确定如何分派。


### 静态派发的特点 {#静态派发的特点}

上面提到 trait 在泛型中的用法，都是编译期确定的，因此都属于静态派发。这种派发方式的特点如下：

-   编译期确定，没有运行时开销，无性能损失，即所谓的“零成本抽象” (_Zero-cost
    Abstraction_)。
-   针对每个具体类型，都会在编译期产生一个“副本”，这会在一定程度增加二进制文件尺寸，有点“以空间换时间”的意思。

    当然，即使不使用 trait + 泛型特性，自己手写代码也并不会比这个更小，这就是
    Stroustrup 所说的 _"What you do use, you couldn't hand code any better"_ 的意思。
-   支持函数调用的内联优化 (inline)。


### 动态派发的动机、用法 {#动态派发的动机-用法}

当涉及 _trait object_ (`&dyn Trait` 或 `Box<dyn Trait>`, 其中 `Trait` 表示某个 trait）时，就会出现动态分派。

动态派发的动机主要是希望实现面向对象语言中的多态功能，类似 C++ 的虚函数。

例如，假设我们要实现一个任务队列，队列中的任务希望足够抽象和通用，因此希望通过一个 trait 来进行约束，类似这样：

<a id="code-snippet--task"></a>
```rust
trait Task {
    fn do_job(&self);
}

struct TaskQueue {
    tasks: Vec<Box<dyn Task>>,
}
```

上述案例中，我们无法在编译期确定 `Vec` 中存储的具体类型，因此，静态分派显然已经无法满足我们的需求，只能使用 `Box<dyn Task>` 这类对象，从而引入动态分派。

有了这个定义，我们可以用一种统一的方式，对 tasks 中的任务进行操作，而无需关心
task 的具体类型：

```rust
impl TaskQueue {
    fn new() -> TaskQueue {
        TaskQueue {
            tasks: Vec::new(),
        }
    }

    fn add(&mut self, t: Box<dyn Task>) {
        self.tasks.push(t)
    }

    fn process(&self) {
        self.tasks.iter().for_each(|t| t.do_job());
    }
}

impl Task for &str {
    fn do_job(&self) {
        println!("do job: {self}");
    }
}

impl Task for i32 {
    fn do_job(&self) {
        println!("do job: {self}");
    }
}

let mut q = TaskQueue::new();
q.add(Box::new("task 1"));
q.add(Box::new(2));
q.process();
```

```text
do job: task 1
do job: 2
```


### 动态派发的实现原理和特点 {#动态派发的实现原理和特点}

trait 的动态派发的实现也和 C++ 的虚函数类似，借用了虚函数表 (vtable) 来进行动态派发：

-   trait object 存储了指向实现了该 trait 的类型实例的指针
-   trait object 存储了在该类型上查找 trait 方法的表（虚函数表）

有了以上两个信息，trait object 就可以在运行时确定具体应该调用哪个函数了。

了解了动态派发的实现原理，其特点也很明显了：

-   有额外的运行时开销（查表开销）
-   不会造成编译膨胀
-   不支持函数调用的内联优化 (inline)

Trait 的对两种派发方式的支持，也体现了 _pay as you go_ 的设计原则：当你需要更高级的抽象能力时，你可以使用动态派发；当你不需要时，trait 的抽象会在编译期被还原成具体类型，无需付出任何额外的代价。


## 可派生的 Trait (Derivable Traits) {#可派生的-trait--derivable-traits}

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

在“[Trait 和生命周期]({{< relref "rust-trait-plus-series#trait-plus-所有权--ownership" >}})”中我们会再次提到 `Copy` trait。


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


## Trait+ 系列 {#trait-plus-系列}

这个章节介绍了一系列 Rust 中利用 trait 实现的通用能力或惯用法，这些内容也是
Rust 编程中较常见的概念、方法和技巧，实用性很强，我称之为“Trait+ 系列”。

考虑到篇幅太长，本章节单独整理成文，请参阅： [Rust Trait+ 系列 ]({{< relref "rust-trait-plus-series" >}})。


## 总结 {#总结}

上面整理了一大堆 trait 相关的功能，看起来很复杂，实际上其底层却是一个统一、通用的概念。只需掌握 trait 这一套概念和用法，就可以类推到各个方面：

-   为外部类型扩展方法
-   实现“操作符重载”
-   泛型中的特征约束
-   面向对象的“多态”
-   类型转换
-   闭包
-   所有权
-   解引用
-   迭代器
-   错误处理
-   并发
-   ......

这种在底层概念和能力上的统一和复用，值得我们学习和借鉴。


## 参考资料 {#参考资料}

-   [Traits: Defining Shared Behavior](https://doc.rust-lang.org/book/ch10-02-traits.html)
-   [Trait objects](https://doc.rust-lang.org/book/ch17-02-trait-objects.html)
-   [Abstraction without overhead: traits in Rust](https://blog.rust-lang.org/2015/05/11/traits.html)
-   [Derivable Traits](https://doc.rust-lang.org/book/appendix-03-derivable-traits.html)
-   [Treating Smart Pointers Like Regular References with the Deref Trait](https://doc.rust-lang.org/book/ch15-02-deref.html)
