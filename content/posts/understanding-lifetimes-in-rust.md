+++
title = "理解 Rust 的 生命周期 (Lifetime)"
date = 2024-03-06T16:01:00+08:00
lastmod = 2024-03-06T16:10:53+08:00
tags = ["rust"]
draft = false
+++

## Lifetime 的主要目的是防止悬空引用 (_dangling references_) {#lifetime-的主要目的是防止悬空引用--dangling-references}

下面的例子中， _borrow checker_ 会检查 `r` 的生命周期 `'a` 比其引用的数据的生命周期 `'b`
要长，因此会拒绝编译通过。

```rust
// ❌ borrowed value does not live long enough
fn main() {
    let r;                // ---------+-- 'a
                          //          |
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
                          //          |
    println!("r: {}", r); //          |
}                         // ---------+
```


## Lifetime 是一种特殊的泛型参数 {#lifetime-是一种特殊的泛型参数}

具体到形式层面，Lifetime 实际上是一种特殊的泛型参数，这些泛型参数为编译器提供了有关引用之间如何相互联系的信息。

参考 [common-rust-lifetime-misconceptions](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md) 一文中的定义，可以加深对 Lifetime 的理解：

> 🌟 变量的生命周期是指它所指向的数据可以被编译器静态验证在其当前内存地址上有效的时间长度。
>
> A variable's lifetime is how long the data it points to can be statically
> verified by the compiler to be valid at its current memory address.


## 在函数中使用 Lifetime {#在函数中使用-lifetime}

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let string1 = String::from("long string is long");

    {
        let string2 = String::from("xyz");
        let result = longest(string1.as_str(), string2.as_str());
        println!("The longest string is: {}", result);
    }
}
```

上例中，由于 _borrow checker_ 无法推断出 x、y 的生命周期和返回值的生命周期之间的关系，因此必须通过生命周期参数来指定。该例中，返回值的生命周期和两个变量中生命周期较短的那个保持一致。

-   生命周期注解并不影响引用的生存时间，而是用于描述多个引用的生命周期之间的关系（主要是描述返回值和入参的生命周期之间的关系）。


## 在结构体中使用 Lifetime {#在结构体中使用-lifetime}

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("Could not find a '.'");
    let i = ImportantExcerpt {
        part: first_sentence,
    };
}
```

上例中， `ImportantExcerpt` 实例的生命周期不能超出其 `part` 字段的生命周期。


## Lifetime 参数的省略规则 (_Lifetime elision rules_) {#lifetime-参数的省略规则--lifetime-elision-rules}

对于一个函数来说：

1.  编译器为每个引用参数分配一个生命周期参数。
2.  如果只有一个输入生命周期参数，则将该生命周期分配给所有的输出生命周期参数。
3.  如果有多个输入生命周期参数，且其中一个是 `&self` 或 `&mut self` (即这是一个方法)，则将 `self` 的生命周期分配给所有的输出生命周期参数。

当引用没有显式生命周期注解时，编译器按照上述规则来计算引用的生命周期。如果上述三条规则走到底，仍然存在无法计算出生命周期的引用时，编译器会停止并报错。


## 理解 `&'static T` 引用 {#理解-and-static-t-引用}

请注意 `&'static T` 和 [T: 'static](#理解-t-static) 二者之间的区别。

`&'static T` 可以通过以下两种方式产生：

-   对静态变量的引用

    例如，字符串字面量因为存储在二进制文件中，在程序运行期间都有效，因此具有
    `'static` 生命周期。例如：
    ```rust
      fn main() {
          let str_literal: &'static str = "字符串字面量";
      }
    ```

-   通过 `Box::leak` 方法在运行时生成一个 `&'static T`, 下面将展开论述。


### 通过 `Box::leak` 生成 `&'static T` 引用 {#通过-box-leak-生成-and-static-t-引用}

```rustic
#[derive(Debug)]
struct A {
    s: String,
}

impl Drop for A {
    fn drop(&mut self) {
        println!("{:?} has been dropped!", self);
    }
}

fn leak_a() -> &'static A {
    let a = A {
        s: "hello".to_string(),
    };
    Box::leak(a.into())
}

fn main() {
    let a: &'static A = leak_a();
    println!("a = {:?}", a);

    // recover `a` Box from the leaked reference
    let a = unsafe {
        let const_ptr = a as *const A;
        let mut_ptr = const_ptr as *mut A;
        Box::from_raw(mut_ptr)
    };
    println!("a = {:?}", a);

    // `a` will be dropped here
}
```

```text
a = A { s: "hello" }
a = A { s: "hello" }
A { s: "hello" } has been dropped!
```

上面这个例子有两个要点：

-   通过 `Box::leak` 生成 `&'static T` 引用
-   通过 `Box::from_raw` 将引用恢复为一个 `Box` 对象（需结合 unsafe 代码）

关于这个主题的进一步的讨论，可以参考[我给 rust-blog 提的一个 PR](https://github.com/pretzelhammer/rust-blog/pull/73)。


## 在泛型中使用 Lifetime {#在泛型中使用-lifetime}


### 理解 `T: 'static` {#理解-t-static}

请注意和 [&amp;'static T 引用](#理解-and-static-t-引用) 之间的区别。

下面的读法有助于正确理解 `T: 'static`:

> 🌟 `T: 'static` 应读作： `T` 受到 `'static` 类型生命周期的约束。

`T: 'static`:

-   包括所有的 `&'static T`
-   也包括所有的 owned types, 因为 owner 可以确保数据一直有效。因此 `T: 'static` ：
    -   可以在运行时动态分配
    -   不需要在整个程序生命周期内有效
    -   可以安全、自由地修改
    -   可以在运行时被释放
    -   可以有不同持续时间的生命周期

<!--listend-->

```rustic
fn drop_static<T: 'static>(t: T) {
    std::mem::drop(t);
}

fn main() {
    let mut strings: Vec<String> = Vec::new();
    for i in 0..10 {
        strings.push(i.to_string());
    }

    // strings are owned types so they're bounded by 'static
    for mut string in strings {
        // all the strings are mutable
        string.push_str("a mutation");
        // all the strings are droppable
        drop_static(string); // ✅
    }
}
```


### 理解 `&'a T` 和 `T: 'a` {#理解-and-a-t-和-t-a}

-   `&'a T` 隐含了 `T: 'a`

    如果一个 `T` 的引用在 `'a` 内有效，那么 `T` 在这个周期内也必须有效，否则前者不成立。

-   `T:'a` 比 `&'a T` 更加通用和灵活
    -   前者可以接受 owned types (指非引用类型) 和引用

    -   后者只能接受引用

-   如果 `T: 'static`, 那么 `T: 'a`, 因为：
    -   前者读作：T 满足静态生命周期约束

    -   后者读作：T 满足 `'a` 生命周期约束

    -   静态生命周期 &gt;= `'a` 生命周期，因此上述结论成立

案例（参考自[common-rust-lifetime-misconceptions](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md))：

```rust
// only takes ref types bounded by 'a
fn t_ref<'a, T: 'a>(t: &'a T) {}

// takes any types bounded by 'a
fn t_bound<'a, T: 'a>(t: T) {}

// owned type which contains a reference
struct Ref<'a, T: 'a>(&'a T);

fn main() {
    let string = String::from("string");

    t_bound(&string); // ✅
    t_bound(Ref(&string)); // ✅
    t_bound(&Ref(&string)); // ✅

    t_ref(&string); // ✅
    t_ref(Ref(&string)); // ❌ - expected ref, found struct
    t_ref(&Ref(&string)); // ✅

    // string var is bounded by 'static which is bounded by 'a
    t_bound(string); // ✅
}
```
