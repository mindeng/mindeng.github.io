+++
title = "设计模式：创建型 (Creational)"
date = 2023-07-07T18:35:00+08:00
lastmod = 2023-07-24T21:16:06+08:00
tags = ["design-pattern", "architecture"]
draft = false
+++

设计模式总目录请参考：[设计模式所支持的设计的可变方面]({{< relref "design-patterns#设计模式所支持的设计的可变方面" >}})。


## 抽象工厂 (Abstract Factory) {#抽象工厂--abstract-factory}


### 意图 {#意图}

> 提供一个接口以创建一系列相关或相互依赖的对象，而无须指定它们具体的类。


### 案例 {#案例}

前面在[依赖倒置原则]({{< relref "design-patterns#dependency-inversion-principle-依赖倒置原则" >}})中，其实已经举过一个贴纸的例子，其实就是抽象工厂模式的一个应用。

除了对 `Product` 和 `Factory` 进行抽象以外，抽象工厂方法还强调了​**产品系列**​的概念。比如《设计模式》一书中经典的例子，支持多种视感（look-and-feel）标准的用户界面，不同的视感风格为诸如滚动条、窗口和按钮等用户界面『窗口组件』定义不同的外观和行为。而某个特定视感风格下的一系列『窗口组件』，就是一个产品系列。我们不应该在 Motif 风格的窗口组件中，混入一个 PM 风格的窗口组件。

这里我们需要解决两个问题：

-   如何确保软件不依赖某种具体的视感风格，以保证软件的可移植性。就像前面的案例中，
    `Editor` 不应该直接依赖某个具体的 `StaticSticker` 一样；
-   如何确保我们不会在 Motif 视感中，错误的使用了一个 PM 风格的组件？

我们可以通过抽象工厂模式，优雅地解决上述两个问题。看一下类图就明白了：

{{< figure src="/ox-hugo/class-diagram-widget-factory.png" >}}

每一种视感标准都对应一个具体的 `WidgetFactory` 子类，客户只需要通过 `WidgetFactory`
即可创建出一组特定风格的窗口组件，无需关心哪些类实现了特定风格的窗口组件，而且可以保证绝对不会错误的混用不同风格的窗口，因为 `WidgetFactory` 强化了同一类型风格组件之间的绑定关系。


### 适用性 {#适用性}

-   一个系统希望独立于它的产品的创建、组合和实现。
-   一个系统存在多个产品系列，在工作时需要选择其中一个产品系列来使用。
-   比较强调产品系列的概念，同一个系列的产品应该配合在引起使用，不同系列的产品不能混用。
-   提供一个产品类库，但只想暴露它们的接口而不是实现。


### 优点 {#优点}

-   它分离了具体实现类
-   替换产品系列变得十分简单，只需替换一个 factory 即可
-   有利于产品的一致性，可以自动确保不同系列的产品之间不会混用


### 缺点 {#缺点}

该模式可以很轻易的扩展新的产品系列，但如果要扩展产品系列中的产品类型，例如上述案例中，增加一种新的窗口组件，会比较困难，因为会涉及 `AbstractFactory` 类及其所有子类的修改。

因此，使用该模式最好一开始就考虑清楚系统中有哪些产品类型，是否相对稳定，否则不太建议使用。


### 相关模式 {#相关模式}

AbstractFactory 类通常用[工厂方法 (Factory Method)](#工厂方法--factory-method)来实现，也可以用[原型(Prototype)](#原型--prototype)
来实现。

一个具体的工厂可以作为一个[单例 (Singleton)](#单例--singleton) 存在。


## 工厂方法 (Factory Method) {#工厂方法--factory-method}


### 意图 {#意图}

> 定义一个用于创建对象的接口，让子类决定具体将创建哪一种类型的实例，即对象的实例化过程被延迟到子类进行。

工厂方法相对比较简单，其实在[Abstract Factory 抽象工厂](#抽象工厂--abstract-factory)中就有工厂方法的运用。


### 类图 {#类图}

{{< figure src="/ox-hugo/factory-method-class.png" >}}


## 生成器 (Builder) {#生成器--builder}


### 意图 {#意图}

> 将一个复杂对象的构建与它的内部表示相分离，使得二者可以独立发生变化。

这里有两层含义：

-   构建过程（Director）和内部表示（Builder）相分离。同样的构建过程，可以生成不同的结果。例如，“解析并遍历 RTF 文档结构”这个过程是可复用的，但是我利用不同的
    `Builder`, 最终可以构建出不同格式的文档（ `Markdown`, `Html`, `Text` 等）。
-   构建过程不用关心产品的内部组成部分是如何创建的（具体由哪些类实例化），以及这些部分是如何组装的。因此，构建过程可以被独立修改。


### 结构 {#结构}


#### 类图 {#类图}

{{< figure src="/ox-hugo/builder-class.png" >}}


#### 时序图 {#时序图}

{{< figure src="/ox-hugo/builder-sequence.png" >}}


### 效果 {#效果}


#### 它使你可以改变一个产品的内部表示 {#它使你可以改变一个产品的内部表示}

`Builder` 对象为 `Director` 提供了一个构造产品的抽象接口，该接口隐藏了这个产品的表示和内部结构，同时也隐藏了该产品是如何装配的。因此，当你改变该产品的内部表示时，只需替换一个 `Builder` 即可。


#### 它使你可以在不同的 `Director` 中复用同一个 `Builder` {#它使你可以在不同的-director-中复用同一个-builder}

例如，你可以在不同的 RTF 文档格式中，复用同一个 `MarkdownBuilder` 。


#### 它使你对构造过程可以进行更精细的控制 {#它使你对构造过程可以进行更精细的控制}

`Builder` 模式和其他创建型模式很大的一个区别是， `Builder` 模式不是调用一个接口一下子就生成产品的，而是通过导向器一步一步构造，这个过程允许你有更高的自由度来定制整个产品。


### 相关模式 {#相关模式}


#### [抽象工厂 (Abstract Factory)](#抽象工厂--abstract-factory) {#抽象工厂--abstract-factory----orgad84e24}

Abstract Factory 与 Builder 的主要区别在于，Builder 模式着重于一步步构造一个复杂对象。而 Abstract Factory 着重于多个系列的产品对象。另外，Builder 在最后一步返回产品，而 Abstract Factory 会立即返回产品。


#### [访问者 (Visitor)]({{< relref "design-patterns-behavioral#访问者--visitor" >}}) {#访问者--visitor----posts-design-patterns-behavioral-dot-pre-processed-dot-md}

这两个模式在某些情况下可能存在竞争关系。例如，对于一个 RTF 来说，既可以用
Visitor 模式来遍历并导出不同格式的文档（参考[案例：富文本文档模型]({{< relref "design-patterns-behavioral#案例-富文本文档模型" >}})），也可以采用
Director + 不同 Builder 的方式来导出不同格式的文档。取决于你的文档模型是否适合采用 Visitor 模式。Visitor 模式是一种更加通用的为某一类集合元素增加不同操作的方法，其目的并非要构造对象。


## 原型 (Prototype) {#原型--prototype}


### 意图 {#意图}

> 通过克隆原型实例的方式来创建新的对象。


### 类图 {#类图}

{{< figure src="/ox-hugo/prototype-class.png" >}}


### 效果 {#效果}

Prototype 具备很多与其他创建型模式（例如 [抽象工厂](#抽象工厂--abstract-factory)）类似的效果，比如它对客户隐藏了具体的产品类，因此减少了客户知道的名字的数目，而且让客户无需修改即可切换到不同的具体产品。

除此之外，Prototype 有一些独有的特点，下面列举一些。


#### 运行时增加和删除产品 {#运行时增加和删除产品}

Prototype 可以很方便的在运行时通过注册原型实例来增加一个新的具体产品，这比抽象工厂模式相对会灵活一些。


#### 减少子类数量 {#减少子类数量}

使用抽象工厂会产生一个与产品类层次平行的 Factory 类层次，而 Prototype 不需要。


## 单例 (Singleton) {#单例--singleton}


### 意图 {#意图}

> 保证一个类仅有一个实例，并提供一个访问它的全局访问点。


### 实现 {#实现}


#### 饿汉式 {#饿汉式}

<!--list-separator-->

-  Java 实现

    ```java
    class Singleton {
        public static Singleton getInstance() {
            return instance;
        }

        // ...

        private Singleton() {}
        private static Singleton instance = new Singleton();
    }
    ```

<!--list-separator-->

-  Kotlin 实现

    ```kotlin
    object Singleton {
        // ...
    }
    ```


#### 双重检查锁 (Double-checked locking) {#双重检查锁--double-checked-locking}

<!--list-separator-->

-  Java 实现

    ```java
    class Singleton {
        public static Singleton getInstance() {
            if (instance == null) {
                synchronized (Singleton.class) {
                    if (instance == null) {
                        instance = new Singleton();
                    }
                }
            }
            return instance;
        }

        private Singleton() {}
        private volatile static Singleton instance;
    }
    ```

<!--list-separator-->

-  Kotlin 实现

    ```kotlin
    class Singleton private constructor() {
        companion object {
            val instance: Singleton by lazy(mode = LazyThreadSafetyMode.SYNCHRONIZED) {
                Singleton()
            }
        }
    }
    ```


#### 静态内部类 {#静态内部类}

这种方式利用 Java 虚拟机的类加载机制来实现懒加载的效果，并处理同步锁的问题。

<!--list-separator-->

-  Java 实现

    ```java
    class Singleton {
        public static Singleton getInstance() {
            return Holder.instance;
        }
        public static void hello() {
            System.out.println("hello");
        }

        static {
            System.out.println("Singleton has been loaded!");
        }

        private Singleton() {}
        private static class Holder {
            static {
                System.out.println("Holder has been loaded!");
            }
            static final Singleton instance = new Singleton();
        }

        public static void main(String[] args) {
            Singleton.hello();
            // 下面这句会触发 Holder 的加载，进而触发 instance 的初始化
            Singleton.getInstance();
        }
    }
    ```

<!--list-separator-->

-  Kotlin 实现

    ```kotlin
    class Singleton private constructor() {
        companion object {
            init {
                println("Singleton has been loaded!")
            }
            val instance get() = Holder.instance
            fun hello() {
                println("hello")
            }
        }

        private object Holder {
            init {
                println("Holder has been loaded!")
            }
            val instance = Singleton()
        }
    }

    Singleton.hello()
    // 下面这句会触发 Holder 的加载，进而触发 instance 的初始化
    Singleton.instance
    ```


### 劣势 {#劣势}

-   从可测试性的角度，应该尽量减少 Singleton 的使用。因为 Singleton 难以被 mock。可以考虑用[抽象工厂 (Abstract Factory)](#抽象工厂--abstract-factory)来代替 Singleton, 并在工厂内部返回其唯一实例。
-   由于单例对象无法被垃圾回收，会导致部分内存或其他资源长期被占据。如果单例是一个占用较多资源的复杂对象，这可能会造成资源紧张的问题。
