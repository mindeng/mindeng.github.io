---
layout: post
title:  "Padding Oracle Attack"
date:   2016-05-13
tags:   [security, AES, CBC, padding, PKCS7, attack]
---

## 原理阐述

### Padding
在密码学中，由于底层加密算法往往是针对固定长度的块来设计的（例如 AES 的 CBC 模式的块大小为 16），所以在对可变长度的明文进行加密时，一般需要额外增加 padding 字段来满足块对齐以便进行加密。

Padding 方法可能有多种，为简单起见，这里只讨论常用的 PKCS7Padding 方法。在 PKCS7Padding 方法中，padding 字段填充的每个字节的值都是相同的，其值均为需要填充的字节个数，例如：如果块大小是 16， 那么明文 "aaaa" 的 padding 就是 [12]\*12。如果明文长度刚好是块大小的整数倍，则需要额外加上一个块的 padding，即 [16]\*16 。

PKCS7Padding 的 Python 实现代码如下：

```python
def pkcs7padding(plaintext):
    bs = 16
    padding_len = bs - len(plaintext) % bs
    return plaintext + padding_len * chr(padding_len)
```

### Padding Oracle
指被攻击对象暴露的一个接口，该接口可以通过任意形式提供，例如可能是命令行，可能是API，也可能是 REST 接口等等。形式不重要，重要的是内容，该接口可以输入密文，返回该段密文解密后的 Padding 是否正确。如果解密相关的应用开发者不小心，很有可能会暴露出该类接口（例如在提供 web 服务时，后端解密代码未对 Padding 异常情况进行捕捉，导致 HTTP 500 错误）。

Padding Oracle Attack 就是利用 Padding Oracle 来进行攻击的，简单来说，攻击者首先需要窃取一段密文，其次要有一个 Padding Oracle 可供利用，然后攻击者便可通过不断篡改密文并发送到 Padding Oracle 进行验证的方式，来对这段密文进行破解。

### XOR（异或）
在解释下一个概念之前，先让我们稍微复习下 XOR（异或）操作。下面是 XOR 的一些基本公式：

```
  A ⊕ A = 0
  A ⊕ 0 = A
  A ⊕ B = B ⊕ A
  (A ⊕ B) ⊕ C = A ⊕ (B ⊕ C)
  ∴ A ⊕ B ⊕ B = A ⊕ (B ⊕ B) = A ⊕ 0 = A
```

### Cipher-Block Chaining (CBC)
本文将阐述针对 AES CBC 模式的 Padding Oracle Attack，因此，有必要先解释下 CBC 模式的工作过程。

顾名思义，CBC 是针对块的加密，而且块与块之间是存在链式关系的（这种链式关系你很快就会理解了）。

#### 加密过程

在 CBC 中，有一个名叫 "block cipher"的东西，这个 "block cipher" 接受『块』作为输入，密文作为输出。本文不涉及 "block cipher" 的具体工作原理，我们可以把 "block cipher" 当作黑盒子来处理。

在 CBC 加密过程中，每个 "plaintext block" 在送入 "block cipher" 前，都需要先和它前面的 "ciphertext block" (即前面相邻的已加密的密文块) 进行 XOR 操作，XOR 的结果再送入 "block cipher"进行加密处理。这意味着每个加密出来的密文块都依赖于前面的明文加密后的结果，因此改变每一个明文字符都会对后面的加密结果产生巨大影响。这是一种被推荐的、比较安全的加密模式。

![CBC](/static/img/cbc.png)

(from [Wikipedia](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Cipher-block_chaining_.28CBC.29) )

#### 解密过程

类似的，在解密过程中，有一个 "block cipher decryption"，这个东东接受『密文块』作为输入，但输出不是明文，是一个中间结果。结合上面加密的过程，我们不难理解，这个中间结果就是明文在和 previous cipher block 异或之后的结果。拿到这个中间结果后，跟进前面异或公式的最后一条可知，只需将该中间结果和 previous cipher block 再做一次异或，即可得到最初的明文块。至此，该 block 解密结束。如果是第一个 cipher block，则将中间结果和 IV 进行一次异或即可得到明文块。

![CBC](/static/img/cbc2.png)

(图片来自 [Rob Heaton's blog](http://robertheaton.com/2013/07/29/padding-oracle-attack/) )

当然，如果是最后一个 block，还需要根据 PKCS7Padding 的规则将尾部 Padding 剔除，剔除 Padding 的 Python 实现代码如下：

```python
def pkcs7unpadding(plaintext):
    padding_len = ord(plaintext[-1])
    # check if the padding_len is valid
    # ...
    return plaintext[:-padding_len]
```

### 攻击过程
有了上述这些概念，攻击过程就比较好理解了。

攻击者就是通过计算上面提到的『中间结果』来达到解密目的的。怎么计算『中间结果』呢？答案是试！利用 Padding Oracle 来试。

首先，明确几个概念：

* 我们窃取了一段密文
* 我们是一块一块来破解的（因为 CBC 是一块一块加密的）
* 我们有一个 Padding Oracle

为了描述方便起见，这里定义几个缩写（可结合下面的图片来理解）：

* C1: 待破解密文块相邻的前面的密文块
* C2: 待破解密文块
* I2: 待破解密文块经过 "block cipher decryption" 解密出来的中间结果，可与 C1 异或后得到明文块
* P2: 待破解密文块解密后的明文块

因此我们有如下公式：

```
P2 = C1 ^ I2
```

C1 是已知的，因此我们的工作就是算出 I2 。

#### 破解最后一个字节
首先，我们从破解最后一个 block 的最后一个字节开始。

根据前面的定义，C2 是密文的最后一个 block，C1 是密文的倒数第二个 block 。看下破解过程：

![cbcfake](/static/img/cbcfake.png)

(图片来自 [Rob Heaton's blog](http://robertheaton.com/2013/07/29/padding-oracle-attack/) )

1. 先把 C1 替换成 [0]\*16 （即16个0），把新的 C1 叫做 C1'
1. 把 C1' + C2 传入 Padding Oracle，成功则跳到第 4 步，否则继续
1. C1'[15] 自增（上限是 255），并重复第 2 步
1. 由于 C1' + C2 通过了 Padding 检查，可以确定 P2'[15] 一定是 1~16 中的一个值，这里我们先假定是1（可通过继续修改 C1' 来进一步确定 P2'[15] 的值，为了避免引入额外的复杂度，这里先不介绍如何确定 P2'[15] 的值，后面再详细介绍）。假设此时 C1'[15] 自增到了 94，则按照如下公式可计算出 I2[15]

```
I2     = C1'     ^ P2'
I2[15] = C1'[15] ^ P2'[15]
I2[15] = 94  ^ 1
       = 95
```

进一步，因为 C1 是已知的，所以:

```
P2[15] = C1[15] ^ I2[15]
       = C1[15] ^ 95
```
至此，P2[15] 已经计算出来了，即最后一个字节已经被破解了！

#### 破解最后一个 block 的剩余字节
为了破解剩余字节，我们先可以把 P2'[15] 定位 2，根据 PKCS7Padding 的定义，如果 padding 是合法的，则 P2'[14] 也一定是 2。我们就根据这个思路来破解倒数第二个字节。

1. 先把 C1'[0..14] 置 0，C1'[15] 设置成 `2 ^ I2[15]` （确保 P2'[15] 是 2）
1. C1' + C2 传入 Padding Oracle，测试成功则跳到第 4 步，否则继续下一步
1. C1'[14] 自增，重复第 2 步
1. 根据以下公式计算 I2[15]:

```
I2     = C1'     ^ P2'
I2[14] = C1'[14] ^ P2'[14]
       = C1'[14] ^ 2
```

由于 C1 是已知的，所以：

```
P2[14] = C1[14] ^ I2[14]
```
至此，P2[14] 计算出来了，即倒数第二个字节被破解了！

重复上述步骤，即可破解最后一个 block 的剩余字节。

重复上面两节的步骤，即可破解所有 block。第一个 block 稍有不同，因为第一个 block 没有对应的 C1，此时， C1 = IV 。

#### 确定最后一个字节的值
我们回过头来看这个问题。

我们来分析一下，如果此时 P2'[15] 是2而不是1，意味着什么呢？根据 PKCS7Padding 的定义，这意味着：

```
P2'[14] == P2'[15] == 2
```

注意此时 C1'[14] 的值是 0，我们可以通过修改 C1'[14] 的值（例如修改成1），然后将 C1' + C2 传入 Padding Oracle 进行测试，如果通过则表示 P2'[15] 只可能是1。什么原因呢？因为 C1'[14] 的值被修改后，必然会影响 P2'[14]，而变化后的 P2' 仍能通过测试，说明 Padding 只能是 1，否则 P2'[14] != P2'[15] 是必然通不过测试的。如果修改后没有通过测试呢？此时说明 P2'[14] >= 2，那我们只需要重复以上步骤（把 C1'[14] 重置为0，修改 C1'[13] 的值为1）再做一遍 Padding 测试即可。如此最多测试 15 次，即可确定 P2'[15] 的值。步骤可归纳如下：

1. i = 14, C1'[i] = 1
1. PaddingOracle(C1' + C2) 是否成功？成功跳到第 4 步，否则继续
1. C1'[i] = 0, i -= 1, 如果 i < 0，则跳转第 5 步；否则 C1'[i] = 1，重复第 2 步
1. 根据 PKCS7Padding 的定义，可确定 P2'[15] 的值为 16 - i - 1，结束
1. 根据 PKCS7Padding 的定义，可确定 P2'[15] 的值为 16，结束

## 程序实现
作为练习和测试，这里用 Python 语言实现了一个 Padding Oracle Attack 的 Demo，见 https://github.com/mindeng/crypto-utils/tree/master/padding-oracle-attack 。

## 参考

* [The Padding Oracle Attack - why crypto is terrifying](http://robertheaton.com/2013/07/29/padding-oracle-attack/)
* [Padding oracle attacks: in depth](https://blog.skullsecurity.org/2013/padding-oracle-attacks-in-depth)

第一篇文章写的比较生动易理解，图文配合的很好（本文图片就是引自这里），缺点是没有提到 P2'[15] 值的确定，甚至直接断定 P2'[15] == 1，不太严谨。经我个人测试是有可能出现其他值的。

第二篇文章写的比较严谨、比较形式化，但也比较枯燥，没那么形象生动。不过这篇文章提到了 P2'[15] 的不确定性，而且简单介绍了解决办法（在Backtracking一节中有简单说明），但未做详细阐述。
