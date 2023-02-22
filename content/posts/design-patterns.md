+++
title = "软件设计原则、设计模式总结"
date = 2023-07-03T16:40:00+08:00
lastmod = 2023-07-24T21:16:07+08:00
tags = ["architecture", "design-pattern", "android"]
draft = false
+++
<!-- weight = 200 -->

## 前言 {#前言}

本文是笔者对软件设计原则、设计模式的一个梳理，很多内容参考自《设计模式：可复用面向对象软件的基础》一书（尤其是[设计模式](#设计模式)部分）。其中也包含了笔者个人的一些思考和总结。


## 概念和术语 {#概念和术语}

本章整理了一些容易混淆的概念和术语。


### 对象间关系 {#对象间关系}

先用一张 UML 图来直观展示一下：

{{< figure src="/ox-hugo/object-relationships.png" >}}

简单解释一下：

Inheritance
: 继承，这个很好理解，就是父类和子类的关系。

Composition
: 合成，这个是除了继承以外，两个对象之间所能有的最强的关系。合成意味着：

    -   `Eye` is a part of `Dog`, 即 `Eye` 是 `Dog` 不可分割的一部分。
    -   `Eye` 的生命周期完且由 `Dog` 控制， `Dog` 消失则 `Eye` 也不复存在。

    从实现上来说，一般 `Eye` 对象作为 `Dog` 的一个成员变量，且由 `Dog` 负责 `Eye` 的创建及其完整的生命周期管理。

Aggregation
: 聚合，关系强度次于 _Composition_ 。其含义如下：

    -   班级由全班学生组成，但学生不是班级不可分割的一部分（因为学生可以转学，但是班级会一直存在）。
    -   两者的生命周期也不需要完且一致。一方面，学生可以辍学或者转学，但班级不受影响；另一方面，班级也可以解散或重组（比如合并到其他班），但学生仍然存在。

    从实现上来说， `Student` 对象由 `Class` 对象所持有，且一般是通过 `Class` 的一个集合类成员变量所持有。

Association
: 关联，关系强度再次之。其仅仅意味着 _Has-A_ 的关系。单从对象持有的角度看，和 _Aggregation_ 的差别不大，我个人觉得主要的区别可能有如下几点：

    -   _Aggregation_ 会更强调“集体－个体”的关系一些，一般来说隐含了“一对多”的意思。
    -   _Association_ 是一种更加通用的“对象间持有关系”的描述，范围会比 Aggregation 更广。事实上， _Aggregation_ 可以看作是 _Association_ 的一种特例。

    从实现上来说， `VideoEditor` 会持有一个 `MediaCodec`, 一般是通过一个成员变量来持有。

Dependency
: 依赖，这个应该是强度最弱的一种关系了，仅表示两者之间存在依赖关系，但并不限定两者之间是如何依赖的。例如，最弱的一种依赖可能是在 A 对象所属的类的某个方法里面使用到了 B 类。正如图中的示例，视频编辑器在加载某张图片素材的时候，可能在 `loadImage()` 方法中使用了 `ImageDecoder`, 但并不会在成员变量中持有该对象的引用。

下面这张图展示了 _Association_, _Composition_, _Aggregation_ 三者之间的关系：

{{< figure src="/ox-hugo/relationships-venn.png" >}}


## 设计原则 {#设计原则}


### SOLID 原则 {#solid-原则}

**SOLID** 指面向对象设计的五个基本原则：

-   **S**​ingle-responsibility Principle 单一职责原则
-   **O**​pen-closed Principle 开闭原则
-   **L**​iskov Substitution Principle 里氏替换原则
-   **I**​nterface Segregation Principle 接口隔离原则
-   **D**​ependency Inversion Principle 依赖倒置原则

下面对这五个基本原则逐一进行介绍。


#### Single-responsibility Principle 单一职责原则 {#single-responsibility-principle-单一职责原则}

> _一个类应该只对一件事情负责。_

换句话说，就是​**一个类应该只有一个引起变化的原因**​。

我们知道，对现有代码进行修改是很容易引起问题的。如果一个类具有两个或更多的引起修改的原因，那么将来这个类变化的几率将会大大上升。而且当它真正被修改时，你设计中的两个或多个方面都会受到影响（取决于该类的职责数量），不可控因素会进一步提高。同时，职责过多也会增加后续维护人员的理解成本。

举个例子，比如设计模式中的[迭代器模式]({{< relref "design-patterns-behavioral#迭代器--iterator" >}})，就帮助我们把对集合的遍历操作这项职责给剥离出来，使得集合内部只需关心集合自身功能的实现，而无需操心如何遍历集合元素这项功能。相反，如果我们直接在集合内部实现迭代功能，那我们就给了这个类两个变化的原因：

1.  如果集合本身的功能（例如元素的存储结构或操作）发生改变，这个类会被修改；
2.  如果遍历的方式发生改变，这个类也会被修改。

因此，这个类将来被修改的机率大幅上升，增加了代码的不稳定因素。另外，由于这两项功能放在一起实现，彼此之间很可能会发生互相耦合，修改其中一项可能会导致另一项也需要修改，从而增加修改的复杂度和出错的概率。

设计模式中还有很多模式都遵守了单一职责原则，例如[抽象工厂]({{< relref "design-patterns-creational#抽象工厂--abstract-factory" >}})将产品对象的构造这一职责独立出来，客户的可以直接通过工厂接口拿到产品接口，而无需关心具体产品是如何实现以及如何实例化的；再比如[桥接模式]({{< relref "design-patterns-structural#桥接--bridge" >}})，将抽象的设计部分和它所倚赖的实现部分分离，使二者可以独立发生变化，等等。

另一个很有用的概念是分离关注点，这个概念和单一职责原则有点类似，但是角度不太一样。具体我们在[分离关注点](#separation-of-concerns-分离关注点)章节中进一步讨论。


#### Open-closed Principle 开闭原则 {#open-closed-principle-开闭原则}

> 系统应该对扩展开放，对修改关闭。

这里的关键词是“修改”。软件之所以要做设计，很大程度上就是因为需要应对未来的修改，换句话说，需要应对未来的变化。这种变化包括新增需求以及对现有需求的修改。

为了适应这种变化，并且在适应变化的同时，保持系统的健壮性，我们必须考虑系统在它的生命周期内会发生怎样的变化。一个不考虑变化的设计，在将来很可能需要大规模重构，这意味着重新设计、开发和测试，以及依赖方的修改，这种代价是十分巨大的。

坚持开闭原则不但能够帮助我们更好的适应变化，而且还有助于我们建立起稳定且高质量的『货架产品』。试想一下，如果我们的组件开发出来之后，不需要因为客户的需求而时常发生修改（只有日常维护、bugfix），在客户有新的需求时，都是通过扩展现有组件（继承或组合）、新增组件的方式来满足客户诉求，那么随着时间的推移，线上的场景验证、线下的测试覆盖会越来越多，现存组件的质量和稳定性会越来越高（因为 bugfix 在持续进行，且没有引入新的修改点）。这样持续发展下去，我们就可以建立一个高质量的组件库，即我们说的『货架产品』。

当然，要做到这一点并不容易。这里整理一个大致的思路供参考：

-   **找出潜在的变化点**​。在设计之初，从需求角度出发，考察模块中哪些部分将来可能会发生变化，把这些潜在的变化点找出来。

-   **对变化进行封装和隔离**​。变化点找出来之后，思考一下，将来这些变化真正来临的时候，我们会如何支持？

    是否可以在不修改现有模块代码的前提下（可以新增代码，例如新写一个类），通过某种机制优雅的支持这种变化？例如在运行时替换某个对象，或者新增一个子类来自定义基类的部分行为？想清楚这点之后，我们就实现了对变化进行封装和隔离。

上面是一个思路，具体应该如何做呢？ 设计模式为我们指明了道路。

​**设计模式可以确保系统能够以某种特定的方式发生变化**​，从而帮助你在面临这种变化时避免重新设计。每一个设计模式都允许系统结构的某个方面的变化独立于其他方面，这样产生的系统可以更好地适应这种变化，从而更加健壮。进一步的阐述见[常见的设计问题及相关模式应用](#常见的设计问题及相关模式应用)、[设计模式所支持的设计的可变方面](#设计模式所支持的设计的可变方面)两节。

一款不能适应变化的软件是没有生命力的，而且注定会以失败告终，让我们积极拥抱变化😏。


#### Liskov Substitution Principle 里氏替换原则 {#liskov-substitution-principle-里氏替换原则}

> 基类对象应该可以被子类对象无缝替换。

除了明显的字面意思，这里从『基类设计者』和『子类实现者』两个角度，补充一点个人的理解。

基类设计角度
: 基类在设计时，应该慎重定义可重写（ `overwrite` ）方法。每个 `overwrite` 方法都应该有明确的设计意图。

    基类定义的每一个 `overwrite` 方法，都应该是有意为之，不能随便定义。例如，模板方法中开放出来的 `overwrite` 方法，是有意让子类重写整个算法流程中的某些步骤。

    慎重定义 `overwrite` 方法，可以有效防止 `overwrite` 方法语义混乱、用途不明确以及子类错误重写的问题，也可以降低子类实现者的心智负担。


子类实现角度
: 子类在实现时，应该理解基类的工作机制，遵守基类的设计意图，严格按照继承协议来重写 `overwrite` 方法，并确保遵循里氏替换原则。


#### Interface Segregation Principle 接口隔离原则 {#interface-segregation-principle-接口隔离原则}

> 提供多个分离的接口，而非提供一个宽泛用途的接口。

提供隔离的接口至少有两方面的好处：

-   从使用者的角度讲，互相隔离接口的接口相较一个大而全的接口，使用起来更加简单、高效，可以有效减少误用，同时降低使用者的心智负担；
-   从设计者的角度讲，提供相互隔离的接口除了有利于保持组件接口的简洁清晰，同时还会迫使设计者思考清楚系统的核心（原子）接口是什么，从而在机制层面对系统的设计思考的更加透彻一些，而不是 case by case 的提供业务所需要的各项功能。

实际上，由于分离接口也意味着分离职责，因此该原则也暗合单一职责原则。


#### Dependency Inversion Principle 依赖倒置原则 {#dependency-inversion-principle-依赖倒置原则}

> 依赖抽象，不要依赖具体实现。

-   高层组件不应该依赖低层组件
-   不管高层组件或低层组件，两者都应该依赖抽象，而非具体实现

考虑这样一个例子，假设我们有一款视频编辑器，其中有一项贴纸功能，允许用户选择不同类型的贴纸，比如有静态贴纸、动态贴纸等。

多种贴纸类型意味着有多个贴纸的实现类，例如：

StaticSticker
: 支持单张图片的静态贴纸

AnimatedSticker
: 支持图片序列帧的动态贴纸

贴纸类型不同，使用方式也有所不同。例如，静态贴纸只需一张图片，以及贴纸的绘制区域、起始时间、结束时间；而动态贴纸需要一个序列帧，而且该序列帧的时间间隔可能是固定的，也可能是不固定的。如果在编辑器的主程序中直接使用这两个未经抽象的贴纸实现类，结果可能是灾难性的：

-   由于主程序需要关注具体的贴纸实现类，导致我们在主程序中引入了一个新的引起变化的原因（参考[单一职责原则](#single-responsibility-principle-单一职责原则)）,后续需要扩展新的贴纸，或者某个贴纸实现需要调整时，都可能会引起主程序的修改；
-   由于不同贴纸的使用逻辑（接口）可能不同，主程序中可能会充斥着各种 `if else` 或者
    `switch case` 的分支语句，在扩展新贴纸，或者调整贴纸工作流程的时候，会引发
    **Shotgun Surgery 霰弹式修改** (参考《重构》3.6 章)。

如何解决该问题？答案就是 **依赖倒置** 。我们应该依赖 Sticker 的抽象接口，而不能直接依赖不同贴纸的具体实现类。具体如何做到呢？下面提供一个思路供参考。

-   对贴纸进行抽象设计，得到贴纸接口 `Sticker`
    仔细分析贴纸的需求，我们发现大致可以定义出如下几个接口：

    -   `getStartTime()` - 获取起始时间
    -   `getEndTime()` - 获取结束时间
    -   `getRect()` - 获取绘制区域
    -   `getImageAtTime()` - 获取任意时刻的图片

    有了上述几个接口，主程序就能够实现基本的贴纸绘制流程了。因此，主程序就可以脱离对贴纸具体实现的依赖，转变成依赖抽象接口 Sticker 了。
-   有了 `Sticker` 之后，我们会发现其实还不太够。因为我们仍然需要在主程序中构造出具体的某个贴纸，不同贴纸的构造逻辑可能是不同的，而且后续可能会发生变化，因此，这仍然会导致对具体实现的依赖。如何摆脱这种依赖？我们进一步将对象的构造过程进行抽象，抽象出一个 `StickerFactory` 接口：
    -   `createSticker(): Sticker` 创建某种类型的贴纸，并返回 `Sticker` 接口

有了上述两个接口，主程序算是彻底摆脱了对贴纸具体实现类的依赖了，且双方都可以独立对实现进行调整，而不会互相产生影响。

也许你可能会问，那 `StickerFactory` 对象又如何构造呢？这个对象其实可以委托给贴纸选择程序来构造，也就是说，在用户选中某一款贴纸时，该贴纸的类型其实已经确定了，在这里可以恰当的构造出具体所需要的 `StickerFactory` （ `StaticStickerFactory` 或者
`AnimatedStickerFactory` 实例）。

这个例子讲完了，看起来是不是很眼熟？没错，这就是[抽象工厂模式]({{< relref "design-patterns-creational#抽象工厂--abstract-factory" >}})的一个应用。下面附上这个例子的类图，方便理解：

{{< figure src="/ox-hugo/sticker-class-diagram.png" >}}


### Separation of Concerns 分离关注点 {#separation-of-concerns-分离关注点}

> 分离关注点是指将软件划分为若干个彼此独立的单元，每个单元处理一个分离的关注点，不同单元之间保持隔离，仅通过定义良好的接口进行通信。

**Separation of Concerns** , 简称 **SoC** 。相比[单一职责原则](#single-responsibility-principle-单一职责原则)来说，我觉得分离关注点的概念会更通用化一些，更多强调了如何降低开发者的心智负担。例如，我在开发 A 模块的时候，可以不用操心其他 B/C/D 模块的任何细节，可以专注投入到 A 模块的开发工作上，这样效率最高，而且不容易出错。

另外， **SoC** 除了表达单一职责的内涵，还隐含了单个模块应该足够内聚，模块之间应该尽量解耦，只能通过定义明确的接口来通信等意思，否则是做不到分离关注点的。

这个概念可以涵盖到多个不同的层面，可以小到一个函数、一个类的划分，大到一个模块、一个子系统，也可以是软件的分层设计。例如 TCP/IP 协议模型，就是一个典型的符合 SoC
原则的设计。


### Law of Demeter 最少知识原则 {#law-of-demeter-最少知识原则}

> 也叫迪米特法则，是指一个实体应当尽量少地与其他实体之间发生相互作用，使得系统功能模块之间相对独立。这样，当修改某一个模块时，就会尽量少的影响到其他模块，扩展也会相对更加容易。

这其实是对软件实体之间的通信进行约束，其本质是要求我们在进行软件设计时，要做到实体内部的​**高内聚**, 以及实体之间的​**低耦合**​。

例如在 Android 开发过程中，如果是采用 _MvvM_ 架构，比较好的实践是尽可能的把你的
_Model_ 层逻辑，甚至是 _ViewModel_ 层逻辑做到平台无关，即 **保持对平台的最少知识（依赖)**,这样做至少有如下好处：

-   可以确保你尊循了 _MvvM_ 架构规范， _Model_ &amp; _ViewModel_ 层不会对 _View_ 层有直接的依赖。 _ViewModel_ 和 _View_ 之间仅保持数据发布/订阅的关系，不对 _View_ 产生直接依赖。在 Android 平台， 避免 _ViewModel_ 对 _View_ 层的依赖十分必要，这可以避免很多生命周期方面的问题（因为 _ViewModel_ 比 _View_ 层的对象往往具有更长的生命周期）。
-   你的 _Model_ &amp; _ViewModel_ 和平台无关，具备足够的灵活性。例如在 Android SDK 发生变更或者手机厂商行为不一致时，可以更容易的在上层做适配，而无需修改核心业务逻辑。
-   整个系统的可测试性将大幅提升， _Model_ &amp; _ViewModel_ 都可以进行独立的单元测试，而且这种​**单元测试是可以脱离 Android 设备或者 Android 模拟器独立进行**​的，即可以直接在 PC 上跑，测试、调试效率也得到了大幅提升。

关于 _ViewModel_ 的单元测试可以参考官方的[这篇 Codelab 示例](https://developer.android.com/codelabs/basic-android-kotlin-compose-test-viewmodel?hl=zh-cn#0)。


### Composite Reuse Principle 组合复用原则 {#composite-reuse-principle-组合复用原则}

> 多用组合(HAS-A)，少用继承(IS-A)。

采用组合的方式来实现新功能，有如下好处：

类之间的耦合更低
: 由于继承属于 **白箱复用** ,父类的内部细节对子类基本是可见的，这种复用方式在某种程度上破坏了父类的封装性，一旦父类的实现发生变化，子类很有可能面临修改。而组合属于 **黑箱复用** ,在复用时仅依赖其外部稳定接口，内部实现细节对客户来说是不可见的。因此，组合的方式明显具有更低的耦合性。

更加简单，不容易出错
: 继承需要对父类的工作机制有一定的了解，一旦对 `overwrite`
    方法（或属性）的设计意图产生错误理解，很容易导致难以预料的后果（参考[里氏替换原则](#liskov-substitution-principle-里氏替换原则)）。组合则相对简单一些，只需理解 `public` 接口即可。

不会产生庞大而不可控的继承体系
: 如果滥用继承，很容易导致一个庞大的继承体系，到最后没有人能真正搞懂整个系统是怎么工作的，改代码变得如履薄冰。而采用组合则不会有这个问题。

另外，组合相比继承需要的知识更少，这点和[最少小知识原则](#law-of-demeter-最少知识原则)也是相符的。


### 其他原则 {#其他原则}

这里列一些可能并不是大家所公认的，但我个人觉得做设计时应该放在心上的“原则”。


#### 优先考虑可测试性 {#优先考虑可测试性}

在做设计时，优先考虑可测试性。这样做有如下好处：

-   降低编写单元测试的难度。如果一开始没有考虑到可测试性，往往可能会导致后续编写单元测试的成本较高，甚至可能出现为了编写单元测试而不得不重构现有设计的情况。
-   有利于提高设计的灵活性，促进产生高内聚、低耦合的系统。如果一开始就考虑整个系统、模块的可测试性，往往可以启发我们做出更灵活的设计，并且促使我们设计出更具“高内聚、低耦合”特点的系统。


#### KISS 原则 {#kiss-原则}

_Keep It Simple and Stupid_.

尽可能让你的设计保持简洁易懂。我觉得主要有如下几个方面：

<!--list-separator-->

-  尽量避免引入新概念

    每个概念都有学习成本，应该尽可能复用软件工程中，或者行业领域内现有的概念体系。

<!--list-separator-->

-  避免过度设计

    设计是为了应对变化，对于不太可能发生变化（或者变化可控）的部分，应该尽量保持简单。

<!--list-separator-->

-  一件事情只保留一种最佳做法

    对于同一件事件，你的设计最好只保留一种达成的途径，而且是最佳的途径。

    尽量让用户少做选择，选择意味着成本，也意味着可能犯错。

    这点有时候可能和灵活性会有些冲突，需要对具体的需求做出权衡考虑。但至少可以在保留灵活性的同时，提供一些缺省的设置或工作模式，或者做一些分层设计，这样可以让那些不需要定制化功能的用户做最少的选择，就像[Facade 模式]({{< relref "design-patterns-structural#适用性" >}})中所做的那样。


## 设计模式 {#设计模式}


### 模式的分类 {#模式的分类}

根据模式的目的和范围，可以将设计模式大致划分为如下类别：

<!-- This HTML table template is generated by emacs/table.el -->
<table border="1">
  <tr>
    <td rowspan="2" align="left" valign="top">
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    </td>
    <td rowspan="2" align="left" valign="top">
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    </td>
    <td colspan="3" align="left" valign="top">
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;目的&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    </td>
  </tr>
  <tr>
    <td align="left" valign="top">
      &nbsp;创建型&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    </td>
    <td align="left" valign="top">
      &nbsp;&nbsp;结构型&nbsp;&nbsp;&nbsp;
    </td>
    <td align="left" valign="top">
      &nbsp;行为型&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    </td>
  </tr>
  <tr>
    <td rowspan="2" align="left" valign="top">
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;范围&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    </td>
    <td align="left" valign="top">
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;类&nbsp;&nbsp;
    </td>
    <td align="left" valign="top">
      &nbsp;Factory&nbsp;Method&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    </td>
    <td align="left" valign="top">
      &nbsp;&nbsp;Adapter&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    </td>
    <td align="left" valign="top">
      &nbsp;Interpreter&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;Template&nbsp;Method&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    </td>
  </tr>
  <tr>
    <td align="left" valign="top">
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;对象&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    </td>
    <td align="left" valign="top">
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;Abstract&nbsp;Factory&nbsp;<br />
      &nbsp;Builder&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;Prototype&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;Singleton&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    </td>
    <td align="left" valign="top">
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;Adapter&nbsp;&nbsp;&nbsp;<br />
      &nbsp;Bridge&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;Composite&nbsp;<br />
      &nbsp;Decorator&nbsp;<br />
      &nbsp;Facade&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;Flyweight&nbsp;<br />
      &nbsp;Proxy&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    </td>
    <td align="left" valign="top">
      &nbsp;Chain&nbsp;of&nbsp;Responsibility&nbsp;<br />
      &nbsp;Command&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;Iterator&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;Mediator&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;Memento&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;Observer&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;State&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;Strategy&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
      &nbsp;Visitor&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    </td>
  </tr>
</table>

按目的分类：

创建型
: 与对象的创建有关。

结构型
: 处理类或对象的组合。

行为型
: 对类或对象怎样交互、怎样分配职责进行描述。

按范围分类：

类模式
: 处理类和子类之间的关系，这些关系通过继承建立，是静态的，在编译时就确定下来了。

对象模式
: 处理对象之间的关系，这些关系在运行时是可以变化的，更具动态性。其实从某种意义上来说，几乎所有模式都使用继承机制，所以『类模式』只指那些集中于处理类间关系的模式，而大部分模式都属于对象模式的范畴。

创建型类模式将对象的部分创建工作延迟到子类，而创建型对象模式则将它延迟到另一个对象中。结构型类模式使用继承机制来组合类，而结构型对象模式则描述了对象的组装方式。行为型类模式使用继承描述算法和控制流，而行为型对象模式则描述了一组对象怎样协作完成单个对象所无法完成的任务。


### 常见的设计问题及相关模式应用 {#常见的设计问题及相关模式应用}

前面在[开闭原则](#open-closed-principle-开闭原则)中提到设计应该支持变化，下面介绍一些导致重新设计的一般原因，以及解决这些问题的设计模式：

1.  **通过显式指定一个类来创建对象**

    在创建对象时指定类名会使得你受到特定实现的约束，而不是特定接口的约束。要避免这种情况，应该间接地创建对象。

    设计模式：[Abstract Factory 抽象工厂]({{< relref "design-patterns-creational#抽象工厂--abstract-factory" >}}), [Factory Method 工厂方法]({{< relref "design-patterns-creational#工厂方法--factory-method" >}}), [Prototype 原型]({{< relref "design-patterns-creational#原型--prototype" >}})。

2.  **对特殊操作的依赖**

    当你为请求指定一个特殊的操作时，完成该请求的方式就固定下来了。为了避免把请求代码写死，你应该在编译时或者运行时支持对这种请求的响应方式进行修改。

    设计模式：[Chain of Responsibility 责任链]({{< relref "design-patterns-behavioral#责任链--chain-of-responsibility" >}}), [Command 命令]({{< relref "design-patterns-behavioral#命令--command" >}})。

3.  **对硬件和软件平台的依赖**

    外部的操作系统接口和应用编程接口（API）在不同的软硬件平台上是不同的。依赖于特定平台的软件将很难移植到其他平台上，甚至很难跟上本地平台的更新。所以设计系统时限制其平台相关性就很重要了。

    设计模式：[Abstract Factory 抽象工厂]({{< relref "design-patterns-creational#抽象工厂--abstract-factory" >}}), [Bridge 桥接]({{< relref "design-patterns-structural#桥接--bridge" >}})。

4.  **对对象表示或实现的依赖**

    知道对象怎样表示、保存、定位或实现的客户在对象发生变化时可能也需要变化。对客户隐藏这些信息能阻止连锁变化。

    设计模式：[Abstract Factory 抽象工厂]({{< relref "design-patterns-creational#抽象工厂--abstract-factory" >}}), [Bridge 桥接]({{< relref "design-patterns-structural#桥接--bridge" >}}), [Memento 备忘录]({{< relref "design-patterns-behavioral#备忘录--memento" >}}), [Proxy 代理]({{< relref "design-patterns-structural#代理--proxy" >}})。

5.  **算法依赖**

    算法在开发和复用时常常被扩展、优化和替代。依赖于某个特定算法的实体在算法发生变化时不得不变化。因此有可能发生变化的算法应该被独立出来。

    设计模式：[Builder 生成器]({{< relref "design-patterns-creational#生成器--builder" >}})，[Iterator 迭代器]({{< relref "design-patterns-behavioral#迭代器--iterator" >}})，[Strategy 策略]({{< relref "design-patterns-behavioral#策略--strategy" >}})，[Template Method 模板方法]({{< relref "design-patterns-behavioral#模板方法--template-method" >}})，[Visitor 访问者]({{< relref "design-patterns-behavioral#访问者--visitor" >}})。

6.  **紧耦合**

    紧耦合的类很难独立地被复用，因为它们是互相依赖的。紧耦合产生单块的系统，要改变或删掉一个类，你必须理解和改变其他许多类。这样的系统是一个很难学习、移植和维护的密集体。

    松散耦合提高了一个类本身被复用的可能性，并且系统更易于学习、移植、修改和扩展。设计模式使用抽象耦合和分层技术来提高系统的松散耦合性。

    设计模式：[Abstract Factory 抽象工厂]({{< relref "design-patterns-creational#抽象工厂--abstract-factory" >}})，[Command 命令]({{< relref "design-patterns-behavioral#命令--command" >}})，[Facade 外观]({{< relref "design-patterns-structural#外观--facade" >}})，[Mediator 中介者]({{< relref "design-patterns-behavioral#中介者--mediator" >}})，[Observer 观察者]({{< relref "design-patterns-behavioral#观察者--observer" >}})， [Chain of Responsibility 责任链]({{< relref "design-patterns-behavioral#责任链--chain-of-responsibility" >}})。

7.  **滥用继承**

    通过定义子类来扩充功能是一种比较笨拙的方式，而且极容易导致子类数量爆炸。定义子类还需要对父类有深入的了解，成本较高、容易犯错且耦合紧密。一旦出现两个维度的定制化信息，极容易导致子类数量爆炸，从而导致整个系统变得难以维护。

    对象组合技术是继承之外构建新功能的另一种灵活方法。新的功能可以通过以新的方式组合已有对象来实现。另一方面，过多使用对象组合也可能会导致设计难以理解。因此，许多设计模式往往会将两者结合起来，例如定义一个子类，并将它的实例和已存在实例进行组合来引入定制的功能。

    设计模式：[Bridge 桥接]({{< relref "design-patterns-structural#桥接--bridge" >}})，[Chain of Responsibility 责任链]({{< relref "design-patterns-behavioral#责任链--chain-of-responsibility" >}})，[Composite 组合]({{< relref "design-patterns-structural#组合--composite" >}})，
    [Decorator 装饰器]({{< relref "design-patterns-structural#装饰器--decorator" >}})，[Observer 观察者]({{< relref "design-patterns-behavioral#观察者--observer" >}})，[Strategy 策略]({{< relref "design-patterns-behavioral#策略--strategy" >}})。

8.  **不能方便地对类进行修改**

    有时你不得不改变一个难以修改的类。也许这个类不属于你维护，你没有源代码，或者对类的修改会导致很多其他依赖方的改动。设计模式提供了在这些情况下对类进行修改的方法。

    设计模式：[Adapter 适配器]({{< relref "design-patterns-structural#适配器--adapter" >}})，[Decorator 装饰器]({{< relref "design-patterns-structural#装饰器--decorator" >}})，[Visitor 访问者]({{< relref "design-patterns-behavioral#访问者--visitor" >}})。


### 设计模式所支持的设计的可变方面 {#设计模式所支持的设计的可变方面}

| 目的 | 设计模式                                                                                                  | 可变的方面                   |
|----|-------------------------------------------------------------------------------------------------------|-------------------------|
| 创建 | [抽象工厂 (Abstract Factory)]({{< relref "design-patterns-creational#抽象工厂--abstract-factory" >}})     | 产品对象家族                 |
|    | [生成器 (Builder)]({{< relref "design-patterns-creational#生成器--builder" >}})                           | 如何创建一个组合对象         |
|    | [工厂方法 (Factory Method)]({{< relref "design-patterns-creational#工厂方法--factory-method" >}})         | 被实例化的子类               |
|    | [原型 (Prototype)]({{< relref "design-patterns-creational#原型--prototype" >}})                           | 被实例化的类                 |
|    | [单例 (Singleton)]({{< relref "design-patterns-creational#单例--singleton" >}})                           | 一个类的唯一实例             |
| 结构 | [适配器 (Adapter)]({{< relref "design-patterns-structural#适配器--adapter" >}})                           | 对象的接口                   |
|    | [桥接 (Bridge)]({{< relref "design-patterns-structural#桥接--bridge" >}})                                 | 对象的实现                   |
|    | [组合 (Composite)]({{< relref "design-patterns-structural#组合--composite" >}})                           | 一个对象的结构和组成         |
|    | [装饰器 (Decorator)]({{< relref "design-patterns-structural#装饰器--decorator" >}})                       | 对象的职责,不生成子类        |
|    | [外观 (Facade)]({{< relref "design-patterns-structural#外观--facade" >}})                                 | 一个子系统的接口             |
|    | [享元 (Flyweight)]({{< relref "design-patterns-structural#享元--flyweight" >}})                           | 对象的存储开销               |
|    | [代理 (Proxy)]({{< relref "design-patterns-structural#代理--proxy" >}})                                   | 如何访问一个对象             |
| 行为 | [责任链 (Chain of Responsibility)]({{< relref "design-patterns-behavioral#责任链--chain-of-responsibility" >}}) | 响应请求的对象               |
|    | [命令 (Command)]({{< relref "design-patterns-behavioral#命令--command" >}})                               | 何时、怎样满足一个请求       |
|    | [解释器 (Interpreter)]({{< relref "design-patterns-behavioral#解释器--interpreter" >}})                   | 一个语言的文法及解释         |
|    | [迭代器 (Iterator)]({{< relref "design-patterns-behavioral#迭代器--iterator" >}})                         | 如何遍历、访问一个集合的各元素 |
|    | [中介者 (Mediator)]({{< relref "design-patterns-behavioral#中介者--mediator" >}})                         | 对象间怎样交互、和谁交互     |
|    | [备忘录 (Memento)]({{< relref "design-patterns-behavioral#备忘录--memento" >}})                           | 一个对象中哪些私有信息存放在该对象之外,以及何时进行存储 |
|    | [观察者 (Observer)]({{< relref "design-patterns-behavioral#观察者--observer" >}})                         | 多个对象依赖于另一个对象,而这些对象又如何保持一致 |
|    | [状态 (State)]({{< relref "design-patterns-behavioral#状态--state" >}})                                   | 对象的状态                   |
|    | [策略 (Strategy)]({{< relref "design-patterns-behavioral#策略--strategy" >}})                             | 算法                         |
|    | [模板方法 (Template Method)]({{< relref "design-patterns-behavioral#模板方法--template-method" >}})       | 算法中的某些步骤             |
|    | [访问者 (Visitor)]({{< relref "design-patterns-behavioral#访问者--visitor" >}})                           | 某些可作用于一组对象上的操作，且无需修改这些对象的类 |
