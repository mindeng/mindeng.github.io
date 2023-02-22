+++
title = "Rust 中的 | (竖线) 符号"
date = 2023-12-18T21:20:00+08:00
lastmod = 2023-12-19T20:46:40+08:00
tags = ["rust"]
draft = false
+++

Rust 中的 `|` 用途比较多，这里做一个简单的整理。


## 模式匹配中的“或”模式 (Pattern Alternatives) {#模式匹配中的-或-模式--pattern-alternatives}

在模式匹配（如 `match` 语句或 `if let` 表达式）中， `|` 可以用来表示多个模式的组合：

```rust
fn process_keypress(&mut self) -> Result<(), std::io::Error> {
    let pressed_key = read_key()?;

    if let Key::Ctrl('q') | Key::Ctrl('x') = pressed_key {
        self.should_quit = true;
    }

    // 和下面这句等价：
    match pressed_key {
        Key::Ctrl('q') | Key::Ctrl('x') => self.should_quit = true,
        _ => (),
    }

    // ...

    Ok(())
}
```


## 闭包参数 {#闭包参数}

在 Rust 的闭包中， 我们在两个 `|` 之间定义闭包的参数：

```rust
let add_one = |x| x + 1;
let result = add_one(5);
```

参数可以有多个，也可以留空。

严格意义来说，在闭包中的用法不是操作符，应该算是 Rust 语法的一部分。


## 位或 (Bitwise OR) 操作 {#位或--bitwise-or--操作}

这个比较简单，和其他语言中一致：

```rust
let a = 0b1010;
let b = 0b1100;
let c = a | b;
```


## 逻辑或 `||` 操作符 {#逻辑或-操作符}

这个也跟其他语言一样，用于连接两个布尔表达式：

```rust
let a = true;
let b = false;

if a || b {
    println!("至少一个条件为真");
} else {
    println!("两个条件都为假");
}
```


## 类型约束中的“或”约束 {#类型约束中的-或-约束}

在使用泛型和 _trait_ 时， `|` 可以用于指定类型必须实现多个 trait 中的任意一个（目前这个用法还在实验阶段，需要在 _Cargo.toml_ 中启用特定的特性）：

```rust
fn do_something<T: Display | Debug>(value: T) {
    // ...
}
```

在这个例子中， `T` 类型参数必须实现 `Display` 或 `Debug` trait 中的至少一个。
