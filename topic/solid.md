#### Code review
有多少人将 code review 当做自己工作的一部分?[听演讲的人都举起了手]。为什么要做 code review？[一些人回答为了阻止不好的代码]
如果 code review 是为了捕捉到不好的代码，那么问题来了，你怎么判断你正在 review 的代码是好还是不好呢？

我们可以很容易的说出“这代码好辣眼睛”或者“这源码写的太吊了”，就像说“这画真美”，“这屋子真大气”一样。但是这些都是主观的，我希望找到一些客观的方法来衡量代码是好还是不好。

为什么要代码评审？
如果代码评审是要捕捉糟糕的代码，那么你如何知道你审查的代码是好的还是糟糕的？
我在找一些客观的方式来谈论代码的好坏属性。

#### Bad code 
What are some of the properties of bad code that you might pick up on in code review?
你可能会在代码审查中遇到以下这些糟糕的代码：
```
    Rigid . Is the code rigid? Does it have a straight jacket of overbearing types and parameters, that making modification difficult?
    代码是否死板？它是否有强类型或参数以至于修改起来很困难？
    Fragile . Is the code fragile? Does the slightest change ripple through the code base causing untold havoc?
    代码是否脆弱？对代码做轻微的改变是否就会引起程序极大的破坏？
    Immobile . Is the code hard to refactor? Is it one keystroke away from an import loop?
    代码是否很难重构？
    Complex . Is there code for the sake of having code, are things over-engineered?
    代码是否过于复杂，是否过度设计？
    Verbose . Is it just exhausting to use the code? When you look at it, can you even tell what this code is trying to do?
    代码是否过于冗长而使用起来很费劲？当查阅代码是否很难看出来代码在做什么？
```
Are these positive sounding words? Would you be pleased to see these words used in a review of your code?
Probably not.
当你做代码审查的时候是否会很高兴看到这些词语？
当然不会。
Good design
But this is an improvement, now we can say things like “I don't like this because it's too hard to modify”, or “I don't like this because i cannot tell what the code is trying to do”, but what about leading with the positive?

Wouldn't it be great if there were some ways to describe the properties of good design, not just bad design, and to be able to do so in objective terms?
好的设计
如果有一些描述优秀的设计属性的方式就更好了，不仅仅是糟糕的设计，是否能在客观条件下做？

SOLID
In 2002 Robert Martin published his book, Agile Software Development, Principles, Patterns, and Practices . I n it he described five principles of reusable software design, which he called the SOLID principles, after the first letters in their names.
在2002年，Robert Martin的Agile Software Development, Principles, Patterns, and Practices 书中提到了五个可重用软件设计的原则 - "SOLID"（英文首字母缩略字）
```
Single Responsibility Principle - 单一功能原则https://zh.wikipedia.org/wiki/%E5%8D%95%E4%B8%80%E5%8A%9F%E8%83%BD%E5%8E%9F%E5%88%99
Open / Closed Principle - 开闭原则
https://zh.wikipedia.org/wiki/%E5%BC%80%E9%97%AD%E5%8E%9F%E5%88%99
Liskov Substitution Principle - 里氏替换原则
https://zh.wikipedia.org/wiki/%E9%87%8C%E6%B0%8F%E6%9B%BF%E6%8D%A2%E5%8E%9F%E5%88%99
Interface Segregation Principle - 接口隔离原则
https://zh.wikipedia.org/wiki/%E6%8E%A5%E5%8F%A3%E9%9A%94%E7%A6%BB%E5%8E%9F%E5%88%99
Dependency Inversion Principle - 依赖反转原则
https://zh.wikipedia.org/wiki/%E4%BE%9D%E8%B5%96%E5%8F%8D%E8%BD%AC%E5%8E%9F%E5%88%99
```
This book is a little dated, the languages that it talks about are the ones in use more than a decade ago. But, perhaps there are some aspects of the SOLID principles that may give us a clue about how to talk about a well designed Go programs.

So this is what I want to spend some time discussing with you this morning.
这本书有点点过时，使用的语言也是十多年前的。但是，或许SOLID原则的某些方面可以给我们一个有关如何谈论一个精心设计的Go语言程序的线索。
Now Go obviously doesn't have classes—instead we have the far more powerful notion of composition—but if you can look past the use of the word class, I think there is some value here.

Why is it important that a piece of code should have only one reason for change? Well, as distressing as the idea that your own code may change, it is far more distressing to discover that code your code depends on is changing under your feet. And when your code does have to change, it should do so in response to a direct stimuli, it shouldn't be a victim of collateral damage.

So code that has a single responsibility therefore has the fewest reasons to change.
现在Go语言显然没有classses - 相反，我们有更为强大的组合的概念 - 但是如果你可以看到过去class的使用，我认为这里有其价值。
为什么一段代码应该只有一个原因改变如此重要？当然，和你自己的代码要修改比较起来，发现自己代码所依赖的代码要修改会更令人头疼。而且，当你的代码不得不要修改的时候，它应该对直接的刺激有反应，而不应该是一个间接伤害的受害者。

所以，代码有单一功能原则从而有最少的原因来改变。

#### Coupling & Cohesion
Two words that describe how easy or difficult it is to change a piece of software are coupling and cohesion.

Coupling is simply a word that describes two things changing together–a movement in one induces a movement in another.

A related, but separate, notion is the idea of cohesion, a force of mutual attraction.

In the context of software, cohesion is the property of describing pieces of code are naturally attracted to one another.

To describe the units of coupling and cohesion in a Go program, we might talk about functions and methods, as is very common when discussing SRP but I believe it starts with Go's package model.

这两个词语描绘了修改一段软件代码是何等的简单或困难。
Coupling - 耦合是两个东西一起改变 - 一个移动会引发另一个移动。
Cohesion - 内聚是相关联但又隔离，一种相互吸引的力量。
在软件方面，内聚是形容代码段自然吸引到另一个的属性。
要描述Go语言的耦合与内聚，我们可以要谈论一下functions和methods，当讨论单一功能原则时它们很常见，但是我相信它始于Go语言的package模型。

#### Pakcage命名
在Go语言中，所有的代码都在某个package中。好的package设计始于他的命名。package名字不仅描述了它的目的而且还是一个命名空间的前缀。Go语言标准库里有一些好的例子：
```go
net/http - 提供了http客户端和服务
os/exec - 执行外部的命令
encoding/json - 实现了JSON的编码与解码
```
在你自己的项目中使用其他pakcage时要用import声明，它会在两个package之间建立一个源码级的耦合。

#### 糟糕的pakcage命名
关注于命名并不是在卖弄。糟糕的命名会失去罗列其目的的机会。
比如说server、private、common、utils 这些糟糕的命名都很常见。这些package就像是一个混杂的场所，因为他们好多都是没有原因地经常改变。

#### Go语言的UNIX哲学
以我的观点，涉及到解耦设计必须要提及Doug McIlroy的Unix哲学：小巧而锋利的工具的结合解决更大的任务或者通常原创作者并没有预想到的任务。
我认为Go语言的Package体现了UNIX哲学精神。实际上每个package自身就是一个具有单一原则的变化单元的小型Go语言项目。

### Open / Closed Principle - 开闭原则
```go
package main

type A struct {
        year int
}

func (a A) Greet() { fmt.Println("Hello GolangUK", a.year) }

type B struct {
        A
}

func (b B) Greet() { fmt.Println("Welcome to GolangUK", b.year) }

func main() {
        var a A
        a.year = 2016
        var b B
        b.year = 2016
        a.Greet() // Hello GolangUK 2016
        b.Greet() // Welcome to GolangUK 2016
}
```
typeA有一个year字段以及Greet方法。
typeB嵌入了A做为字段，从而，使B提供的Greet方法遮蔽了A的，调用时可以看到B的方法覆盖了A。
但是嵌入不仅仅是对于方法，它还能提供嵌入 type 的字段访问。如你所见，由于A和B都在同一个package内，B可以访问A的私有year字段就像B已经声明过。

因此 嵌入是一个强大的工具，它允许Go语言type对扩展是开放的。
```go
package main

type Cat struct {
        Name string
}

func (c Cat) Legs() int { return 4 }

func (c Cat) PrintLegs() {
        fmt.Printf("I have %d legs\n", c.Legs())
}

type OctoCat struct {
        Cat
}

func (o OctoCat) Legs() int { return 5 }

func main() {
        var octo OctoCat
        fmt.Println(octo.Legs()) // 5
        octo.PrintLegs()         // I have 4 legs
}
```
在上边这个例子中，typeCat有Legs方法来计算它有几条腿。我们嵌入Cat到一个新的typeOctoCat中，并声明Octocats有五条腿。然而，尽管OctoCat定义了自己有五条腿，但是PrintLegs方法被调用时会返回4。

这是因为PrintLegs在typeCat中定义。它会将Cat做为它的接收者，因此它会使用Cat的Legs方法。Cat并不了解已嵌入的type，因此它的嵌入方法不能被修改。
由此，我们可以说Go语言的types对扩展开放，但是对修改是关闭的。
事实上，Go语言接收者的方法仅仅是带有预先声明形式的参数的function的语法糖而已：
```
func (c Cat) PrintLegs() {
        fmt.Printf("I have %d legs\n", c.Legs())
}

func PrintLegs(c Cat) {
        fmt.Printf("I have %d legs\n", c.Legs())
}
```
第一个function的接收者就是你传进去的参数，而且由于Go语言不支持重载，所以说OctoCats并不能替换普通的Cats，这就引出了接下来一个原则：
#### Liskov Substitution Principle - 里氏替换原则
该原则由Barbara Liskov提出，大致上,它规定了两种类型如果调用者不能区分出他们行为的不同，那么他们是可替代的。
基于class的编程语言，里氏替换原则通常被解释为一个抽象基类的各种具体子类的规范.但是Go语言没有class或者inheritance（继承），因此就不能以抽象类的层次结构实现替换。
#### Interfaces - 接口
相反，Go语言的interface才有权替换。在Go语言中，type不需要声明他们具体要实现的某个interface，相反的，任何想要实现interface的type仅需提供与interface声明所匹配的方法。
就Go语言而言，隐式的interface要比显式的更令人满意，这也深刻地影响着他们使用的方式。

精心设计的interface更可能是小巧的，流行的做法是一个interface只包含一个方法。逻辑上来讲小巧的interface使实现变得简单，反之就很难做到。这就导致了由常见行为连接的简单实现而组成的package。
#### io.Reader
```go
type Reader interface {
        // Read reads up to len(buf) bytes into buf.
        Read(buf []byte) (n int, err error)
}
```
我最喜爱的Go语言interface - io.Reader
interfaceio.Reader非常简单，Read读取数据到提供的buffer，并返回调用者读取数据的bytes的数量以及读取期间的任何错误。它看起来简单但是很强大。
因为io.Reader可以处理任何能转换为bytes流的数据，我们可以在任何事情上构建readers：string常量、byte数组、标准输入、网络数据流、gzip后的tar文件以及通过ssh远程执行的命令的标准输出。
所有这些实现对于另外一个都是可替换的，因为他们都履行了相同的简单合同。
因此，里氏替换原则在Go语言的应用，可以用 Jim Weirich 的格言来总结：
```go
Require no more, promise no less.
–Jim Weirich
```
接下来就到了"SOLID"第四个原则。
#### Interface Segregation Principle - 接口隔离原则
```go
Clients should not be forced to depend on methods they do not use.
–Robert C. Martin
```
在Go语言中，接口隔离原则的应用是指一个方法来完成其工作的孤立行为的过程。举个“栗子”，编写方法来保存一个文档结构到磁盘的任务。
```go
// Save writes the contents of doc to the file f.
func Save(f *os.File, doc *Document) error
```
我可以这样定义这个Save方法，使用*os.File做为保存Document的文件。但是这样做会有一些问题。
Save方法排除了保存数据到网络位置的选项。假如过后要加入网络储存的需求，那么该方法就需要修改也就意味着要影响到所有使用该方法的调用者。

因为Save直接地操作磁盘上的文件，测试起来很不方便。要验证其操作，测试不得不在文件被写入后读取其内容。另外测试必须确保f被写入一个临时的位置而且过后还要删除。
\*os.File还包含了许多跟Save无关的方法，像读取路径以及检查路径是否是软连接。如果Save方法只使用*os.File相关的部分将会非常有用。
我们如何做呢：
```go
// Save writes the contents of doc to the supplied ReadWriterCloser.
func Save(rwc io.ReadWriteCloser, doc *Document) error
```
使用io.ReadWriteCloser来应用接口隔离原则，这样就重新定义了Save方法使用一个interface来描述更为通用的类型。
随着修改，任何实现了io.ReadWriteCloser接口的type都可以代替之前的*os.File。这使得 Save不仅扩展了它的应用范围同时也给Save的调用者说明了type*os.File哪些方法是操作相关的。

做为Save的作者，我没有了在*os.File上调用无关的方法选项了，因为他们都被隐藏于io.ReadWriteCloser接口。我们可以进一步地应用接口隔离原则。

首先，Save方法不太可能会保持单一功能原则，因为它要读取的文件内容应该是另外一段代码的责任。（译注：待更新）因此我们可以缩小接口范围，只传入writing和closing。
```go
// Save writes the contents of doc to the supplied WriteCloser.
func Save(wc io.WriteCloser, doc *Document) error
```
其次，通过向Save提供一种机制来关闭它的数据流，会导致另外一个问题：wc会在什么情况下关闭。Save可能会无条件的调用Close或在成功的情况下调用Close。
如果它想要在写入 document 之后再写入额外的数据时会引起 Save 的调用者一个问题。
```go
type NopCloser struct {
        io.Writer
}
// Close has no effect on the underlying writer.
func (c *NopCloser) Close() error { return nil }
```
一个原始解决方案会是定义一个新的type，在其内嵌入io.Writer以及重写Close方法来阻止Save方法关闭底层数据流。
但是这样可能会违反里氏替换原则，如果NopCloser并没有关闭任何东西。
```go
// Save writes the contents of doc to the supplied Writer.
func Save(w io.Writer, doc *Document) error
```
一个更好的解决办法是重新定义Save只传入io.Writer，剥离它的所有责任除了写入数据到数据流。
通过对Save方法应用接口隔离原则，同时得到了最具体以及最通用的需求函数。我们现在可以使用Save方法来保存数据到任何实现了io.Writer的地方。
```go
A great rule of thumb for Go is accept interfaces, return structs.
–Jack Lindamood
```
#### Dependency Inversion Principle - 依赖反转原则
```go
High-level modules should not depend on low-level modules. Both should depend on abstractions.
Abstractions should not depend on details. Details should depend on abstractions.
–Robert C. Martin
```
对于Go语言来讲，依赖反转意味着什么呢：
如果你应用以上所有的原则，代码已经被分解成离散的有明确责任和目的的package，你的代码应该描述了它的依赖interface以及这些interface应该只描述他们需要的功能行为。换句话说就是他们不会再过多的改变。
因此，我认为Martin所讲的在Go语言的应用是context，即你import graph（译注：后文用“导入图”代替）的结构。
在Go语言中，你的导入图必须是非循环。不遵守此非循环的需求会导致编译错误，但是更为严重的是它代表了一系列的设计错误。

所有条件都相同的情况下精心设计的导入图应该是广泛的以及相对平坦的，而不是又高又窄。如果你有一个package的函数在没有其他package的情况下就无法操作，也许这就表明了代码没有考虑pakcage的边界。
依赖反转原则鼓励你尽可能地像导入图一样在main package或者最高层级的处理程序内对具体细节负责，让低层级代码来处理抽象的接口。
“SOLID” Go语言设计
回顾一下，当应用到Go语言设计中，每个“SOLID”原则都是强有力的声明，但是加在一起他们有一个中心主题。
```
单一功能原则 鼓励你在package中构建functions、types以及方法表现出自然的凝聚力。types属于彼此，functions为单一目的服务。
开闭原则 鼓励你使用嵌入将简单的type组合成更为复杂的。
里氏替换原则 鼓励你在package之间表达依赖关系时用interface，而非具体类型。通过定义小巧的interface，我们可以更有信心地切实满足其合约。
接口隔离原则 鼓励你仅取决于所需行为来定义函数和方法。如果你的函数仅仅需要有一个方法的interface做为参数，那么它很有可能只有一个责任。
依赖反转原则 鼓励你在编译时将package所依赖的东西移除 - 在Go语言中我们可以看到这样做使得运行时用到的某个特定的package的import声明的数量减少。
```


https://www.jianshu.com/p/0aebd9618300
https://www.cnblogs.com/lanxuezaipiao/archive/2013/06/09/3128665.html
http://www.importnew.com/10656.html
https://dave.cheney.net/2016/08/20/solid-go-design
