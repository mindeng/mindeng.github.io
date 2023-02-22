+++
title = "设计模式：结构型 (Structural)"
date = 2023-07-07T18:35:00+08:00
lastmod = 2023-07-24T21:16:06+08:00
tags = ["design-pattern", "architecture"]
draft = false
+++

设计模式总目录请参考：[设计模式所支持的设计的可变方面]({{< relref "design-patterns#设计模式所支持的设计的可变方面" >}})。


## 适配器 (Adapter) {#适配器--adapter}


### 意图 {#意图}

> 将一个类的接口转换成客户希望的另外一个接口，使得原本不兼容的模块之间可以协同工作。


### 类图 {#类图}

{{< figure src="/ox-hugo/adapter-class.png" width="300" >}}


### 相关模式 {#相关模式}

和 [Bridge 桥接](#桥接--bridge) 有点类似，但是出发点不同：

-   Bridge 的目的是将接口部分和实现部分分离，从而可以对它们较为容易也相对独立地加以改变。
-   Adapter 意味着改变一个已有对象的接口。

[Decorator 装饰器](#装饰器--decorator) 在不改变接口的情况下，增强了其他对象的功能，因此 Decorator 对应用程序的透明性比较好，而且可以支持递归组合。

[Proxy 代理](#代理--proxy) 在不改变它的接口的条件下，为另一个对象定义了一个代理。


## 桥接 (Bridge) {#桥接--bridge}


### 意图 {#意图}

> 将抽象部分与它的实现部分分离，使它们可以独立地变化。


### 类图 {#类图}

{{< figure src="/ox-hugo/bridge-class.png" >}}

上图中 `Window` 和 `WindowImp` 之间就是 Bridge 的关系。


## 组合 (Composite) {#组合--composite}


### 意图 {#意图}

> 将对象组合成树形结构以表示“部分－整体”的层次结构。Composite 使得用户对单个对象和组合对象的使用具有一致性。


### 类图 {#类图}

{{< figure src="/ox-hugo/composite-class.png" >}}


## 装饰器 (Decorator) {#装饰器--decorator}


### 意图 {#意图}

> 动态地给一个对象添加一些额外的职责（功能）。

下面我们结合案例来阐述一下。


### 案例分析：持久化工具 {#案例分析-持久化工具}


#### 持久化工具 {#持久化工具}

假设我们有一个数据持久化工具，其功能很简单，就是写入持久化数据：

```python
# 持久化接口
class Serializer:
    def write(self, bs):
        pass
    def close(self):
        pass

# 基于文件的持久化实现
class FileBackendSerializer(Serializer):
    def __init__(self, path):
        self._file = open(path, 'wb')

    def write(self, bs)
        self._file.write(bs)

    def close(self):
        self._file.close()

# 基于服务器的持久化实现
class ServerBackendSerializer(Serializer):
    def __init__(self, address):
        self._server = self._connect(address)

    def write(self, bs):
        self._server.send(bs)

    def close(self):
        self._server.disconnect()

    # ...
```

如上图所示，客户通过 `Serializer` 接口来使用该功能，底层实现可能是基于文件的
`FileBackendSerializer`, 或者是基于服务器的 `ServerBackendSerializer`, 客户可以根据不同情况选用不同的持久化实现。

我们画一下类图：

{{< figure src="/ox-hugo/decorator-class1.png" >}}


#### 增加 gzip 压缩功能 {#增加-gzip-压缩功能}

某一天，我们发现持久化的数据量越来越大，因此希望能够给上述持久化工具增加 gzip 的支持，以减少磁盘空间占用。

当我们想为一个类扩展某想功能时，一种常见的做法是为其创建一个子类，并在子类中重写一些方法，来添加一些额外的功能。例如，在这个例子中，我们可以写一个
`FileBackendSerializer` 的子类 `GzipFileBackendSerializer`, 来为文件持久化类增加
gzip 的功能：

```python
import gzip

# 基于文件的持久化实现，同时支持 gzip 压缩功能
class GzipFileBackendSerializer(FileBackendSerializer):
    def __init__(self, path):
        self._file = open(path, 'wb')

    def write(self, bs)
        bs = gzip.compress(bs)
        self._file.write(bs)

    def close(self):
        self._file.close()
```

类似的，我们也可以为 `ServerBackendSerializer` 增加一个子类
`GzipServerBackendSerializer`, 为基于服务器的持久化类增加 gzip 功能。

此时，类图变成下面这样：

{{< figure src="/ox-hugo/decorator-class2.png" >}}


#### 提升数据安全性 {#提升数据安全性}

新需求又来了，我们希望为某些敏感数据提供加密功能，以便提升数据的安全性。和前面的
gzip 功能类似，我们也可以通过扩展子类来实现，扩展后的类图如下所示：

{{< figure src="/ox-hugo/decorator-class3.png" >}}


#### Gzip + 加密功能 {#gzip-plus-加密功能}

针对有些数据，我们既希望压缩，又希望加密，应该怎么处理呢？如果仍然按照创建子类的思路，我们的类图大概会发展成这样：

{{< figure src="/ox-hugo/decorator-class4.png" >}}

是不是感觉哪里不太对？子类数量越来越多，而且开始出现了不少冗余代码，例如
`SecureGzipFileBackendSerializer` 和 `SecureFileBackendSerializer` 的代码一定有不少冗余的成分， `SecureGzipServerBackendSerializer` 和 `SecureServerBackendSerializer`
也是如此。


#### 引入装饰者模式 {#引入装饰者模式}

有没有更好的方案，来解决上述这类问题？答案是用组合替代继承（参考[组合复用原则]({{< relref "design-patterns#composite-reuse-principle-组合复用原则" >}})），具体到这里，就是利用装饰器（Decorator）模式。

我们来演示一下，采用装饰器模式会如何实现 gzip 功能，以及加密功能：

```python

import gzip

# 增加 gzip 功能的 Serializer
class GzipSerializer(Serializer):
    def __init__(self, s: Serializer):
        self._s = s

    def write(self, bs):
        bs = gzip.compress(bs)
        self._s.write(bs)

# 增加加密功能的 Serializer
class SecureSerializer(Serializer):
    def __init__(self, s: Serializer):
        self._s = s

    def write(self, bs):
        bs = self._encrypt(bs)
        self._s.write(bs)
```

客户侧使用起来也非常简单：

```python
s = FileBackendSerializer('/path/to/file')

# 增加 gzip 功能
s = GzipSerializer(s)
# 增加加密功能
s = SecureSerializer(s)
```

上述的组合可以灵活搭配，例如，你可以为某些数据增加 gzip 功能，某些数据增加安全功能，另一些数据同时增加 gzip 和安全功能。

另外，配合 `ServerBackendSerializer` 使用也完全没有问题：

```python
s = ServerBackendSerializer('example.com')

# 增加 gzip 功能
s = GzipSerializer(s)
# 增加加密功能
s = SecureSerializer(s)
```

如果有其他的新增诉求，例如对数据做过滤，也只需按照类似的方式，增加一个
`FilterSerializer` 即可。这里的关键点是，在保持接口一致的前提下，通过组合的方式在原有的对象上包装（装饰）上一层新的功能。

我们看一下采用装饰器模式后的类图：

{{< figure src="/ox-hugo/decorator-class5.png" >}}

是不是干净清爽了很多？


### 相关模式 {#相关模式}

-   [适配器 (Adapter)](#适配器--adapter): Decorator 不同于 Adapter, 因为 Decorator 不改变对象的接口，而仅添加（或改变）对象的功能。
-   [组合 (Composite)](#组合--composite): 可以将 Decorator 看成是一个退化的、仅有一个组件的 Composite。然而这两者的目的不同，Decorator 的目的是为对象添加额外的功能，而非建立一个具有层次结构的对象聚合。
-   [策略 (Strategy)]({{< relref "design-patterns-behavioral#策略--strategy" >}}): 用一个装饰可以为对象添加额外的功能，而 Strategy 可以让你动态替换某种功能。这是改变对象的两种途径。


## 外观 (Facade) {#外观--facade}


### 意图 {#意图}

> 为子系统中的一组接口提供一个一致的界面，Facade 模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。

下面这个示意图，可以比较形象的表达 Facade 模式的意图：

{{< figure src="/ox-hugo/2023-07-07_17-23-28_screenshot.png" >}}


### 适用性 {#适用性}

-   当你要为一个复杂子系统提供一个简单接口时。

    子系统往往因为不断演化而变得越来越复杂，在使用大多数模式时，都会产生更多更小的类。这使得子系统更具可复用性，也更容易对子系统进行定制，但也给那些不需要定制子系统的用户带来一些使用上的困难。​**Facade 可以提供一个简单的缺省视图**, 这一视图对大多数用户来说已经足够，而那些需要更多的可定制性的用户可以越过 Facade 层。

-   当你需要构建一个层次结构的子系统时，​**使用 Facade 模式定义子系统中每层的入口点**​。如果子系统之间是相互依赖的，可以让它们仅通过 Facade 进行通信，从而简化它们之间的依赖关系。


### 相关模式 {#相关模式}

-   [抽象工厂 (Abstract Factory)]({{< relref "design-patterns-creational#抽象工厂--abstract-factory" >}}) 模式可以 Facade 模式一起使用以提供一个接口，该接口可以隐藏子系统对象的创建细节。
-   [中介者 (Mediator)]({{< relref "design-patterns-behavioral#中介者--mediator" >}}) 模式与 Facade 模式有些相似之处，它也抽象了一些已有类的功能。但是它们的目的不同，Mediator 主要是抽象对等对象之间的通信，这些对象知道
    Mediator 的存在。而子系统并不知道 Facade 的存在。
-   [单例 (Singleton)]({{< relref "design-patterns-creational#单例--singleton" >}}) 。通常仅需要一个 Facade 对象，这种时候可以考虑使用单例。


## 享元 (Flyweight) {#享元--flyweight}


### 意图 {#意图}

> 运用共享技术有效地支持大量细粒度的对象。


### 适用性 {#适用性}

当以下情况都成立时，可以使用 Flyweight 模式：

-   一个应用程序使用了大量的对象。
-   完全由于使用大量的对象造成很大的内存开销。
-   对象的大多数状态都可以变为外部状态。
-   如果删除对象的外部状态，那么可以用相对较少的共享对象取代很多组对象。
-   应用程序不依赖于对象标识。由于 Flyweight 对象可以被共享，所以两个逻辑上不同的对象，其物理上可能是同一个对象，因此应用程序不应该依赖对象标识的比较。


### 结构 {#结构}

{{< figure src="/ox-hugo/flyweight-class.png" >}}


### 相关模式 {#相关模式}

在实现 [State 状态]({{< relref "design-patterns-behavioral#状态--state" >}}) 模式和 [Strategy 策略]({{< relref "design-patterns-behavioral#策略--strategy" >}}) 模式时，如果涉及状态或策略较多的，可以考虑采用 Flyweight 模式来实现。


## 代理 (Proxy) {#代理--proxy}


### 意图 {#意图}

> 为其他对象提供一种代理以控制对这个对象的访问。


### 不同类型的代理 {#不同类型的代理}


#### 远程代理（Remote Proxy） {#远程代理-remote-proxy}

为一个对象在不同的地址空间提供局部代表。Android 的 `AIDL` 生成的 `Stub.Proxy` 类就是这样一种代理。


#### 虚代理（Virtual Proxy） {#虚代理-virtual-proxy}

按需创建开销较大的对象。

<!--list-separator-->

-  _Copy-on-write_ (COW) 优化

    这里拓展一下，还可以实现透明的 _copy-on-write_ 优化。拷贝一个庞大而复杂的对象是一种开销很大的操作，如果这个拷贝根本没有被修改，那么这些开销就没有必要。用代理延迟这一拷贝过程，我们可以保证只有当这个对象被修改的时候才对它进行拷贝。


#### 保护代理（Protection Proxy） {#保护代理-protection-proxy}

控制对原始对象的访问。保护代理用于对象应该有不同的访问权限的时候。


#### 智能引用（Smart Reference） {#智能引用-smart-reference}

取代简单的指针，它在访问对象时执行一些附加操作，典型用途包括：

-   对指向实际对象的引用计数，这样当该对象没有引用时，可以自动释放它（也称为 Smart Pointer）。
-   当第一次引用一个持久对象时，将它装入内存。
-   在访问一个实际对象前，检查是否已经锁定了它，以确保其他对象不能改变它。


### 结构 {#结构}

```text
Proxy

      +--------+                  +---------+
      | Client +----------------->+ Subject |
      +--------+                  +---------+
                                  | request |
                                  | ...     |
                                  +---------+
                                       ᐞ
           +---------------------------+
           |                           |
   +-------+-----+   realSubject  +----+----+
   | RealSubject +<---------------+ Proxy   |      +------------------------+
   +-------------+                +---------+      |  ...                   |
   |  request    |                | request +------+  realSubject.request() |
   |  ...        |                | ...     |      |  ...                   |
   +-------------+                +---------+      +------------------------+
```


### 相关模式 {#相关模式}

-   [Adapter 适配器](#适配器--adapter): 适配器为它所适配的对象提供一个不同的接口。相反，代理提供与它的实体相同的接口。
-   [Decorator 装饰器](#装饰器--decorator): 装饰的实现部分和代理有点类似，但是目的不同。装饰为对象添加一个或多个功能，而代理则控制对对象的访问。另外实现上虽然有相似之处，但还是有些细微的差异。例如，Remote Proxy 并不包含对实体的直接引用，而只是一个间接引用（例如 Android `AIDL` 的例子）。
