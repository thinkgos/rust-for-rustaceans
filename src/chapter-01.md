# 第一章 - 基础  

当你深入研究`Rust`高级知识之前, 对基础知识有扎实的理解非常重要. 与其他编程语言一样, 在`Rust`中, 当你开始以更复杂的方式使用该语言时, 各种关键字和概念的准确含义变得非常重要. 在本章中, 我们将讲解许多`Rust`的原语, 并尝试更清晰地定义它们的含义, 工作原理, 以及它们为何如此设计. 具体来说, 我们将了解变量和值区别, 它们在内存中的表示方式, 以及在程序的不同内存区域. 然后, 我们将讨论所有权、借用和生命周期的一些微妙之处, 这些是你在继续学习本书之前必须掌握的内容.

你可以选择从头到尾阅读本章, 也可以将它作为参考资料, 查阅那些你不太确定的概念. 我建议你只有在完全理解本章内容后再继续学习, 因为对这些原语工作原理的误解会很快影响你对更高级主题的理解, 或者导致你错误地使用它们.

## 谈谈内存  

并非所有内存都是一样的. 在大多数编程环境中, 你的程序可以访问栈(`stack`)、堆(`heap`)、寄存器(`register`)、文本段(`text segment`)、内存映射寄存器(`memory-mapped register`)、内存映射文件(`memory-mapped file`), 甚至是非易失性 `RAM`(`Nonvolatile RAM`). 在特定情况下, 你选择使用哪一种会影响到你能其中存储什么, 它能存储多久, 以及您使用什么机制来访问它. 这些内存区域的具体细节因平台而异且也超出了本书的范围, 但有些内存区域对理解`Rust`代码非常重要, 因此值得在这里介绍.

### 内存术语  

在深入探讨内存区域之前, 你首先需要了解值、变量和指针之间的区别. 在`Rust`中, 一个值是由类型和该类型的值域的组合. 一个值可以使用其类型的表示方式转化为字节序列, 但其抽象角度来看, 值更像是程序员所表达的"意义". 例如, `u8` 类型中的数字 `6` 是数学整数 `6` 的一个实例, 它在内存中的表示是字节 `0x06`. 类似地, `"Hello world"`是字符串域值域中的一个值, 其表示方法是`UTF-8`编码的字节序列. 一个值的意义与它的字节存储位置无关.

一个值被存储在一个*place*(位置)中, 这是`Rust`中表示"一个可以容纳值的位置"的术语. 这个位置可以在栈上, 也可以在堆上, 或者其他位置. 最常见的存储值的位置是变量, 它是栈上的一个具名的值槽.

指针是一个包含某段内存地址的值, 因此指针指向一个位置. 指针可以被解引用, 以访问其指向的内存位置中存储的值. 我们可以将同一个指针存储在多个变量中, 从而使这些变量间接地引用同一块内存位置, 从而引用同一个底层值.

请看清单 2-1 中的代码, 它说明了这三个要素.

```rust
let x = 42;
let y = 43;
let var1 = &x;
let mut var2 = &x;
var2 = &y; // (1)

// 清单 2-1: 值、变量和指针
```

这里有四个不同的值. `42`(一个`i32`), `43`(一个`i32`), 变量`x` 的地址(一个指针), 以及 变量`y` 的地址(一个指针). 还有四个变量: `x`、`y`、`var1` 和 `var2`. 后两个变量都持有指针类型的值, 因为引用在本质上就是指针. 虽然 `var1` 和 `var2` 最初存储的是同一个值, 但它们分别存储该值的独立副本; 当我们修改 `var2`(1)中存储的值时, `var1` 中的值不会受到影响. 尤其是`=` 运算符, 它将右侧表达式的值赋给左侧命名的位置.

一个有趣的例子说明变量、值和指针之间的区别有多重要, 比如在下面的语句中:

```rust
let string = "Hello world";
```

即使我们将一个字符串值赋给变量 `string`, 但该变量的实际值是指向字符串值"Hello world"中第一个字符的指针, 而不是整个字符串值本身. 这时你可能会说: "等等, 那么字符串值存储在哪里呢? 指针指向哪里?" 如果你注意到了这一点, 那说明你观察力很敏锐--我们稍后就会解释.

> NOTE: 从技术上讲, 字符串还包括字符串的长度. 我们将在第 3 章讨论胖指针类型时讨论这个问题.  

### 深入了解变量

我之前给出的变量定义比较宽泛, 本身不太可能有用. 当你接触更复杂的代码时, 你将需要一个更准确的思维模型, 来帮助你理解程序实际在做什么. 我们可以使用许多这样的模型. 详细描述它们将耗费几章的篇幅, 也超出了本书的范围, 但大体上可以将它们分为两类: 高层模型(`High-level`)和低层模型(`low-level`). 高层模型在处理生命周期和借用之类的代码时非常有用, 而低层模型则适用理解不安全代码和裸指针. 接下来两个小节将介绍变量变量模型, 这足够满足本书的大部分内容.

#### 高层模型

在高级模型中, 我们不把变量看作是存放字节的地方. 而是把它们看作在程序中用于标识值的名字, 随着值的创建、移动和使用而存在. 当你把一个值赋给变量后, 这个值就由该变量命名. 当以后访问这个变量时, 你可以想象从上一次访问到这次访问这间画一条线, 这条线表示两次访问之间的依赖关系. 如果该变量中的值被移动了, 就不能再从该它画出依赖线了.

在这个模型中, 变量只有在持有合法值的情况下存在; 你不能从一个未初始化或已被移动了值的变量画出依赖线, 因为它实际上并不存在. 使用这种模型, 整个程序由许多依赖线组成, 这此依赖线通常称为流, 每一条流都会追踪某个值的特定实例的生命周期. 流可以分叉和合并, 每个分支都会跟踪该值的一个独立生命周期. 编译器可以检查, 在程序的任何给定点, 所有可以并行存在的流是否兼容. 例如, 不能有两个并行的流同时对一个值进行可变访问. 也不能在没有值(未初始化)的流中借用一个值. 清单2-2显示了这两种情况的示例.

```rust
let mut x;
// 这是非法的, 没有地方可以获取流
// assert_eq!(x, 42);
x = 42;     // (1)
// 这是正确的, 可以从上面分配的值中画出一个流. 
let y = &x; // (2)
// 这就建立了第二个来自 x 的、可变的流. 
x = 43;     // (3)
// 这样就继续从 y 那里获得流, 而 y 又从 x 那里获得流.  
// 但这条流与对 x 的分配相冲突! 
assert_eq!(*y, 42); // (4)

// 清单 2-2: 借用检查器会发现的非法流
```

首先, 在`x`被初始化之前, 我们不能使用它, 因此我们无法画出流. 只有当我们给`x`赋值后, 我们才能从它那里画出流. 这段代码有两个流: 一个从`(1)`到`(3)`的独占(`&mut`)流, 一个从`(1)`到`(2)`到`(4)`的共享(`&`)流. 借用检查器检查每个流的每个节点, 并确保没有其他不兼容的流同时存在. 在本例中, 当借用检查器检查`(3)`处的独占流时, 它看到了终止于`(4)`处的共享流. 由于你不能同时对一个值进行独占和共享使用, 借用检查器(正确地)拒绝了这段代码. 请注意, 如果没有`(4)`, 这段代码可以正常编译. 共享流将在`(2)`处终止, 而当检查`(3)`处的独占流时, 不会有冲突的流存在.

如果声明一个新变量与之前的变量同名, 它们仍被视为不同的变量. 这被称为"shadows"(变量遮蔽)-- 后一个变量"shadows"(遮蔽)了前一个同名的变量. 这两个变量共存, 但后续代码无法再使用前一个变量. 这个模型与编译器, 特别是借用检查器, 对你的程序的理解大致吻合, 编译器内部使用这种模式来生成高效的代码.

#### 底层模型

变量是对内存位置的命名, 这些位置可能包含合法的值, 也可能不包含. 你可以把变量看作一个“值槽”. 当你对它赋值时, 这个槽被填满了, 它的原有的值(如果有的话)就会被释放和并替换掉. 当你访问这个变量时, 编译器会检查该槽是否为空, 如果为空, 说明该变量未被初始化或其值已被移动了. 指向变量的指针持有的是该变量背后的内存, 可以被解引用来获取它的值. 例如, 在语句`let x: usize` 中, 变量`x`是栈上某个内存区域的命名, 这块内存足够存储一个 `usize` 类型的值, 尽管它没有一个明确的值(其槽是空的). 如果你给这个变量赋值, 比如`x = 6`, 那么这个内存区域就会保存表示值`6`的位, 即使你多次赋值`x`, 并不会改变`&x`的值, 这个模型与`C`和`C++`等许多低级语言所使用的内存模型相匹配, 当需要显示推理内存行为时很有用.

> 注意: 在本例子中, 我们忽略了CPU寄存器, 并将其视为一种优化. 实际上, 如果一个变量不需要内存地址, 编译器可能会使用一个寄存器来存储这个变量, 而不是一个内存区域.

你可能会发现这两个模型中的某一个与你之前的思维模型更契合, 但我建议你试着理解这两个模型. 它们都是同样有效的, 而且都是简化的, 就像任何有用的思维模型一样. 如果您能从这两个角度来看待一段代码, 您就会发现在理解复杂的代码时会容易得多, 也更容易理解为什么它们能或不能按照您的预期编译和运行.

### 内存区域

现在你已经掌握了我们如何使用内存, 我们需要谈谈内存到底是什么. 内存其实分为许多不同的区域, 也许令人惊讶的是, 并不是所有的区域都存储在你的计算机的`DRAM`中. 使用哪个部份内存, 会显著影响你如何编写代码. 就编写`Rust`代码而言, 三个最重要的区域是栈、堆和静态内存.

#### 栈

栈是一段内存区域, 程序使用它作为函数调用时的临时工作空间. 每次函数被调用时, 都会在栈顶分配一块连续的内存区域, 称为栈帧. 接近栈的底部的是主函数的帧, 当函数调用其他函数时, 新的的栈帧被依次压入栈中. 每个函数的栈帧包含该函数内部的所有变量, 以及传入该函数的所有参数. 当函数返回时, 它对应的栈帧就会被释放.

构成函数局部变量值的字节不会立即被清除, 但访问它们是不安全, 因为它们可能被随后的函数调用所分配的栈桢覆盖. 即使它们没有被覆盖, 它们也可能包含非法的值, 例如函数返回时已被移动走的值.

栈帧, 以及它们最终会消失这一关键事实, 与`Rust`中的生命周期(`lifetime`)概念密切相关. 任何存储在栈帧中的变量, 在该栈帧消失后都无法被访问, 所以对它的任何引用, 其生命周期最多只能和该栈帧的生命周期一样长.

#### 堆

堆是一块与程序当前调用栈无关的内存池. 存储在堆中的值会一直存在, 直到它们被显式地释放. 当您希望某个值的生命周期超过当前函数栈帧时, 这就非常有用. 如果该值是函数的返回值, 调用函数可以在其栈上预留一块空间, 供被调用函数在返回之前将该值写入其中. 但如果你想将该值发送到另一个线程, 而与当前线程可能根本不共享栈帧, 你可以将该值存储在堆上.

堆允许您显式地分配一段连续的内存区域. 分配时, 你会得到一个指向该内存区域起点位置的指针. 该内存段将为你保留, 直到你显式释放它; 这个过程通常被称为"释放", 这个名称来源于`C`标准库中的对应函数. 由于从堆上分配的内存不会在函数返回时释放, 所以您可以在某个地方为一个值分配内存, 然后将指向该值的指针传递给另一个线程, 并让该线程安全地继续操作这个值进行. 或者, 换句话说, 当你在堆上分配内存时, 得到的指针具有不受约束的生命周期——你的程序想让它存活多长时间, 它就能存活多长时间.

在`Rust`中, 与堆交互的主要机制是`Box`类型. 当您写下`Box::new(value)`时, 这个值被放在堆上, 而您得到的(`Box<T>`)是指向堆上该值的指针. 当`Box`最终被释放时, 内存将被释放.

如果你忘记释放堆内存, 它将会一直存在, 最终你的应用程序会耗尽你机器上的所有内存. 这被称为"内存泄漏", 通常是你想要避免的. 然而, 在有些情况下, 你确实想要泄漏内存. 例如, 你有一个整个程序都能访问的只读配置. 你可以把它分配到堆上, 然后用`Box::leak`显式地泄露它, 以获得一个`'static`生命周期的引用.

#### 静态内存

"静态内存"其实是一个统称, 指的是你程序编译后生成的文件中几个密切相关区域. 当程序执行时, 这些区域会自动加载到程序的内存中. 静态内存中的值在程序的整个执行过程中一直存在. 程序的静态内存包含程序的二进制代码, 这部份通常映射为只读. 程序在运行过程中, 会逐条执行文本段中的二进制指令, 并在调用函数时跳转. 静态内存还包含用`static`关键字声明的变量, 以及代码中的某些常量值, 比如字符串.

特殊的生命周期`'static`, 其名称来源于静态内存区域, 表示某个引用在"静态内存还存在的整个期间"都是有效的, 也就是直到程序结束为止. 由于静态变量的内存是在程序启动时就被分配, 因此根据定义, 指定静态内存中变量的引用就是`'static`, 因为在程序结束之前它不会被释放. 反之则不成立, 某些`'static`引用并不一定指向静态内存——但这个名字仍然是合理的: 一旦你创建了一个具有`'static`生命周期的引用, 就程序的其他部分的角度来看, 它所指向的内容就等同于在静态内存中一样, 因为它可以在程序的任意时刻安全使用, 直到程序结束.

在使用`Rust`时, `'static`生命周期比真正的静态内存(例如, 通过`static`关键字)更常见. 这是因为`static`经常出现在类型参数的`trait` 约束中. 像`T: 'static`这样的约束表示, 类型参数`T`能够存活足够长的时间, 甚至可以一直存在到程序结束. 本质上, 这个约束要求`T`是拥有所有权且自给自足的, 要么它不借用任何其他(非`'static`)值, 要么它借用的值也是`'static`, 从而确保它们可以存活到程序结束. 一个`'static`作为约束的好例子是`std::thread::spawn`函数, 它用于创建了一个新的线程, 它要求传递给它的闭包是`'static` . 由于新线程的生命周期可能比当前线程更长, 因此它不能引用存储在当前线程堆栈上的任何内容. 新线程只能引用在其整个生命周期内(可能是在程序的剩余时间内)都有效的值.

> 注意: 您可能想知道 `const` 与 `static` 有何不同. `const` 关键字将其后定义的项声明为常量. 常量项可以在编译时完全计算出来, 任何引用它们的代码都将在编译期间被替换为该常数的计算值. 常量没有与之关联的内存或其他存储空间(它不是一个位置). 您可以将`const`看作是特定值的一个方便的名称.

## 所有权

`Rust`的内存模型的核心思想是: 值只有一个唯一的拥有者, 也就是说, 只有一个位置(通常是一个作用域)负责最终释放每个值的内存. 这一点通过借用检查器强制执行的. 如果一个值被移动, 比如将其赋值给一个新变量、推入`vector`(向量)或放在堆上, 则它的所有权将从旧位置移动到新位置. 此时, 尽管构成该值的位置在技术上仍然存在, 但你不能再通过原始拥有者的变量访问该值. 相反, 您必须通过指向新位置的变量来访问已被移动的值.

有些类型是"叛逆者", 它们不遵守这条规则. 如果一个值的类型实现了特殊的`Copy trait`, 即使它被重新分配到一个新的内存位置, 它也不会被视为移动. 相反, 该值会被复制, 新旧位置仍然都可访问. 从本质上说, 在移动的目的地构造了另一个完全相同的实例. `Rust`中大多数基本类型(比如整数和浮点类型)都是`Copy`类型. 要成为`Copy`类型, 必须能够通过复制位(bit)来复制该类型的值. 这排除了所有包含非`Copy`类型的类型, 以及那些在值被释放时需要释放资源的类型.

要理解其中的原因, 考虑一下如果像 `Box` 这样的类型是`Copy`会发生什么. 如果我们执行 `box2 = box1`, 那么 `box1` 和 `box2` 都会认为他们拥有为为该`box` 分配的堆内存, 当他们超出作用域时都会试图释放这块内存. 重复释放内存可能会产生灾难性的后果.

当一个值的拥有者不再需要它时, 清理这个值的责任就落在该所有者身上, 具体方式是将其释放掉. 在`Rust`中, 当持有该值的变量超出作用域时, 就会自动释放该值. 类型通常递归地释放它们所包含的值, 因此释放一个复杂类型的变量可能会导致许多值被释放. 由于`Rust`对所有权的严格要求, 我们不会意外地多次释放同一个值. 一个变量如果只是持有另一个值引用, 它并不拥有该值, 因此当变量被释放时, 该值不会被释放.

清单 2-3 中的代码给出了围绕所有权、移动和复制语义以及放弃的规则的快速总结.

```rust
let x1 = 42;
let y1 = Box::new(84);
{  // 开始一个新的作用域
  let z = (x1, y1); // (1)
  // z 离开作用域, 并被释放; 
  // 它一次析构了 x1 和 y1 中的值
} // (2)
// x1 的值是 Copy 语义,  所以它不会移动给 z
let x2 = x1; // (3)
// y1 的值不是 Copy 语义, 所以它会移动给 z
// let y2 = y1; // (4)

// 清单 2-3: 移动和复制语义
```

一开始我们有两个值, 数字`42`和包含数字`84`的`Box`(堆分配的值). 前者是`Copy`类型, 而后者不是. 当我们将`x1`和`y1`放入元组`z1`时, `x1`被复制到`z` 中, 而`y1` 被移动到`z` 中. 此时, `x1`仍然可以访问, 并可以再次使用(3) . 另一方面, 一旦`y1`的值被移动到(4), 它就变得不可访问, 任何尝试访问它都将导致编译器错误. 当`z`超出范围作用域(2)时, 元组所包含的值将被释放, 这将进一步释放从`x1`复制的值和从`y1`移动的值. 当`y1`中的`Box`被释放时, 它还释放用于存储`y1`值的堆内存.

>释放顺序
>
> `Rust`会自动释放超出作用域的值, 比如清单2-3中内部作用域的`x1`和`y1`. 释放顺序的规则相当简单: 变量(包括函数参数)按相反的顺序释放, 嵌套值按源代码的顺序释放.
>
> 这听起来可能很奇怪, 为什么会有这样的差异? 不过, 如果我们仔细研究一下, 就会发现它很有道理. 假设你编写了一个函数, 声明了一个字符串, 然后将该字符串的引用插入到一个新的哈希表中. 当函数返回时, 哈希表必须先被释放; 如果字符串先被释放, 那么哈希表就会持有一个无效的引用! 一般来说, 后来的变量可能包含对早期值的引用, 而由于`Rust`的生命周期规则, 反之则不能发生. 出于这个原因, `Rust`以相反的顺序释放变量.
>
> 现在, 我们可以对嵌套的值(如元组、数组或结构体中的值)有同样的行为, 但这可能会让用户感到惊讶. 如果你构建了一个包含两个值的数组, 如果数组的最后一个元素先被释放, 那就显得很奇怪. 这同样适用于元组和结构, 最直观的行为是第一个元组元素或字段先被释放, 然后是第二个, 以此类推. 与变量不同的是, 在这种情况下没有必要颠倒释放顺序, 因为`Rust`(目前)不允许在单个值中进行自我引用. 所以, `Rust`采用了直观的选项.

## 借用和生命周期

`Rust`允许一个值的拥有者通过引用将该值借给其他人, 而不放弃所有权. 引用是一种指针, 但附带了额外的使用约束, 比如该引用是否提供了对被引用值的独占访问, 或者该被引用值是否可以同时有其他引用指向它.

### 共享引用

共享引用(`&`), 顾名思义, 是一种可以被共享的指针. 可以同时存在任意数量的共享引用指向同一个值, 并且共享引用都是可`Copy`, 因此您可以轻松地创建更多的共享引用. 共享引用指向的值是不可变的; 您不能修改或赋值一个共享引用指向的值, 也不能将共享引用强制转换为可变引用.

`Rust`编译器假设: 共享引用指向的值在引用存在期间, 它所指向的值就不会被改变. 例如, 如果`Rust`编译器发现某个函数多次读取同一个共享引用背后的值, 那么它完全有权只读取一次并重用该值. 更具体地说, 清单2-4中的断言应该永远不会失败.

```rust
fn cache(input: &i32, sum: &mut i32) {
    *sum = *input + *input;
    assert_eq!(*sum, 2 * *input);
} 

// 清单 2-4: Rust 假设共享引用是不可变的. 
```

编译器是否选择应用某个特定的优化, 在很大程度上其实是无关紧要的. 因为编译器的优化策略(启发式规则)会随着时间不断变化, 所以你编写代码时, 应依据编译器"允许做什么"来进行思考, 而不是根据编译器在特定时间特定情况下会做什么来编写代码.

### 可变引用

共享引用的另一种选择是可变引用: `&mut T`. 对于可变引用, `Rust`编译器同样充分利用这种引用所附加的约束: 编译器假定在可变引用存在期间, 不会有其他线程通过共享引用或其他可变引用访问这个目标值. 换句话说, 它假定这个可变引用是独占的. 这使得编译器能够执行一些在其他语言中难以实现的有趣优化. 以清单2-5中的代码为例.

```rust
fn noalias(input: &i32, output: &mut i32) {
    if *input == 1 {
        *output = 2; // (1)
    } if *input != 1 {  // (2)
        *output = 3;
    }
}

// 清单 2-5:  Rust 假设可变借用是独占的
```

在`Rust`中, 编译器可以假定`input`和`output`不会指向同一块内存. 因此, (1)处对`output`的重新赋值不能影响(2)处的检查, 整个函数可以被编译为一个简单的 `if-else` 块. 如果编译器不能依赖可变引用的独占性约束, 那么这种优化就会失效, 因为在`noalias(&x, &mut x)`这样的用户, (1)的`input`可能导致`output`为3.

一个可变引用只允许修改引用指向的那块内存位置. 至于你是否可以改变该引用本身更深层的值, 取决于中间类型所提供的方法. 用一个例子可能更容易理解, 所以考虑清单 2-6.

```rust
let x = 42;
let mut y = &x; // y &i32 类型
let z = &mut y; // z 是 &mut &i32 类型

// 清单 2-6: 可变性只适用于直接引用的内存
```

在这个例子中, 你可以通过引用不同的变量, 将指针`y`的值改为不同的值(即不同的指针), 但不能改变指针所指向的值(即`x`的值). 同样, 你可以通过`z`改变`y`的指针值, 但不能改变`z`本身, 让它持有另一个引用.

拥有一个值与拥有该值的可变引用的主要区别在于, **拥有者有责任在不再需要该值时负责将其释放**. 除此之外, 你可以通过一个可变引用做任何事情, 就像你拥有这个值的一样, 但有一个注意事项: **如果你移动了可变引用后面的值, 那么你必须在它的位置上留下另一个值, 如果你没有这样做, 值的拥有者仍然会认为它需要释放这个值, 但是那时却没有值可以释放**.

清单 2-7 给出了将值移动到可变引用后面的方法示例.

```rust
fn replace_with_84(s: &mut Box<i32>) {
    // 这是不可能的, 因为 *s 会变成空:
    // let was = *s; // (1)
    // 但是这可以: 
    let was = std::mem::take(s); // (2)
    // 这也可以: 
    *s = was; // (3)
    // 可以在 &mut 后面交换值: 
    let mut r = Box::new(84);
    std::mem::swap(s, &mut r); // (4)
    assert_ne!(*r, 84);
}

let mut s = Box::new(42);
replace_with_84(&mut s);
// (5)

// 清单 2-7: 可变性仅适用于直接引用的内存. 
```

我添加了代表非法操作的注释行, 你不能简单地将值移出`1`, 因为调用者仍然认为他们拥有该值, 并将会在`5`处再次释放它, 导致双重释放. 如果你只是想留下一些有效的值, `std::mem::take`(2)是一个不错的选择. 它相当于`std::mem::replace(&mut value, Default::default())`; 它将值从可变引用后面移出, 但为该类型留下一个新的默认的值. 默认值是一个独立的、被拥有的值, 所以当作用域在`5`处结束时, 调用者可以安全地释放它.

另外, 如果你不需要引用后面的旧值, 你可以用一个你已经拥有的值覆盖它(3), 让调用者稍后再释放这个值. 当你这样做的时候, 原先在可变引用后面的值会被立刻释放.

最后, 如果你有两个可变的引用, 你可以交换它们的值(4), 即使你不拥有它们中的任何一个, 因为两个引用最后都会有一个合法的值, 供它们各自的拥有者在之后释放.

### 内部可变性

有些类型提供**内部可变性**, 意思是它们允许你通过共享引用来修改值. 这些类型通常依赖于额外的机制(如原子CPU指令)或不变性, 来在不依赖独占引用语义的情况下, 提供安全的可变性. 这些类型通常分为两类: 一类允许您通过共享引用获得一个可变引用, 另一类允许通过共享引用的情况来替换某个值.

第一类包括`Mutex`和`RefCell`这样的类型, 它们包含有安全机制, 来确保对于它们提供可变引用的任何值, 同一时刻只能存在一个可变引用, 并且不能用任何共享引用. 在底层, 这些类型(以及类似的类型)都依赖于一个名为`UnsafeCell`的类型, 它的名称会立即让你对使用它保持谨慎. 我们将在第10章中更详细地介绍`UnsafeCell`, 但现在你应该知道, 这是通过共享引用进行可变操作的唯一正确方法.

提供内部可变性的另一种类型, 是那些不会返回内部值的可变引用的类型, 而只是提供操作该值的方法. `std::sync::atomic`中的原子整数类型和`std::cell::cell`类型就属于这一类. 您无法直接获取这种类型后面的`usize`或`i32`的引用, 但是可以在在某个时间点读取和替换它的值.

> NOTE: 标准库中的`Cell`类型是通过不变式实现安全内部可变性的一个有趣的例子. 它无法跨线程共享, 也不提供对`Cell`所含值的引用. 相反, 它的方法完全替换内部值, 要么返回该值的副本. 由于内部值不能存在任何引用, 因此随时都可以安全地移动该值. 而且, 由于`Cell`不能在多个线程间共享, 因此即使通过共享引用进行修改, 其内部值也不会被并发修改.

### 生命周期

如果您正在阅读这本书, 您可能已经熟悉了生命周期的概念, 很可能是通过编译器不断提示违反了生命周期规则的经历中学到了. 这种程度的理解已经足够应对你将要编写的大部分`Rust`代码, 但随着我们深入研究`Rust`更复杂的部分, 你将需要一个更严谨的思维模型来应对生命周期问题.

`Rust`开发新手经常被教导把生命周期为作用域相对应物: 生命周期从对创建某个变量的引用开始, 到该变量被移动或超出作用域时结束. 这种理解通常是正确的且有用的, 但实际情况要更复杂一些. 生命周期其实是某个引用在有效的代码区域的名称. 虽然生命周期经常与作用域重合, 但它们不必完全一致, 这一点我们稍后会看到.

#### 生命周期和借用检查器

`Rust`生命周期的核心是借用检查器. 每当使用某个生命周期为`'a`的引用时, 借用检查器都会检查`'a`是否仍然有效. 它会沿着引用的使用路径回溯到 `'a` 开始的地方, 即引用的起始点, 并检查该路径上是否有冲突的使用. 这样可以确保引用仍然指向一个可以安全访问的值. 这类似于我们在本章前面讨论的高级"数据流"抽像思维模型; 编译器检查我们正在访问的引用流不会与任何其他并行的流相冲突.

清单 2-8 显示了一个简单的代码例子, 其中有对`x`的引用的生命周期注释.

```rust
let mut x = Box::new(42);
let r = &x;   // (1)            // 'a
if rand() > 0.5 {
    *x = 84;  // (2)
} else {
    println!("{}", r); // (3)   // 'a
}
// (4)

// 清单 2-8: 生命周期不需要是连续的
```

当我们对`x`进行引用时, 生命周期从位置(1)开始. 在第一个分支(2)中, 我们立即尝试修改`x`, 将其值更改为`84`, 这需要一个`&mut x`的可变引用. 借用检查器取出`x`的可变引用并立即检查其使用情况. 它发现在获取引用和使用引用之间没有冲突, 所以它接受代码. 如果你习惯了将生命周期理解为作用域, 这里可能会感到意外, 因为`r`在位置(2)仍然在作用域中(在(4)退出作用域). 但是借用检查器足够聪明, 它意识到如果这个分支被执行, 以后就不会再使用`r`, 因此在这里对`x`进行可变访问是没有问题的. 或者换一种说法, 在(1)处创建的生命周期不会扩展到这个分支: 没有来自`r`(2)之后的流, 因此不存在冲突流. 然后借用检查器在(3)的打印语句中找到了对`r`的使用. 它沿着路径返回到(1), 并发现没有冲突的用途((2)不在该路径上), 所以它也接受这种使用.

如果我们在清单2-8中在(4)处再添加一个对`r`的使用, 代码将无法编译. 生命周期`'a`将从(1)一直持续到(4)(`r` 的最后一次使用), 当借用检查器检查`r`的新的使用时, 它会在(2)处发现一个冲突的使用.

生命周期可以变得相当复杂. 在清单2-9中, 你可以看到一个有漏洞的生命周期的例子, 它在开始和最终结束的地方间歇性地失效了

```rust
let mut x = Box::new(42);
let mut z = &x;        // (1)   // 'a
for i in 0..100 {
    println!("{}", z); // (2)   // 'a
    x = Box::new(i);   // (3)
    z = &x;            // (4)   // 'a
} 
println!("{}", z);              // 'a

// 清单 2-9: 生命周期有漏洞
```

当我们获取`x`的引用时, 生命周期从(1)开始. 然后我们在(3)处移出`x`, 这将结束生命周期`'a`, 因为它的引用不再有效. 借用检查器认为`'a`在(2)处已经结束, 这使得`x` 和(3)之间没有冲突流. 然后, 通过更新`z`在(4)中的引用, 从而重新启动了生命周期. 无论代码现在是循环回到`2`, 还是继续到最后的`println!`语句, 这两个使用点现在都有一个合法的值来源, 而且不存在冲突的数据流, 因此借用检查器接受了该代码.

再次说明, 这与我们之前讨论的内存数据流模型完全一致. 当`x`被移动了, `z` 就不存在了. 当我们稍后重新赋值`z`时, 其实是我们创建了一个全新的变量, 它只从那一刻开始存在. 以这种模型来看, 这个例子就显得不奇怪了.

> NOTE: 借用检查器是本身是是保守的, 而且必须如此. 如果它不能确定一个借用是否合法, 它就会拒绝它, 因为允许非法借用的后果可能是灾难性的. 借用检查器正在不断变得更智能, 但有些时候它仍然需要我们“解释”某个借用为何是合法的. 这就是为什么我们有`unsafe`的`Rust`的部分原因.

#### 泛型生命周期

有时你需要在自定义的类型中存储引用, 这些引用需要有一个生命周期, 以便借用检查器可以在该类型的方法中使用这些引用时检查其有效性. 如果你希望类型的方法返回一个比`self`引用更长的生命周期时, 尤其重要.

`Rust` 允许你像对类型参数进行泛型那样, 对一个或多个生命周期参数进行定义. **Steve Klabnik** 和 **Carol Nichols** 合著的<<`The Rust Programming Language`>>(No Starch Press, 2018)详细介绍了这个主题, 所以我在此不再赘述基本知识. 但是, 当您编写更复杂的此类类型时, 有两个关于类型和生命周期之间的交互的细节值得你注意.

首先, 如果你的类型也实现了`Drop`, 那么释放你的类型也算使用你的类型泛型的任意生命周期或类型. 本质上来讲, 当你的类型的实例被释放时, 借用检查器会检查此时是否仍然合法地使用任何泛型生命周期参数. 这是必要的, 因为你的`Drop`代码确实可能会使用到这些引用. 如果你的类型没有实现 `Drop`, 释放这个类型就不算使用它里面的引用, 并且只要用户不再使用你的类型实例, 就可以忽略存储在其中的引用, 就像我们在清单2-7中看到的那样. 我们将在第10章中更多地讨论这些关于`Drop`的规则.

其次, 尽管一个类型可以对多个生命周期进行泛型化, 但这么做通常只会让类型签名不必要地复杂化. 通常情况下, 一个类型只对单个泛型生命周期就足够了, 并且编译器会选择所有插入到该类型中的引用中生命周期中较短的那个, 作为该生命周期参数的实际值. 你应该只在以下情况中使用多个生命周期参数: 当你的类型内部包含多个引用, 并且类型的方法需要返回一个只与其中某个引用的生命周期相关的引用时.

请看清单2-10中的类型, 它为您提供了一个迭代器, 迭代器将遍历被其它特定字符串分隔.

```rust
struct StrSplit<'s, 'p> {
    delimiter: &'p str,
    document: &'s str,
}
impl<'s, 'p> Iterator for StrSplit<'s, 'p> {
    type Output = &'s str;
    fn next(&self) -> Option<Self::Output> {
        todo!()
    }
}
fn str_before(s: &str, c: char) -> Option<&str> {
    StrSplit { document: s, delimiter: &c.to_string() }.next()
}

// 清单 2-10:  一个需要多个泛型生命周期的类型
```

当你构造这个类型时, 你需要提供要搜索的`delimiter`和`document`, 它们都是对字符串值的引用. 当你要求搜索下一个字符串时, 返回的是对`document`的引用. 想像一下, 如果你在这个类型中使用单一生命周期会发生什么. 迭代器产生的值会被绑定到`document`和`delimiter`共同的生命周期上. 这将使`str_before`这样的函数无法编写: 因为它的返回类型的生命周期与函数局部变量的生命周期相关联--也就是由`to_string`创建出来的`String`--而借用检查器将拒绝该代码.

#### 生命周期型变

"型变"(`Variance`)是程序员经常接触到的一个概念, 但很少知道它的名称, 因为它大多时候是隐形的. 简单来说, 变量描述了哪些类型是其他类型的子类型, 以及何时可以用子类型代替父类型(反之亦然). 一般来说, 如果类型`A`至少和类型`B`一样有用, 那`A`就是`B`的子类型. 在`Java`中, 如果`Turtle`是`Animal`的子类型, 你可以把`Turtle`传给接受`Animal`的函数, 或者在`Rust`中, 你可以把一个`&'static str`传给接受`&'a str`的函数.

虽然型变通常隐藏在幕后, 但它经常出现, 我们需要对它工作原理有所了解. `Turtle`是`Animal`的一个子类型, 因为`Turtle`比某些不确定的`Animal`更"有用"--`Turtle`可以做任何`Animal`能做的事, 而且可能更多. 同样,`'static`是`'a`的一个子类型, 因为`'static`的生命周期至少与任何`'a`一样长, 所以更有用. 或者, 更一般地说, 如果`'b:'a`(`'b`比`'a`具有更长的生命周期), 那么`'b`就是`'a`的一个子类型. 这显然不是正式的定义, 但是它已经足够接近, 具有实际指导意义.

所有类型都有一个型变, 它定义了在某个类型的位置上可以使用哪些相似的类型. 型变分为三种: **协变**(`covariant`)、**不变**(`invariant`)和**逆变**(`contravariant`).

- 如果你可以用一个子类型来替代该类型, 那么这个类型就是**协变**的. 例如, 如果一个变量是`&'a T`类型, 你可以给它提供一个`&'static T`类型的值, 因为`&'a T` 在`'a`上是协变的. `&'a T`在`T`上也是协变的, 所以你可以把一个`&Vec<&'static str>`传递给一个接受`&Vec<&'a str>`的函数.
- 有些类型是**不变**的, 这意味着你必须准确提供完全相同的类型. `&mut T`就是一个例子--如果一个函数接受一个`&mut Vec<&'a str>`, 你不能把一个`&mut Vec<&'static str>`传给它. 也就是说,`&mut T`在`T`上是不变的. 如果你这样做, 该函数可能在`Vec`中放入一个短生命周期的字符串, 然后调用者会继续使用它, 错误的认为它是一个`Vec<&'static str>`, 从而认为包含的字符串是`'static`!. **任何提供可变性的类型一般都是不变的**, 原因也是如此, 例如, `Cell<T>` 在`T`上是不变的.
- 最后一种类别, **逆变**, 是针对函数参数而言的. 如果函数类型的参数不那么通用, 那么它们就会更有用. 如果你将参数类型本身的型变与它们作为函数参数时的型变进行对比, 这一点就更清楚了:

```rust
let x: &'static str; // 更有用, 活的更长
let x: &'a str; // 不太有用, 活得更短
fn take_func1(&'static str) // 更严格, 所以不那么有用
fn take_func2(&'a str) // 不太严格, 所以更有用
```

这种相返的关系表明, `Fn(T)`在`T`上是逆变的.

那么, 当涉及到生命周期时, 为什么需要学习型变呢? 当您考虑泛型生命周期参数如何与借用检查器交互时, 型变就变得非常重要了.
考虑清单2-11所示的类型, 它在一个字段中使用多个生命周期.

```rust
struct MutStr<'a, 'b> {
    s: &'a mut &'b str
}
let mut s = "hello";
*MutStr { s: &mut s }.s = "world"; // (1)
println!("{}", s);

// 清单 2-11: 需要多个泛型生命周期的类型
```

乍一看, 在这里使用两个生命周期似乎是多余的--们并没有像在清单2-10中的 `StrSplit` 那样, 需要通过方法来区分结构中不同部分的借用, 但是如果你把这里的两个生命周期换成一个`'a`, 代码就不再能被编译了! 这就是为什么我们在这里使用了两个生命周期. 而这一切都是因为型变.

> 注意: (1)处的语法可能看起来很奇怪. 它相当于定义了一个持有 `MutStr` 的变量 `x`, 然后写 `*x.s = "world"`, 只是没有变量, 所以 `MutStr` 被立即销毁了.

在(1)处, 编译器必须确定生命周期参数应该被设置哪个生命周期. 如果有两个生命周期, `'a`被设置为有待确定的`s`的借用生命周期, `'b`被设置为`'static`, 因为那是提供的字符串"hello"的生命周期. 如果只有一个生命周期`'a` , 编译器推断该生命周期必须是`'static`.

当我们后来试图通过共享引用访问字符串引用`s`来打印它时, 编译器试图缩短`MutStr`使用的`s`的可变借用, 以允许`s`的共享借用.

在双生命周期的情况下, `'a`只是在`println!`之前结束, `'b`保持不变. 另一方面,在单生命周期的情况下, 我们遇到了一些问题. 编译器想缩短`s`的借用时间, 但为此也必须缩短`str`的借用时间. 虽然`&'static str`一般来说可以缩短为任何`&'a str`(`&'a T` 在`'a`中是协变的), 但这里它在`&mut T`后面, 而`&mut T`在`T`中是不变量的. 不变要求相关类型永远不会被子类型或父类型取代, 所以编译器缩短借用的尝试失败了, 它报告说这个仍然是可变的借用, 哎哟!

由于不型带来的灵活性的降低, 你想确保你的类型在尽可能多的泛型参数上保持协变(或在适当情况下保持逆变). 如果这需要引入额外的生命期参数, 你需要仔细权衡增加一个新参数的认知成本和型变的人体工程学代价.

## 总结

本章的目标是建立一个坚实且统一的基础, 以便我们在接下来的章节中能够继续深入构建. 到现在, 我希望你已经牢牢掌握了`Rust`的内存和所有权模型, 而之前那些你从借用检查器那里遇到的报错, 现在看起来也不那么神秘了. 你或许之前已经了解了我们所讲内容的一部分, 但希望这一章能为你构建一个更完整的整体概念, 展示它们是如何协同工作的. 在下一章中, 我们将对类型系统进行类似的深入探讨. 我们将了解类型在内存中的表示方式, 了解泛型和`trait`是如何产生实际运行代码的, 还会看看`Rust`为更高级的用例使用场景提供的一些特殊类型和`trait`结构.
