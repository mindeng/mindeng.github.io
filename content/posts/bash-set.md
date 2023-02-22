+++
title = "Bash: set 命令用法介绍"
date = 2023-06-26T12:49:00+08:00
lastmod = 2023-06-26T14:47:47+08:00
tags = ["linux", "tools", "bash"]
draft = false
+++

_Bash_ 在执行脚本的时候，会创建一个新的 _shell_, 每个 _shell_ 都有自己独立的执行环境，这个环境会有一些默认行为，而这些默认行为可以通过 `set` 命令来修改。

这里介绍几种常用的 `set` 命令。

> 注： `set` 命令在不带参数执行时，会显示当前 shell 的所有环境变量、函数。


## `set -u` 遇到不存在的变量时报错，并停止执行 {#set-u-遇到不存在的变量时报错-并停止执行}

默认情况下， _bash_ 遇到不存在的变量时会忽略。为了让脚本更加严谨和安全，报错并停止执行是一种更好的选择。

<a id="code-snippet--set-u"></a>
```bash
set -u

echo $non_exist_var
```

Output:

```text
bash: line 4: non_exist_var: unbound variable
```


## `set -x` 运行命令之前，先打印命令本身 {#set-x-运行命令之前-先打印命令本身}

<a id="code-snippet--set-x"></a>
```bash
set -x

unameOutput="$(uname -s)"
echo $unameOutput
```

Output:

```text
++ uname -s
+ unameOutput=Darwin
+ echo Darwin
Darwin
```


## `set -e` 发生错误时及时终止脚本 {#set-e-发生错误时及时终止脚本}

默认情况下，脚本执行过程中如果出现运行失败的命令， _Bash_ 会继续执行后面的命令，而这往往是不太安全的行为（因为后面的命令很可能依赖前面命令的成功执行）。

<a id="code-snippet--set-e1"></a>
```bash
cd /non/exist/path
echo "do something dangerous like rm files"
```

Output:

```text
bash: line 2: cd: /non/exist/path: No such file or directory
do something dangerous like rm files
```

实践当中，为了避免该问题，我们可以利用逻辑运算符的短路行为来及时终止命令的执行：

<a id="code-snippet--set-e2"></a>
```bash
cd /non/exist/path || { echo "/non/exist/path is not found"; exit 1; }
echo "do something dangerous like rm files"
```

Output:

```text
bash: line 2: cd: /non/exist/path: No such file or directory
/non/exist/path is not found
```

这种手工处理的方式比较麻烦，而且容易遗漏。 `set -e` 命令可以比较好的解决这个问题，该命令设置后，一旦某个命令失败，会立即停止后续命令的执行。

<a id="code-snippet--set-e3"></a>
```bash
set -e

cd /non/exist/path
echo "do something dangerous like rm files"
```

Output:

```text
bash: line 4: cd: /non/exist/path: No such file or directory
```


## `set -o pipefail` 处理管道错误 {#set-o-pipefail-处理管道错误}

`set -e` 不适用于管道命令，例如：

<a id="code-snippet--pipefail1"></a>
```bash
set -e

cat /non/exist/path | echo hello
echo world
```

Output:

```text
hello
cat: /non/exist/path: No such file or directory
world
```

`set -o pipefail` 可以解决该问题：

<a id="code-snippet--pipefail2"></a>
```bash
set -eo pipefail

cat /non/exist/path | echo hello
echo world
```

Output:

```text
hello
cat: /non/exist/path: No such file or directory
```


## 总结 {#总结}

上述四个 `set` 命令可以按照如下方式一起设置：

```bash
# 写法一
set -euxo pipefail

# 写法二
set -eux
set -o pipefail
```

建议放在所有 _Bash_ 脚本开头。

另外也可以在执行 _Bash_ 脚本时，从命令行传入这些参数：

```bash
bash -euxo pipefail /path/to/script.sh
```


## 参考资料 {#参考资料}

-   [The Set Builtin (Bash Reference Manual)](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html)
-   [Bash 脚本 set 命令教程 - 阮一峰的网络日志](https://www.ruanyifeng.com/blog/2017/11/bash-set.html)
