+++
title = "设计模式：行为型 (Behavioral)"
date = 2023-07-07T18:36:00+08:00
lastmod = 2023-07-24T21:16:06+08:00
tags = ["design-pattern", "architecture"]
draft = false
+++

设计模式总目录请参考：[设计模式所支持的设计的可变方面]({{< relref "design-patterns#设计模式所支持的设计的可变方面" >}})。


## 责任链 (Chain of Responsibility) {#责任链--chain-of-responsibility}


### 意图 {#意图}

> 使多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系。将这些对象连成一条链，并沿着这条链传递该请求，直到有一个对象处理它为止。


### 案例 {#案例}

Android 的事件传递机制就是一个典型的责任链模式的应用。该应用结合了 [组合
(Composite)]({{< relref "design-patterns-structural#组合--composite" >}}) 模式，利用已有的视图树结构，将请求从视图树的根节点（DecorView）一直派发到各个子节点，直到某个视图处理该事件为止。


### 相关模式 {#相关模式}

-   [组合 (Composite)]({{< relref "design-patterns-structural#组合--composite" >}}): 可以利用已有的 Composite 结构来传递请求，形成责任链。


## 命令 (Command) {#命令--command}


### 意图 {#意图}

> 将某种请求封装为一个对象，换句话说，将请求参数化，这样可以解耦请求的创建方和实现方，也方便对请求进行排队、记录日志，以及支持撤销等操作。


### 结构 {#结构}

{{< figure src="/ox-hugo/command-class.png" >}}


### 用法介绍 {#用法介绍}


#### 实现菜单、按钮功能 {#实现菜单-按钮功能}

Command 模式特别适合用来实现菜单、按钮的功能。用 Command 模式实现有如下好处：

-   可以很方便的让一个菜单和一个按钮代表同一项功能，只需让二者共享同一个 Command
    对象即可。
-   可以很轻松的动态替换某项菜单或按钮的功能（例如实现上下文有关的菜单），只需动态替换 Command 对象即可。
-   还可以很方便将若干个命令组合成一个更大的命令，实现命令脚本（command scripting）功能。参考 [MacroCommand 宏命令](#macrocommand-宏命令)。


#### MacroCommand 宏命令 {#macrocommand-宏命令}

`MacroCommand` 是一个具体的 `Command` 子类，它用来执行一个命令序列。 `MacroCommand` 运用了[Composite 组合]({{< relref "design-patterns-structural#组合--composite" >}})模式来实现这种层次结构，参考下面的类图：

{{< figure src="/ox-hugo/macro-command-class.png" >}}


### 协作 {#协作}

{{< figure src="/ox-hugo/command-sequence.svg" >}}


## 解释器 (Interpreter) {#解释器--interpreter}


### 意图 {#意图}

> 给定一个语言，定义它的文法的一种表示，并定义一个解释器，这个解释器使用该表示来解释语言中的句子。


### 动机 {#动机}

如果一种特定类型的问题发生的频率足够高，那么可能就值得将该问题的各个实例表述为一个简单语言中的句子。这样就可以构建一个解释器，该解释器通过解释这些句子来解决该问题。


### 案例 {#案例}

搜索匹配一个模式的字符串是一个常见问题。正则表达式是描述字符串模式的一种标准语言。与其为每一个模式都构造一个特定的算法，不如使用一种通用的搜索算法来解释执行一个正则表达式。


## 迭代器 (Iterator) {#迭代器--iterator}


### 意图 {#意图}

> 提供一种方法顺序访问一个聚合对象中的各个元素，而又不需要暴露该对象的内部表示。


### 类图 {#类图}

迭代器模式应用广泛，原理也比较简单。下面是一个典型的类图。

{{< figure src="/ox-hugo/iterator-class.png" >}}


### 相关模式 {#相关模式}

-   [组合 (Composite)]({{< relref "design-patterns-structural#组合--composite" >}}): 迭代器可以被用来迭代组合模式中的对象。
-   [工厂方法 (Factory Method)]({{< relref "design-patterns-creational#工厂方法--factory-method" >}}): 如上面的类图所示，迭代器中利用了工厂方法来实例化对应的迭代器子类。
-   [备忘录 (Memento)](#备忘录--memento): 迭代器可使用 memento 来捕获一个迭代的状态。


## 中介者 (Mediator) {#中介者--mediator}


### 意图 {#意图}

> 用一个中介对象来封装一系列的对象交互。中介者使各对象不需要显式地相互引用，从而使其耦合松散，而且可以独立地改变它们之间的交互。


### 结构 {#结构}

{{< figure src="/ox-hugo/mediator-class.png" >}}


### 效果 {#效果}

减少了子类生成
: Mediator 将原本分布于多个对象间的行为集中在一起。改变这些行为只需要生成 Mediator 的子类即可，这样各个 Colleague 类可被复用。

将各 Colleague 解耦
: Mediator 有利于各 Colleague 间的松耦合，你可以独立地改变和复用各 Colleague 类和 Mediator 类。

简化了对象协议
: 用 Mediator 和各 Colleague 间的一对多交互来代替 Colleague 间的多对多交互。一对多的关系更易于理解、维护和扩展。

对对象如何协作进行了抽象
: 将中介作为一个独立的概念并将其封装在一个对象中，使你将注意力从对象各自本身的行为转移到它们之间的交互上来。这有助于弄清楚一个系统中的对象是如何交互的。

使控制集中化
: 中介者模式将交互的复杂性变为中介者的复杂性。因为中介者封装了协议，这会带来上述提到的好处，但也可能导致该对象变得比较复杂和难以维护。


### 相关模式 {#相关模式}

-   [外观 (Facade)]({{< relref "design-patterns-structural#外观--facade" >}}): Facade 模式与中介者的不同之处在于，它是对一个子系统进行抽象，从而提供一个更为方便的接口。它的协议主要是单向的，即通过 Facade 接口来访问子系统。相反，Mediator 提供了各 Colleague 对象不支持的协作行为，而且协议是多向的。
-   [观察者 (Observer)](#观察者--observer): Mediator 可以作为一个 Observer 来订阅各 Colleague 的状态变化，并做出响应（例如将状态变化的结果传播给其他的 Colleague）。


## 备忘录 (Memento) {#备忘录--memento}


### 意图 {#意图}

> 在不破坏封装性的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态。这样以后就可以将该对象恢复到原先保存的状态。


### 结构 {#结构}

{{< figure src="/ox-hugo/memento-class.png" >}}


#### Memento 备忘录 {#memento-备忘录}

-   备忘录存储原发器对象的内部状态。原发器根据需要决定备忘录存储原发器的哪些内部状态。
-   防止原发器以外的其他对象访问备忘录。备忘录实际上有两个接口，管理者（Caretaker）只能看到备忘录的 **窄接口** （只能将备忘录传递给其他对象），而原发器能够看到一个​**宽接口**, 允许它访问恢复到先前状态所需的所有数据。


#### Originator 原发器 {#originator-原发器}

-   原发器创建一个备忘录，用以记录当前时刻它的内部状态。
-   使用备忘录恢复内部状态。


#### Caretaker 管理者，例如“撤销机制” {#caretaker-管理者-例如-撤销机制}

-   负责保存好备忘录
-   不能对备忘录的内容进行操作或检查。


### 相关模式 {#相关模式}

-   [Command 命令](#命令--command): 命令可使用备忘录来为可撤销的操作维护状态。
-   [Iterator 迭代器](#迭代器--iterator): 备忘录可用于迭代器的实现，用于存储迭代器的当前状态。


## 观察者 (Observer) {#观察者--observer}


## 状态 (State) {#状态--state}


## 策略 (Strategy) {#策略--strategy}


## 模板方法 (Template Method) {#模板方法--template-method}


## 访问者 (Visitor) {#访问者--visitor}


### 意图 {#意图}

> 当我们遍历一个对象结构的元素时，该模式允许我们在不修改各个元素的类结构的前提下，为每个元素增加任意类型的新操作。


### 案例：富文本文档模型 {#案例-富文本文档模型}


#### 富文本文档模型 {#富文本文档模型}

假设我们要实现一个富文本文档模型，该模型的结构类似下图所示：

{{< figure src="/ox-hugo/visitor-page-tree.svg" >}}

对应的类图如下：

{{< figure src="/ox-hugo/visitor-page-class.png" >}}


#### 统计字符个数 {#统计字符个数}

我们希望为上述文档结构增加字符个数统计功能，一个简单的实现可能类似这样：

```python
class Element:
    # ...
    def char_count(self):
        raise NotImplementedError()

class Block(Element):
    # ...
    def char_count(self):
        num = 0
        for c in self.children:
            num += c.char_count()

class RichText(Element):
    # ...
    def char_count(self):
        return len(self.text)

class FileLink(Element):
    # ...
    def char_count(self):
        return 0

page = Page()
# ...
total_count = page.char_count()
print(f"total char count: {ccv.total_count}")
```

该实现会递归统计整个文档结构树，并返回最终的总和。

此时，类图变成类似这样：

{{< figure src="/ox-hugo/visitor-page-class2.png" >}}

可以看到，我们通过为基类 `Element` 增加一个方法 `char_count()`, 并在相应的子类中实现该方法，来实现字符个数的统计功能。


#### 文档导出 {#文档导出}

如果仅仅是增加一个字符个数统计功能，上述方法看起来并无大碍。接下来我们考虑一下，如果要增加文档导出功能，比如需要支持导出为如下不同格式的文档：

-   markdown
-   html
-   pdf
-   纯文本
-   ...

我们应该如何应对？如果延续上面的简单方案，我们可以在 `Element` 中增加类似下面的接口：

-   export_markdown()
-   export_html()
-   export_pdf()
-   export_text()
-   ...

如果我们希望再增加一个拼写检查的功能，可能还需要增加一个 `spellcheck()` 方法。这显然不是一个理想的方案，这会导致我们的类结构十分的不稳定，基类和子类的接口不断膨胀、不断引入新的变化点，既不符合[单一职责原则]({{< relref "design-patterns#single-responsibility-principle-单一职责原则" >}})，也不符合[开闭原则]({{< relref "design-patterns#open-closed-principle-开闭原则" >}})。

我们看一下这种方案下的类图：

{{< figure src="/ox-hugo/visitor-page-class3.png" >}}


#### 引入访问者模式 {#引入访问者模式}

如何在不修改 `Element` 及其子类结构的前提下，为我们的文档模型新增各种不同类型的操作呢？访问者模式可以帮助我们解决这类问题。

我们先看一下引入访问者模式之后，字符统计功能会怎么写：

```python
class Element:
    # ...
    def accept(self, v: Visitor):
        raise NotImplementedError()

class Block(Element):
    # ...
    def accept(self, v: Visitor):
        v.visitBlock(self)
        for c in self.children:
            c.accept(v)

class RichText(Element):
    # ...
    def accept(self, v: Visitor):
        v.visitRichText(self)

class FileLink(Element):
    # ...
    def accept(self, v: Visitor):
        v.visitFileLink(self)

class ListItemBlock(Block):
    # ...
    def accept(self, v: Visitor):
        v.visitListItemBlock(self)
        for c in self.children:
            c.accept(v)

# ...

class Visitor:
    def visitBlock(self, b: Block):
        pass
    def visitRichText(self, t: RichText):
        pass
    def visitFileLink(self, f: FileLink):
        pass
    def visitListItemBlock(self, f: FileLink):
        pass
    # ...

class CharCountVisitor(Visitor):
    def __init__(self):
        self.total_count = 0

    def visitRichText(self, t: RichText):
        self.total_count += len(t.text)

ccv = CharCountVisitor()
page.accept(ccv)
print(f"total char count: {ccv.total_count}")
```

看起来似乎变复杂了，但是通用性和扩展性得到了大幅提升。例如，如果想增加一个
markdown 导出功能，只需新增一个 `MarkdownExporterVisitor` 即可：

```python
class MarkdownExporterVisitor(Visitor):
    def __init__(self):
        self.content = []

    def visitRichText(self, t: RichText):
        c = t.text
        if t.font_weight == 'italic':
            c = f'*{t.text}*'
        elif t.font_weight == 'bold':
            c = f'**{t.text}**'
        # ...

        self.content.append(c)

    def visitListItemBlock(self, li: ListItemBlock):
        prefix = '- '
        if li.list_type == 'ordered_list':
            prefix = '1. '
        # handle indentation...

        self.content.append(prefix)

    # ...
```

新增其他文档格式的导出功能，以及拼写检查等功能也是类似，只需独立的增加一个类即可。这使得在新增一项对文档模型的操作时，我们的系统做到了“对修改关闭，对扩展开放”，而且每一项新增功能都很好的封装在了独立的新增类当中，而不是散落在文档结构的各个层次的类当中。

我们看一下使用访问者模式之后的类图：

{{< figure src="/ox-hugo/visitor-page-class4.png" >}}


### 适用性 {#适用性}

通过上述案例，我们可以看到，如果被访问的对象层级本身不太稳定（例如随时可能添加一种新的 `Element` ），那么可能不太适合使用 Visitor 模式。因为在这种情况下，会涉及对
`Visitor` 结构的修改（增加新的 visit 方法），从而导致 Visitor 继承层级的不稳定。

这里需要权衡的是，哪里发生变化的可能性比较大？如果对象层级本身变化的可能性比较大，则可能不适合使用 Visitor 模式；如果对象层级本身较稳定，而增加一种新的操作的可能性比较大，则比较适合使用 Visitor 模式。


### 相关模式 {#相关模式}

-   [组合 (Composite)]({{< relref "design-patterns-structural#组合--composite" >}}): Visitor 模式可以搭配 Composite 模式一起使用，用来对一个通过
    Composite 模式构建的对象层次结构进行遍历操作。上述的[富文本文档模型案例](#案例-富文本文档模型)，就是这样一个例子。
-   [解释器 (Interpreter)](#解释器--interpreter): 访问者也可以在解释器模式中使用。
