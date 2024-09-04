+++
title = "发布我的第一个 Crate: django-auth"
date = 2024-01-14T19:37:00+08:00
lastmod = 2024-09-04T18:39:13+08:00
tags = ["rust", "django"]
draft = false
+++

今天发布了我的第一个 crate: [django-auth](https://crates.io/crates/django-auth), 虽然是一个非常简单的 crate, 但麻雀虽小，五脏俱全，API 文档、测试用例、doc test 等一个都不能少 😎。

先简单介绍一下这个库，然后再介绍一下在 [crates.io](https://crates.io/) 上发布 crate 的流程。


## django-auth {#django-auth}

[Django](https://www.djangoproject.com/) 是一个 Python 的 web framework, 若干年前用曾经 Django 写过一些 Web 应用。最近刚好在学习 Rust, 因此计划用 Rust 将其中一个应用重写一遍练练手。

首先遇到的第一个问题，便是账号鉴权的问题：

1.  旧的应用使用的是 Django 的 auth 体系，数据库中保存的密码 hash 值由 Django 框架自动生成；
2.  如果不希望让所有用户重置密码（这样做会导致所有用户的旧密码失效），就需要在新写的 Rust 应用中兼容 Django 的 auth 体系；
3.  要想兼容 Django 的 auth 体系，就需要将 Django 生成的 auth 数据库表迁移出来，并且能够利用这些数据来验证用户密码。

经过上面这么一番分析，相信你应该知道这个 crate 是干什么的了。主要就两个功能：

-   验证用户输入的密码是否正确，对应 API： [django_auth](https://docs.rs/django-auth/latest/django_auth/fn.django_auth.html)
-   为新用户（或者当用户修改密码时），生成 Django 风格的 hashed password, 对应
    API: [django_encode_password](https://docs.rs/django-auth/latest/django_auth/fn.django_encode_password.html)

> 🌟 如果对 Django 的密码存储格式感兴趣，可以参考这个文档：[Password management in
> Django](https://docs.djangoproject.com/en/5.0/topics/auth/passwords/) 。

除了可以作为 lib 使用外，还附赠了一个 cli 工具，用来验证密码、对密码进行编码：

`$ cargo run --example auth`:

```text
Authenticate or generate Django-managed passwords

Usage: auth <COMMAND>

Commands:
  encode  Encode a password in Django-style
  verify  Verify a Django stored hashed password
  help    Print this message or the help of the given subcommand(s)

Options:
  -h, --help     Print help
  -V, --version  Print version
```

🚀 就这么简单！


## Crate 发布流程 {#crate-发布流程}

1.  创建 crate.io 账号

    打开网站 [crate.io](https://crates.io/)，点击右上角登录 crate.io。需要注意两点：

    -   目前只支持 github 账号登录。
    -   用于登录的 github 账号似乎要先拥有一个 Organization（我因为之前已经有
        Organization，所以不确定没有行不行）。

2.  获取 API token

    打开 [API Token](https://crates.io/settings/tokens) 页面，按照指示生成一个 API token, 复制 token, 然后到命令行运行：
    ```shell
    cargo login
    ```
    按照提示将 token 粘贴并回车即可。

3.  验证邮箱

    发布 crate 前需要先验证邮箱，否则会发布失败。打开 [profile](https://crates.io/settings/profile) 页，按照指示填入邮箱并验证即可。

4.  填写 package 信息

    发布前需要在 `Cargo.toml` 文件中的 `package` 段中填写如下信息：

    -   **license or license-file:** 版权信息。
    -   **description:** 库的一句话介绍。
    -   **homepage:** 主页。
    -   **documentation:** 文档链接（可选，不填则会自动使用该 crate 对应的 [docs.rs](https://docs.rs/)
        文档链接）。
    -   **repository:** 代码仓库。
    -   **readme:** readme 文件名（可选，如果有 README.md 文件，会自动使用）。

    更详细的信息可以参考 [官方的 Crate 发布指南](https://doc.rust-lang.org/cargo/reference/publishing.html)，也可以参考 django-auth 的
    [Cargo.toml](https://github.com/mindeng/django-auth/blob/main/Cargo.toml) 。

5.  发布 Crate

    package 信息确认后，可以在你的 crate 项目根目录，先运行如下命令检查一下发布流程：
    ```shell
    cargo publish --dry-run
    ```
    该命令不会真正执行发布操作，但是会把提交到 crate.io 之前要做的事情都做一遍。

    另外，可以执行如下命令，检查发布时将会打包上传的所有文件（注意检查是否包含一些可能泄漏私人信息的文件）：
    ```shell
    cargo package --list
    ```

    > ⚠️ \`cargo publish\` 命令会将当前目录下所有的 version controlled 文件 (即所有由版本管理器管理的文件) 都打包并上传。如果你有一些带有敏感信息的文件，例如密码、
    > token 等私人配置文件，请一定记得从版本管理器中忽略掉这些敏感文件（使用 git 的话可以在 \`.gitignore\` 中配置）。

    如果有些文件比较大，或者觉得没必要上传到 <https://crates.io/> （例如一些测试数据等），但确实需要进行版本管理的，可以在 \`Cargo.toml\` 文件中配置，例如：
    ```toml
    [package]
    exclude = [
    "testdata/*",
    ]
    ```
    该配置会忽略掉 \`testdata\` 目录下所有文件。

    确认无误后，可以运行如下命令，真正将 crate 发布至 crate.io:
    ```shell
    cargo publish
    ```
    ✅ 搞定！
