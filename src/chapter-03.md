# 第三章 设计接口

每个项目, 无论大小, 都有一个`API`. 事实上, 它通常有几个. 其中有些是面向用户的, 比如`HTTP`端点或命令行接口, 有些是面向开发者, 比如库的开放接口. 除此之外, `Rust crate`还有一些内部接口: 每个类型、`trait`和模块边界都有自己的微型`API`, 你的代码与之交互. 随着你的代码库的规模和复杂性的增长, 你会发现非常值得在如何设计内部`API`上投入一些心思和精力, 使用和维护代码尽可能的愉快.

在这一章中, 我们将探讨在`Rust`中编写惯用接口的一些最重要的考虑事项, 无论这些接口的用户是你自己还是使用你的库的其他开发者. 这基本上可以归结为四个原则: 你的接口应该是不出意外的(`unsurprising`), 灵活的(`flexible`), 明显的(`obvious`)和受约束的(`constrained`). 我将依次讨论这些原则, 为编写可靠、可用的接口提供一些指导.

我强烈建议读完本章后, 看看``Rust`API`指南(`https://rust-lang.github.io/api-guidelines/`). 那里有一个很好的清单, 根据清单中详细介绍了解每项建议. 本章中的许多建议也可以通过`cargo clippy`工具检查的, 如果你还没有使用该工具, 建议开始在代码中使用它. 我还鼓励你阅读`RFC 1105`(`https://rust-lang.github.io/rfcs/1105-api-evolution.html`)和`The Cargo Book`中关于`SemVer`兼容性的章节(`https://doc.rust-lang.org/cargo/reference/semver.html`), 这些章节涵盖了`Rust`中哪些是、哪些不是破坏性变更.

## 不意外的(Unsurprising)

最小意外原则, 又称最小意外法则, 在软件工程中经常被提及, 同样也适用于`Rust`接口. 尽可能地, 你的接口应该足够直观, 应该直观到如果用户需要推断, 他们通常会推断正确. 当然, 并不是所有关于你的应用程序的接口能直观呈现, 但任何不令人意外的东西都应该是直观的. 核心思想是紧贴用户可能已经了解的东西, 这样他们就不必以不同于他们习惯的方式重新学习概念. 这样一来, 你就可以把他们的脑力节省下来, 用于解决那些真正与你的接口有关的问题.

有很多方法可以使你的接口变得可预测. 在这里, 我们将探讨如何使用命名、常见`trait`和人体工程学`trait`的技巧来帮助用户.

### 命名惯例

用户在使用接口时, 首先会通过接口的名称来了解它; 他们会立即从所接触到的类型、方法、变量、字段和库的名称中推断出一些东西. 如果你的接口重用了其他(也许是常见的)接口的名称--比如说方法和类型, 用户就会知道可以对你的方法和类型做出某些假设. 名为`iter`的方法可能接收`&self`, 并且可能给你一个迭代器. 名为`into_inner`的方法可能会接收`self`, 并且返回某种包装好的类型. 名为`SomethingError`的类型可能实现了`std::error::Error`, 并出现在各种`Result`中. 通过使用相同目的的通用名称, 使用户更容易推断事物的作用, 并让他们更容易理解你的接口的不同之处.

由此推论, 名字相同的东西实际上应该以同样的方式工作. 否则, 例如, 如果你的`iter`方法使用`self`, 或者`SomethingError`类型没有实现`Error`, 用户很可能会根据他们期望的接口工作方式写出错误的代码. 他们会感到意外和沮丧, 并不得不花时间去研究你的接口与他们的期望有何不同. 如果我们能为用户省去这种麻烦, 我们就应该这么做.

### 类型的共同traits

`Rust`中的用户还会做出一个主要的假设, 即接口中的一切都"都能正常工作". 他们期望能够用`{:?}`打印任何类型, 并将任何东西发送到另一个线程, 他们还期望每个类型都是`Clone`. 在可能的情况下, 我们应该再次避免让用户感到意外, 并积极实现大多数标准`trait`, 即使我们并不立即需要它们.

由于第二章中讨论的一致性规则, 编译器将不允许用户在需要时实现这些`trait`. 用户不允许为外部类型实现一个外部`trait`(如`Clone`), 相反. 他们需要将你的接口类型包裹在他们自己的类型中, 如果不了解该类型的内部结构, 编写一个合理的实现可能会相当困难.

在这些标准`trait`中, 首先是`Debug trait`. 几乎每个类型都应该实现`Debug`, 即使它只打印类型的名称. 使用`#[derive(Debug)]`通常是实现接口中`Debug trait`的最好方法, 但请记住, 所有派生`trait`都会自动为任何泛型参数添加相同的约束. 你可以简单地通过利用`fmt::Formatter`上的各种`debug_` 辅助函数来编写自己的实现.

紧随其后的是`Rust`的自动`trait Send` 和 `Sync`(以及在较小的程度上的`Unpin`). 如果一个类型没有实现这些`trait`之一, 通常需要一个很好的理由的. 不是`Send`的类型不能被放在`Mutex`中, 甚至不能在包含线程池的应用程序中. 未实现`Sync`的类型不能通过 `Arc` 共享或放在静态变量中. 用户已经开始期望类型能在这些情况下工作, 特别是在几乎所有东西都在线程池上运行的异步世界中, 如果你不确保你的类型实现这些`trait`, 他们会感到沮丧. 如果你的类型不能实现这些`trait`, 请确保妥善记录这一事实及其原因.

你应该实现的下一组几乎通用的`trait`是`Clone`和`Default`. 这些`trait`可以很容易地被派生或实现, 对大多数类型来说, 实现这些`trait`是有意义的. 如果你的类型不能实现这些`trait`, 请确保在你的文档中注明, 因为用户通常期望能够根据自己的需要轻松地创建更多(和新)类型的实例. 如果他们不能, 他们会感到惊讶.

在预期`trait`的层次结构中再往下一步就是比较`trait`. `PartialEq`, `PartialOrd`, `Hash`, `Eq`, 和 `Ord`. `PartialEq trait`是特别可取的, 因为用户在某些时候不可避免地会有两个你的类型的实例, 他们希望用`==`或`assert_eq`来比较! 即使你的类型只对同一类型的实例进行等价比较, 也值得实现`PartialEq`, 以使你的用户能够使用 `assert_eq`!

`PartialOrd`和`Hash`更为专业, 适用范围可能没那么广, 但在可能的情况下, 你也要实现它们.  这对于用户可能用作`map`的键的类型, 或者他们可能使用任何`std::collection`集合类型来进行重复的类型, 尤其如此, 因为它们往往需要这些边界. 除了`PartialEq`和`PartialOrd`之外, `Eq`和`Ord`还对实现类型的比较操作有额外的语义要求. 这些在这些`trait`的文档中都有很好的记录, 只有当你确定这些语义确实适用于你的类型时, 你才应该实现它们.

最后, 对于大多数类型来说, 实现`serde crate`的`Serialize`和`Deserialize trait`是有意义的. 这些都可以很容易地派生出来, 而且`serde_derive` 包甚至有机制可以重写一个字段或枚举变体的序列化. 由于`serde` 是一个第三方板块, 你可能不希望添加对它的必要依赖. 因此, 大多数库选择提供一个`serde trait`, 只有在用户选择时才增加对`serde` 的支持.

你可能想知道为什么我没有把可派生`Copy trait`列入本节. 有两件事使`Copy`与其他提到的`trait`不同. 第一件事是, 用户一般不期望类型是`Copy`; 恰恰相反, 他们倾向于期望, 如果他们想要某个东西的两个副本, 他们必须调用 `clone`. 复制改变了移动给定类型的值的语义, 这可能会让用户感到意外. 这与第二个观察相联系: 一个类型很容易不再是`Copy`, 因为`Copy`类型高度受限. 一个开始很简单的类型很容易最终不得不容纳一个字符串, 或者其他一些非拷贝类型. 如果发生这种情况, 你不得不删除`Copy`的实现, 这就是一个向后不兼容的变化. 相比之下, 你很少需要删除`Clone`的实现, 所以这是个不太沉重的承诺.

### 人体工程学`trait`的实现 (Ergonomic`trait`Implementations)

`Rust`不会自动为对实现`trait`的类型的引用实现`trait`. 换个说法, 一般情况下, 你不能用`&Bar`调用`fn foo<T: Trait>(t: T)`, 即使`Bar:Trait`. 这是因为`Trait`可能包含了取值为`&mut self`或`self`的方法, 这显然不能在`&Bar`上调用. 尽管如此, 这种行为可能会让看到 `Trait`只有`&self`方法的用户感到非常惊讶.

因此, 在定义新的`trait`时, 你通常会想为该`trait`提供适当的通用实现, 如`&T where T: Trait`, `&mut T where T: Trait`, 以及`Box<T> where T: Trait`. 你可能只能实现其中的一部分, 这取决于`Trait`的方法的具体接收器. 标准库中的许多`trait`都有类似的实现, 正是因为这样可以减少用户的意外.

迭代器是另一种情况, 在这种情况下, 你通常想在对一个类型的引用上特别添加`trait`实现. 对于任何可以被迭代的类型, 考虑为`&MyType`和`&mut MyType`实现`IntoIterator`. 这样, 就像用户所期望的那样`for`可以在你的类型的借用实例上正常工作, .

### 包装类型

`Rust`没有经典意义上的对象继承.  然而, `Deref trait`和它的表亲`AsRef`都提供了类似于继承的东西. 如果`T: Deref<Target = U>`的话, 这些`trait`允许你拥有一个`T`类型的值, 并通过直接在`T`类型的值上调用`U`类型方法. 这对用户来说, 就像魔法一样, 一般来说非常的棒.

如果你提供的是相对透明的包装类型(如`Arc`), 你很有可能想要实现`Deref`, 这样用户就可以通过使用`.`操作符来调用内部类型上的方法. 如果访问内部类型不需要任何复杂或潜在的缓慢逻辑, 你也应该考虑实现`AsRef`, 它允许用户轻松地将`&WrapperType`作为`&InnerType`使用. 对于大多数包装类型, 你还应尽可能地实现`From<InnerType>`和`Into<InnerType>`, 这样你的用户就可以轻松地添加或删除包装.

你可能也遇到过 `Borrow trait`, 它感觉与`Deref`和`AsRef`非常相似, 但实际上有点不同. 具体来说, `Borrow`是为一个更狭窄使用情况而定制的: 允许调用者提供同一类型的多个基本相同的变体中的任何一个. 也许, 它本被称为等价(`Equivalent`). 例如, 对于一个`HashSet<String>`, `Borrow`允许调用者提供一个`&str`或者一个`&String`. 虽然`AsRef`也可以实现同样的功能, 但如果没有`Borrow`的额外要求, 即目标类型对`Hash`、`Eq`和`Ord`的实现与实现类型完全相同, 那么这么做就不安全了. `Borrow`还为`T`、`&T`和`&mut T`提供了一个`Borrow<T>`的通用实现, 这使得它在`trait`约束中的使用非常方便, 可以接受一个给定类型的自有值或引用值. 一般来说, `Borrow`适用于你的类型本质上等同于另一个类型, 而`Deref`和 `AsRef`则广泛适用于实现你的类型可以"作为"的任何东西.

> `Deref`和固有方法
> 当`T`上有以`self`的方法时, 围绕点运算符和`Deref`的魔法会让人感到困惑和意外. 例如, 给定一个值 `t: T`, 不清楚`t.frobnicate()`是对`T`还是对底层的`U`进行`frobnicate`的调鵑!
> 因此, 那些允许你透明地调用内部类型的方法的类型应该避免使用固有方法. `Vec`有一个`push`方法, 即使它解除对`slice`的引用, 因为你知道`slice`不会很快得到一个`push`方法. 但是, 如果您的类型取消对用户控制的类型的引用, 那么您添加的任何固有方法也可能存在于该用户控制的类型上, 从而导致问题. 在这些情况下, 倾向于`fn frobnicate(t: t)`形式的静态方法. 这样, `t.frobnicate()`总是调用`U::frobnicate`, 而`t::frobnicate(t)`可以用来`T`本身.

## 灵活的

你写的每一段代码都隐含地或明确地包括一个契约. 契约由一组要求和一组承诺组成. 要求是对如何使用代码的限制, 而承诺是对代码如何被使用的保证. 当设计一个新的接口时, 你要仔细考虑这个契约. 一个好的经验法则是避免强加不必要的限制, 只做出你能遵守的承诺.  增加限制或删除承诺通常需要对语义版本进行重大变更, 而且可能会破坏其他地方的代码. 另一方面, 放宽限制或给出额外的承诺, 通常是向后兼容的.

在`Rust`中, 限制通常以`trait`约束和参数类型的形式出现, 而承诺则以`trait`实现和返回类型的形式出现. 例如, 比较清单3-1中的三个函数签名

```rust
fn frobnicate1(s: String) -> String
fn frobnicate2(s: &str) -> Cow<'_, str>
fn frobnicate3(s: impl AsRef<str>) -> impl AsRef<str>

// 清单 3-1: 具有不同契约的类似函数签名
```

这三个函数签名都接收一个字符串并返回一个字符串, 但它们的契约却截然不同.

第一个函数要求调用者以`String`类型的形式拥有字符串, 它承诺将返回一个拥有(所有权)的 `String`. 由于契约要求调用者分配字符串, 并要求我们返回一个拥有(所有权)的字符串, 我们以后不能以向后兼容的方式使这个函数免分配.

第二个函数放宽了契约: 调用者可以提供任何字符串的引用, 所以用户不再需要分配或放弃字符串的所有权. 它还承诺返回一个`std::borrow::Cow`, 这意味着它可以返回一个字符串引用或者一个所有权的字符串, 这取决于它是否需要拥有该字符串. 这里的承诺是, 该函数将始终返回一个`Cow`, 这意味着我们不能在以后改变为使用其他优化的字符串表示. 调用者也必须特别提供一个`&str`, 因此, 如果他们自己的一个预先存在的`String`, 他们必须将其解除引用为一个`&str`来调用我们的函数.

第三个函数取消了这些限制. 它只要求用户传入可以生成字符串引用的类型, 并且只承诺返回值可以生成字符串引用.

这些函数签名中没有哪个一定比其他的更好. 如果函数中需要一个字符串的所有权, 你可以使用第一个参数类型来避免额外的字符串拷贝. 如果你想让调用者利用已分配并返回字符串的情况, 第二个返回类型为`Cow`的函数可能是一个好选择. 相反, 我想让你从中得到的启示是, 你应该仔细考虑你的接口所绑定的契约, 因为事后改变它可能是破坏性变更.

在本节的其余部分, 我将举例说明经常出现的接口设计决策, 以及它们对接口契约的影响.

### 泛型参数

接口必须对用户提出的一个明显的要求是, 他们必须向你的代码提供哪些类型. 如果你的函数明确地接受一个`Foo`, 用户必须拥有并给你一个`Foo`. 这是无法绕过的. 在大多数情况下, 使用泛型而不是具体类型是值得的, 这样可以让调用者传递任何符合你的函数实际需要的类型, 而不是只传递一种特定的类型. 将清单3-1中的`&str`改为`AsRef<str>`是这种放松的一个例子. 以这种方式放宽要求的一个方法是, 从参数的完全泛型化, 不加任何约束, 然后根据编译器的错误来发现你需要添加哪些约束.

然而, 如果将这种方法发挥到极致, 就会使每个函数的每个参数都成为自己的泛型, 这将是既难读又难理解的. 对于何时应该或不应该将某个参数泛型化, 并没有硬性规定, 因此请根据自己的最佳判断来决定. 一个好的经验法则是, 如果你能想到用户可能经常合理地使用其他类型, 而不是一开始使用的具体类型, 那可以就把参数设成泛型.

你可能还记得, 在第2章, 泛型代码通过单态化, 对曾经使用过的每一种类型的组合都会拷贝一份副本. 考虑到这一点, 使大量参数泛化的想法可能会让你担心你的二进制文件过于庞大. 在第2章中, 我们也讨论了如何使用动态分发来缓解这种情况, 其性能代价(通常)可以忽略不计, 这在这里也适用. 对于那些你无论如何都要通过引用来获取的参数(记得`dyn Trait`不是`Sized`, 你需要一个宽指针来使用它们), 你可以很容易地用一个使用动态派发的参数来替换你的泛型参数. 例如, 你可以用`&dyn AsRef<str>`来代替`impl AsRef<str>`.

不过, 在你去做这件事之前, 有几件事情你应该考虑一下. 首先, 你是代表用户做出这个选择的, 而用户无法选择不使用动态分发. 如果你知道你要应用动态分发的代码永远不会对性能敏感, 这可能是好的. 但如果有用户想在他们的高性能应用中使用你的库, 那么在热循环中调用的函数中的动态分发可能会成为一个问题. 其次, 在写这篇文章的时候, 只有当你有一个简单的`trait`约束时, 使用动态分发才能发挥作用, 比如`T:AsRef<str>`或`impl AsRef<str>`. 对于更复杂的约束, `Rust`不知道如何构造动态分发`vtable`, 所以你不能采取例如`&dyn Hash + Eq`. 最后, 请记住, 对于泛型, 调用者总是可以通过传入一个`trait`对象来选择动态分发. 反之则不然: 如果你带了一个`trait`对象, 那就是调用者必须提供该对象.

我们可能一开始使用具体类型的接口, 然后随着时间的推移再将它们变成泛型. 这种做法值得尝试, 这可能行的通, 但请你记住, 这种变化并不一定向后兼容, 了解其原因, 想象一下将`fn foo(v: &Vec<usize>)`改为`fn foo(v: impl AsRef<[usize]>)`. 虽然每个`&Vec<usize>`都实现了`AsRef<[usize]>`, 但类型推断仍然会给用户带来问题. 考虑一下如果调用者用`foo(&iter.collect())`来调用`foo`会发生什么. 在最初的版本中, 编译器可以确定它应该收集到一个`Vec`, 但现在只知道它需要收集到某个实现`AsRef<[usize]>`的类型. 而且可能有多个这样的类型, 所以有了这个改变, 调用者的代码就不会再能编译了!

### 对象安全

当你定义一个新的`trait`时, 该`trait`是否是对象安全的(见第2章"编译和分发"的结尾)是`trait`契约的一个不成文的部分. 如果`trait`是对象安全的, 用户可以使用`dyn Trait`将实现你的`trait`的不同类型视为单一的通用类型. 如果不是, 编译器将不允许该`trait`的`dyn Trait`. 你应该倾向于你的`trait`是对象安全的, 即使这对使用它们的人机工程学来说有一点代价(比如使用`impl AsRef<str>`而不是`&str`), 因为对象安全可以使你的`trait`有新的使用方法. 如果你的`trait`必须有一个泛型方法, 考虑它的泛型参数是否可以`trait`本身, 或者它的泛型参数是否也可以使用动态分发来保持`trait`的对象安全. 另外, 你可以添加一个与该方法约束的`where Self: Sized trait`, 这样就可以只用该`trait`的具体实例来调用该方法(而不是通过`dyn Trait`). 你可以在`Iterator`和`Read trait`中看到这种模式的例子, 它们是对象安全的, 但在具体实例上提供了一些额外的方便方法.

你应该愿意做出多少牺牲来保护对象的安全, 这个问题没有唯一的答案. 我的建议是, 你要考虑你的`trait`将如何被使用, 以及用户想把它作为一个`trait`对象使用是否有意义. 如果你认为用户可能希望使用你的`trait`的许多不同的实例放在一起使用, 你应该更努力地提供对象安全. 例如, 动态分发对于`FromIterator``trait`来说是没有用的, 因为它的一个方法不接受`self`, 所以你首先就不能构造一个`trait`对象. 同样, `std::io::Seek` 作为一个`trait`对象本身是相当无用的, 因为你能用这样一个`trait`对象做的唯一事情就是探索, 而无法读写.

> `Drop trait`对象
> 你可能认为`Drop trait`作为一个`trait`对象也是无用的, 因为作为一个`trait`对象, 你能用`Drop`做的就是析构它. 但事实证明, 有一些库特别希望能够丢弃任意类型. 例如, 一个提供延迟丢弃值的库, 用于并发垃圾收集或只是延迟清理, 只关心值是否可以被丢弃, 而不关心其他. 有趣的是, `Drop`的故事并没有结束; 因为`Rust`也需要能够丢弃`trait`对象, 每个 `vtable` 都包含`drop`方法. 实际上, 每个`dyn Trait`也是一个`dyn Drop`.

请记住, 对象安全是你的公共接口的一部分, 如果你以一种向后兼容的方式修改了一个`trait`, 比如增加了一个带有默认实现的方法, 但这使得该`trait`不再是对象安全的, 你需要提升你的主要语义版本号.

### 借用 vs 所有权

对于在`Rust`中定义的几乎每一个函数、`trait`和类型, 你必须决定它是否应该拥有数据的所有权, 或者只是持有对其数据的引用. 无论你做出什么样的决定, 都会对界面的人体工程学和性能产生深远影响, 幸运的是, 这些决定往往是自己做出的.

如果你写的代码需要数据的所有权, 比如调用带有`self` 的方法或将数据转移到另一个线程, 它必须存储所有权数据. 当你的代码必须拥有数据时, 一般也应该让调用者提供拥有的数据, 而不是通过引用取值并克隆它们. 这使得, 调用者可以控制分配, 并且可以预先了解使用相关接口的成本.

另一方面, 如果你的代码不需要拥有这些数据, 它应该使用引用进行操作. 这个规则的一个常见例外是像`i32`、`bool`或`f64`这样的小类型, 它们直接存储和复制与通过引用存储一样便宜. 不过, 不要以为这条规则适用所有的`Copy`类型都是正确的; `[u8; 8192]`是`Copy`, 但如果到处存储和复制它的成本会很昂贵.

当然, 在现实世界中, 事情往往没有那么一目了然. 有时, 你事先并不知道你的代码是否需要拥有数据. 例如, `String::from_utf8_lossy`需要拥有传递给它的字节序列的所有权中包含无效的`UTF-8`序列时, 才需要获得该序列的所有权. 在这种情况下, `Cow`类型是你的朋友: 如果数据允许, 它可以让你对引用进行操作, 如果需要, 它可以让你产生一个拥有所有权的值.

其他时候, 引用的生命周期会使接口复杂化, 以至于使用起来很麻烦. 如果你的用户在使用你的接口后代码编译时很费劲, 那就说明你可能要(甚至不必要地)对某些数据块拥有所有权. 如果你这样做, 在你决定对可能是一大块字节的数据进行堆分配之前, 先从那些克隆成本低或者对性能不敏感的数据开始.

### 易出错的和阻塞的析构函数

以I/O为中心的类型在析构时往往需要进行清理. 这可能包括刷新写入磁盘的数据, 关闭文件, 或优雅地终止与远程主机的连接. 执行这种清理的自然地方是类型的`Drop`实现. 不幸的是, 一旦一个值被丢弃, 除了`panic`之外, 我们没有办法向用户传达错误了. 异步代码中也会出现类似的问题, 我们希望在有工作未完成时就结束工作. 当`drop`被调用时, 执行器可能已经关闭了, 我们没有办法做更多的工作. 我们可以尝试启动另一个执行器, 但这也会带来一系列的问题, 比如异步代码中的阻塞, 我们将在第8章看到.

这些问题没有完美的解决方案, 无论我们做什么, 一些应用程序将不可避免地回落到我们的`Drop`实现. 出于这个原因, 我们需要通过`Drop`提供尽力而为的清理. 如果清理出错, 至少我们尝试过--吞下错误并继续前进. 如果一个执行器仍然可用, 我们可能会生成一个`future`来进行清理, 但如果它永远不会运行, 我们也已经尽力了.

不过, 我们应该为那些希望不留下线程的用户提供更好的选择. 我们可以通过提供一个显式的析构器来做到这一点. 这通常以一个方法的形式出现, 该方法拥有`self`的所有权, 并暴露销毁过程中固有的任何错误(使用`-> Result<_, _>`)或异步(使用`async fn`). 细心的用户可以使用该方法来优雅地销毁任何相关的资源.

> NOTE: 一定要在文档中显示突出析构函数!

像往常一样, 这需要权衡利弊. 一旦添加了显式析构函数, 就会遇到两个问题. 首先, 由于你的类型实现了`Drop`, 你不能再在析构函数中移出该类型的任何字段. 这是因为在你的显式析构器运行后`Drop::drop`仍然会被调用, 而且它需要`&mut self`, 这要求`self`的任何部分都没有被移动. 其次, `drop` 接收的是`&mut self`, 而不是`self`, 所以你的`Drop` 实现不能简单地调用你的显式析构函数并忽略其结果(因为它并不拥有`self`). 有几个方法可以解决这些问题, 但都不完美.

第一个方法是是顶层类型成为一个包裹在`Option`里的新类型, 而这个新类型又持有一些持有该类型所有字段的内部类型. 然后你可以在两个析构函数中使用`Option::take`, 并且只在内部类型还没有被占用时才调用内部类型的显式析构函数. 因为内层类型没有实现`Drop`, 所以你可以拥有那里的所有字段的所有权. 这种方法的缺点是, 你想在顶层类型上提供的所有方法现在必须包括通过`Option`(你知道它总是`Some`, 因为`Drop`还没有被调用)到内部类型上的字段的代码.

第二个解决方法是使你的每个字段都能被取走. 你可以通过用`None`替换`Option`来"取走"它(这就是`Option::take`的作用), 但你也可以对许多其他类型的字段这样做.  例如, 你可以通过简单地用它们廉价的构造默认值替换`Vec`或`HashMap`来取走它们--`std::mem::take`是你的朋友. 如果你的类型有合理的"空"值, 这种方法就很好用, 但如果你必须用`Option`包裹几乎所有的字段, 然后用一个匹配的`unwrap`来修改这些字段的每一次访问, 就会变得很乏味.

第三种选择是在`ManuallyDrop`类型中保存数据, 它可以解引用到内部类型, 所以无需解包. 你也可以在`drop`中使用`ManuallyDrop::take`来在销毁时取得所有权. 这种方法的主要缺点是`ManuallyDrop::take`是不安全的. 没有任何安全机制来确保你在调用`take`后不会尝试使用`ManuallyDrop`中的值, 或者不会多次调用`take`. 如果你这样做了, 你的程序就会默默地表现出未定义的行为, 并会发生不好的事情.

最终, 你应该选择这些方法中最适合你应用的方法. 我倾向于选择第二种方案, 只有当你发现自己处于`Option`中时才会切换到其他方法. 如果代码足够简单, 你可以很容易地检查你的代码的安全性, 而且你对自己的能力有信心, 那么`ManuallyDrop`解决方案是非常好的.

## 易理解的

虽然有些用户可能熟悉支撑接口的实现的某些方面, 但他们不可能理解所有的规则和限制. 他们不会知道在调用`bar`之后再调用`foo`是绝对不行的, 也不会知道只有在月亮呈47度角且过去18秒内没有人打喷嚏的情况下, 调用不安全方法`baz`才是安全的. 只有当接口清楚地表明发生了一些奇怪的事情, 他们才会去查阅文档或仔细阅读类型签名. 因此, 对你来说, 让用户尽可能容易地理解你的接口, 并让他们尽可能难以错误地使用你的接口是至关重要的. 在这方面, 你所掌握的两个主要技术是你的文档和类型系统, 所以让我们依次看一下这两个技术.

> NOTE: 你也可以利用命名来向用户暗示, 一个接口的内容不只是看起来那么简单. 如果用户看到一个名为`dangerous`的方法, 他们很有可能会阅读其文档.

### 文档

让接口透明化的第一步是写好文档. 我可以写一整本书来介绍如何编写文档, 但在这里我们还是专注于针对`Rust`的建议.

首先, 清楚地记录代码可能执行意外操作的情况, 或者它依赖于用户执行超出类型签名规定的事情. `panic`是这两种情况的一个很好的例子: 如果你的代码可能会恐慌, 请记录这一事实, 以及它可能恐慌的情况. 同样地, 如果你的代码可能会返回一个错误, 请记录它在哪些情况下会返回错误. 对于不安全的函数, 记录调用者必须保证什么才能使调用安全.

其次, 在`crate`和模块层面上为你的代码提供端到端的使用范例. 这些示例比特定类型或方法的示例更重要, 因为它们让用户感觉到所有东西是如何结合在一起的. 有了对接口结构的高层次理解后, 开发者可能很快就会意识到特定的方法和类型的作用, 以及它们应该在何处使用. 端到端示例也给用户一个自定义使用的起点, 他们可以, 而且经常会复制粘贴这个示例, 然后根据他们的需要进行修改. 这种"边做边学"的方式往往比让他们尝试从组件中拼凑出一些东西更有效.

> NOTE: 特定于方法的示例表明, 是的, `len`方法确实返回了长度, 不太可能让用户对你的代码有什么新的了解.

第三, 组织文档. 把所有的类型、`trait`和函数放在一个顶层的模块中, 会让用户感到不知从何下手. 利用模块的优势, 将语义相关的项目组合在一起. 然后, 使用文档内的链接来相互连接项目. 如果类型A的文档谈到了B `trait`, 那么就应该在这里链接到该`trait`. 如果能让用户更容易地探索你的接口, 他们就不太会错过重要的联系或依赖关系. 也可以考虑用`#[doc(hidden)]` 来标记你的接口中那些不打算公开但由于历史遗留原因需要的部分, 这样就不会使文档变得杂乱无章.

最后, 尽可能丰富你的文档. 链接到解释概念、数据结构、算法或接口的其他方面的外部资源, 这些资源可能在其他地方有很好的解释. `RFCs`、博客文章和白皮书都是很好的选择, 如果有相关的话. 使用`#[doc(cfg(..))]` 来强调只在特定配置下才可用的项目, 这样用户就能很快意识到文档中列出的某些方法是不可用的. 使用`#[doc(alias = "...")]` 以其它名称显示类型和方法, 以便用户搜索. 在顶层文档中, 指出用户常用的模块、特性、类型、`trait`和方法.

### 类型系统指导

类型系统是确保你的接口是明显的、自动文档化和防误用的绝佳工具. 你可以利用几种技术使你的接口很难被误用, 从而使它们更有可能被正确使用.

第一种是语义类型, 即添加类型来表示值的含义, 而不仅仅是其原始类型. 最典型的例子是布尔运算: 如果函数需要三个布尔参数, 那么很有可能一些用户会弄乱这些值的顺序, 并在出了大问题之后才意识到这一点. 另一方面, 如果提供三个不同的双变量枚举类型的参数, 编译器没报错那么用户就不会获得错误的顺序: 如果他们试图将 `DryRun::Yes`传递给`overwrite`参数, 这将根本不起作用, 将`overwrite::No`作为`dry_run`参数也不行. 除了布尔类型, 我还可以应用语义类型. 例如, 围绕数字类型的`newtype`可以为所包含的值提供一个单位, 或者它可以将原始指针参数限制在仅由另一个方法返回的参数上.

一个密切相关的技术是使用零大小的类型来表示类型实例的某一个特定的事实为真. 例如, 考虑一个叫做`Rocket`的类型, 它代表真正的火箭的状态. 无论火箭处于什么状态, 火箭上的一些操作(方法)都应该是可用的, 但有些操作只有在特殊情况下才有意义. 例如, 如果火箭已经被发射了, 就不可能再发射. 同样的, 如果火箭还没有发射, 也不可能分离燃料箱. 我们可以将这些建模为枚举变体, 但是这样一来, 所有的方法在每个阶段都是可用的, 我们就需要引入可能的恐慌了.

相反, 如清单3-2所示, 我们可以在`Rocket`上引入一个通用参数`Stage`, 并用它来限制什么情况下可以使用什么方法.

```rust
struct Grounded;   // (1)
struct Launched;
// and so on
struct Rocket<Stage = Grounded> {
    stage: std::marker::PhantomData<Stage>, // (2)
}
impl Default for Rocket<Grounded> {}    // (3)
impl Rocket<Grounded> {
    pub fn launch(self) -> Rocket<Launched> { }
}
impl Rocket<Launched> { // (4)
    pub fn accelerate(&mut self) { }
    pub fn decelerate(&mut self) { }
}
impl<Stage> Rocket<Stage> { // (5)
    pub fn color(&self) -> Color { }
    pub fn weight(&self) -> Kilograms { }
}

// 第 3-2 项: 使用标记类型来限制实现的方法
```

我们引入单元类型来表示火箭的每个阶段(1). 我们实际上不需要存储阶段--只需要存储它提供的元信息--所以我们把它存储在`PhantomData`(2) 后面, 以保证它在编译时将其消除. 然后, 我们只在`Rocket`持有特定类型的参数时为其编写实现块. 你只能在地面上建造一个火箭(目前), 而且你只能从地面上发射它(3). 只有当火箭发射后, 你才能控制它的速度(4). 无论火箭处于什么状态, 你都可以对它做一些事情, 这些事情我们放在一个通用的实现块中(5).  你会注意到, 以这种方式设计的接口, 用户根本不可能在错误的时间调用方法, 我们已经将使用规则编码在类型本身中, 并使非法状态无法表示.

这个概念也延伸到许多其他领域; 如果函数忽略了指针参数, 除非给定的布尔参数为真, 那么最好把这两个参数结合起来. 有了一个枚举类型, 其中一个变体代表`false`(没有指针), 一个变体代表 `true`, 持有一个指针, 无论是调用者还是实现者都不会误解这两者之间的关系. 这是一个强大的想法, 我强烈建议你加以利用.

另一个让接口显而易见的小工具是`#[must_use]`注解. 把它添加到任何类型、`trait`或函数中, 如果用户的代码接收到该类型或`trait`的元素, 或调用该函数, 而没有显示地处理它, 编译器就会发出警告. 你可能已经在`Result`的上下文中看到了这一点: 如果一个函数返回`Result`, 而你没有把它的返回值赋值给某个地方, 你会得到一个编译器警告. 请注意不要过度使用这个注解--只有在用户不使用返回值时很可能会犯错时才会添加它.

## 受约束的

随着时间的推移, 一些用户会依赖你的接口的每一个属性, 无论是错误还是功能. 这对于公开的库来说尤其如此, 因为你无法控制你的用户. 因此, 在进行用户可见的改变之前, 你应该仔细考虑. 无论你是添加新的类型、字段、方法或`trait`实现, 还是更改现有的实现, 你都要确保这个改变不会破坏现有用户的代码, 而且你打算将这个变更保留一段时间. 频繁的向后不兼容的变更(语义版本中的主要版本增加)肯定会引起用户的不满.

许多向后不兼容的变更是显而易见的, 比如重命名一个公共类型或删除一个公共方法, 但有些更改更为微妙, 与`Rust`的工作方式有很大关系. 在这里, 我们将介绍一些比较棘手的微妙变化, 以及如何为它们规划变化. 你会发现, 你需要在其中一些变化与你希望你的接口灵活性之间取得平衡--有时候, 有些东西必须要让步.

### 类型修改

删除或重命名公共类型几乎肯定会破坏某些用户的代码. 为了解决这个问题, 你要尽可能地利用`Rust`的可见性修改器, 比如`pub(crate)`和`pub(in path)`. 你拥有的公有类型越少, 以后更改的自由度就越大, 而不会破坏现有的代码.

不过, 用户代码可以在更多的方面依赖你的类型, 而不仅仅是名称. 请看清单3-3中的公共类型和该代码的给定代码.

```rust
// 在你的接口
pub struct Unit;
// 在用户的代码
let u = lib::Unit;

// 清单 3-3: 一个看起来无辜的公共类型
```

现在想想如果你给`Unit`添加一个私有字段会发生什么. 即使添加的字段是私有的, 但这个更改仍然会破坏用户的代码, 因为他们所依赖的构造函数已经消失了. 类似地, 请看清单3-4中的代码和用法.

```rust
// 你的接口
pub struct Unit { pub field: bool };
// 用户代码
fn is_true(u: lib::Unit) -> bool {
    matches!(u, Unit { field: true })
}

// 清单 3-4: 访问单个公共字段的用户代码
```

在这里, 给`Unit`添加一个私有字段也会破坏用户代码, 这次是因为`Rust`的穷举模式匹配检查逻辑能够看到用户看不到的接口部分. 编译器发现有更多的字段, 尽管用户代码无法访问它们, 并以不完整为由拒绝用户的模式匹配. 如果我们将元组结构构变成带有命名字段的普通结构, 也会出现类似的问题: 即使字段本身完全相同, 但旧的模式对新的类型定义也不再有效.

`Rust`提供了`#[non_exhaustive]`属性来帮助缓解这些问题. 你可以把它添加到任何类型的定义中, 编译器将不允许在该类型上使用隐式构造函数(如`lib::Unit { field1: true }`)和非穷举模式匹配(即没有尾巴的模式, `..`). 如果你怀疑自己将来可能会修改某个特定的类型, 这是一个很好的属性. 但它确实限制了用户的代码, 例如剥夺了用户依赖穷举模式匹配的能力, 所以如果你认为给定的类型可能会保持稳定, 请避免添加该属性.

### `trait`实现

正如第2章中所述, `Rust`的一致性规则不允许对给定类型的多个`trait`的实现. 由于我们不知道下游代码可能添加了哪些实现, 所以添加一个现有`trait`的通用实现通常是一种破坏性的改变. 同样的道理也适用于为一个现有类型实现一个外来`trait`, 或者为一个外来类型实现一个现有`trait`--在这两种情况下, 外来`trait`或类型的所有者可能同时添加一个冲突的实现, 所以这一定是一个破坏性的变更.

删除`trait`的实现是一种破坏性的变更, 但为新的类型实现`trait`从来都不是问题, 因为任何`crate`都不能有与该类型冲突的实现.

也许与直觉相反, 在为现有的类型实现任何`trait`也要小心谨慎. 请看清单 3-5 中的代码, 就会明白其中的原因.

```rust
// crate1 1.0
pub struct Unit;
pub trait Foo1 { fn foo(&self) }
// note that Foo1 is not implemented for Unit

// crate2; depends on crate1 1.0
use crate1::{Unit, Foo1};
trait Foo2 { fn foo(&self) }
impl Foo2 for Unit { .. }
fn main() {
    Unit.foo();
}

// 清单 3-5: 为一个现有的类型实现一个`trait`可能会引起问题. 
```

如果你在`crate1`中添加了`impl Foo1 for Unit`, 而没有将其标记为破坏性变更, 那么下游的代码会突然停止编译, 因为现在对`foo`的调用是不明确的. 这甚至可以适用于新的公共`trait`的实现, 如果下游的包使用通配符导入(使用`cate1::*`). 如果你提供了一个`prelude`模块, 并指示用户使用通配符导入, 你将特别需要记住这一点.

对现有`trait`的大多数改变也是破坏性的改变, 例如改变方法签名或添加新方法. 改变方法的签名会破坏该`trait`的所有实现, 可能还会破坏很多使用, 而添加一个新的方法"只是"破坏所有的实现. 不过, 添加一个带有默认实现的新方法并没有问题, 因为现有的实现将继续适用.

我在这里说 "一般 "和 "大多数", 是因为作为接口作者, 我们有一个工具可以让我们绕过其中的一些规则: 密封的`trait`. 一个密封的`trait`是一个只能由其他包使用, 而不能实现的`trait`. 这立即使一些破坏性的变化变得不那么破坏. 例如, 你可以为一个密封的`trait`添加一个新的方法, 因为你知道在当前的包之外没有任何实现需要考虑. 同样地, 你可以为新的外部类型实现一个密封的`trait`, 因为你知道定义该类型的外部包不可能添加一个冲突的实现.

密封`trait`最常用于派生`trait`--为实现特定其他`trait`的类型提供通用实现的`trait`. 只有当外部的包实现你的`trait`没有意义时, 你才应该密封`trait`; 这严重限制了该`trait`的实用性, 因为下游的`crate`将不再能够为他们自己的类型实现该`trait`. 你也可以使用密封的`trait`来限制哪些类型可以被用作类型参数, 比如在清单3-2中的火箭例子中, 将`Stage`类型限制为只有`Grounded`和`Launched`的类型.

清单 3-6 显示了如何封存一个`trait`, 以及如何在定义箱中为它添加实现.

```rust
pub trait CanUseCannotImplement: sealed::Sealed /* (1)  */ { .. }
mod sealed {
    pub trait Sealed {}
    impl<T> Sealed for T where T: TraitBounds {} // (2)
}
impl<T> CanUseCannotImplement for T where T: TraitBounds {}

// 清单 3-6: 如何密封一个`trait`并为其添加实现
```

诀窍是添加一个私有的、空的`trait`, 作为你希望密封(1)的`trait`的一个父`trait`. 由于父`trait`在一个私有模块中, 其他的`crate`无法访问它, 因此也无法实现它. 封闭的`trait`要求底层类型实现`Sealed`, 所以只有我们明确允许的类型(2)才能最终实现该`trait`.

> NOTE: 如果你确实以这种方式密封了`trait`, 请确保你记录了这一事实, 这样用户在试图自己实现`trait`时就不会感到沮丧了.

### 隐性契约

有时, 对代码某一部分所做的更改会以微妙的方式影响接口中其他部分的契约. 发生这种情况的两种主要方式是通过重导出和自动`trait`.

#### 重导出

如果你的接口的任何部分暴露外部类型, 那么对这些外部类型的任何改变也是对你接口的改变. 例如, 考虑一下如果你迁移到一个新的依赖关系的主要版本, 并将该依赖关系中的一个类型作为你的接口中的一个迭代器类型公开, 会发生什么. 依赖于你的接口的用户可能也会直接依赖该依赖关系, 并期望你的接口提供的类型与该依赖关系中的同名类型相同. 但是你更改了依赖项的主要版本, 即使类型的名称是相同, 这也不再是真的相同了. 清单3-7显示了一个这样的例子.

```rust
// your crate: bestiter
pub fn iter<T>() -> itercrate::Empty<T> { .. }
// their crate
struct EmptyIterator { it: itercrate::Empty<()> }
EmptyIterator { it: bestiter::iter() }

// 清单 3-7: 重新导出使外部的包成为接口契约的一部分. 
```

如果你的`crate`从`itercrate 1.0`移到`itercrate 2.0`, 但其他方面没有变化, 那么本列表中的代码将不再被编译. 尽管类型没有改变, 编译器认为(正确地)`itercrate1.0::Empty`和`itercrate2.0::Empty`是不同的类型. 因此, 你不能将后者赋值给前者, 这将破坏您的接口.

为了减少类似的问题, 通常最好使用`newtype`模式来包装外部类型, 然后只公开外部类型中你认为有用的部分. 在很多情况下, 你可以通过使用`impl Trait`来避免`newtype`包装器, 只向调用者提供非常小的契约. 通过较少的承诺, 就可以减少破坏性的改动.

> SEMVER 的诀窍
> `itercrate`的示例可能让你产生误解. 如果`Empty`类型没有改变, 那么为什么编译器不允许任何使用它的代码继续工作, 而不管代码是使用它的1.0还是2.0版本？答案是很..... 复杂. 归根结底: `Rust`编译器并不会因为两个类型字段相同, 就认为它们是相同的. 举个简单的例子, 想象一下`itercrate 2.0`为`Empty`增加了一个`#[derive(Copy)]`. 现在, 这个类型突然有了不同的移动语义, 这取决于你使用的是1.0还是2.0! 而用其中一个类型编写的代码在另一个类型中就无法运行了.
>
> 这个问题往往会出现在大型的、广泛使用的库中, 随着时间的推移, `crate`中的某个地方很有可能发生破坏性的改动. 不幸的是, 语义上的版本控制是在`crate`层面上进行的, 而不是在类型层面上, 因此, 任何地方的破坏性改变都是一种破坏性改变.
>
> 一切并没有结束. 几年前, David Tolnay(`serde`的作者, 还有其他大量的`Rust`贡献者)想出了一个巧妙的技巧来处理这种情况. 他称其为"semver技巧". 这个想法很简单: 如果某个类型的`T`在破坏性变改中保持不变(比如从1.0到2.0), 那么在发布2.0之后, 你可以发布一个新的1.0次要版本, 该版本依赖于2.0, 并且用2.0中的`T`的重导出替换`T`.
>
> 这样做可以确保两个主要版本中都只有一个单一的`T`类型. 这反过来又意味着任何依赖于`1.0`的板块都可以使用`2.0`的 T, 反之亦然. 因为这只发生在你明确选择的类型上, 因此那些实际上会破坏的变更将继续存在.

#### 自动 traits(Auto-Traits)

`Rust`有一些`trait`, 会根据每个类型所包含的内容自动实现. 其中与本讨论最相关的是`Send`和`Sync`, 尽管`Unpin`、`Sized`和`UnwindSafe trait`也有类似的问题. 就其本质而言, 这些`trait`为你接口中的几乎所有类型添加了一个隐藏的承诺. 这些`trait`甚至可以通过其他类型的消除类型传播, 比如`impl Trait`.

这些`trait`的实现(通常)是由编译器自动添加的, 但这也意味着, 如果这些实现不再适用, 也不会自动添加. 所以, 如果你有一个包含私有类型`B`的公共类型`A`, 而你改变了`B`, 使其不再是`Send`, 那么`A`现在也不再是`Send`了. 这就是一个破坏性的变更!

这些变更可能很难被跟踪, 通常直到接口的用户抱怨他们的代码不再工作时才会被发现. 为了在这些情况发生之前捕捉到它们, 在你的测试套件中加入一些简单的测试是个不错的做法, 以检查你的所有类型是否以你期望的方式实现了这些`trait`.  清单3-8给出了这样一个测试的例子.

```rust
fn is_normal<T: Sized + Send + Sync + Unpin>() {}
#[test]
fn normal_types() {
    is_normal::<MyType>();
}

// 清单 3-8: 测试一个类型是否实现了一组特征
```

注意, 该测试并不运行任何代码, 只是测试代码的编译情况. 如果`MyType`不再实现`Sync`, 测试代码将不能编译, 你将知道你刚才的改变破坏了自动`traits`的实现.

> 从文档中隐藏项目
> 通过`#[doc(hidden)]`属性可以让你在文档中隐藏一个公共项目, 而不会让碰巧知道它存在的代码无法访问. 这通常被用来公开宏所需要的方法和类型, 且用户代码不需要的. 这种隐藏如何与你的接口契约互动还存在一些争议. 一般来说, 标记为`#[doc(hidden)]`的项目只在其公共效应范围内才被视为契约的一部分; 例如, 如果用户代码最终可能包含一个隐藏的类型, 那么该类型是`Send`是契约的一部分, 而其名称则不是. 隐藏的固有方法和隐藏在密封`trait`上的方法通常不是你的接口契约的一部分, 尽管你应该确保在这些方法的文档中明确说明这一点. 是的, 隐藏的项目仍然应该被记录下来!

## 总结

在本章中, 我们探讨了设计`Rust`接口的许多方面, 无论是供外部使用, 还是仅仅作为`crate`中不同模块之间的一个抽象边界. 我们介绍了很多具体的陷阱和技巧, 但最终, 高层次的原则应该指导你的思考方向: 你的接口应该是不令人意外的、灵活的、明显的和受约束的. 在下一章中, 我们将深入探讨如何表述和处理`Rust`代码中的错误.
