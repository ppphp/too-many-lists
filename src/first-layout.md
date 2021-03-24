# 基本数据布局

好吧，那么什么是链表呢？基本上，它是一堆在堆上的数据片(嘘，内核的人!)，它们依次
指向对方。链表是程序员不应该碰的东西，而函数式程序员却用它来做所有事情。那么，我
们应该向函数式程序员询问链表的定义，这似乎很公平。他们可能会给你类似下面的定义：

```haskell
List a = Empty | Elem a (List a)
```

其内容大致为“一个链表要么是空的，要么是一个元素，后面是一个链表”。这是一个递归
定义，表示为*和类型*，这是“一个可以有不同值的类型，这些值可能是不同的类型”的花
哨名字。Rust称和类型为`枚举`! 如果你来自于类C语言，这正是你所熟悉和喜爱的枚举，
但是是超速的。所以让我们把这个函数定义转录到Rust中吧!

现在，我们将避免使用泛型，保持简单。我们将只支持存储有符号的32位整数:

```rust ,ignore
// in first.rs

// pub says we want people outside this module to be able to use List
pub enum List {
    Empty,
    Elem(i32, List),
}
```

*呼*，我被淹没了。我们还是继续编译吧：

```text
> cargo build

error[E0072]: recursive type `first::List` has infinite size
 --> src/first.rs:4:1
  |
4 | pub enum List {
  | ^^^^^^^^^^^^^ recursive type has infinite size
5 |     Empty,
6 |     Elem(i32, List),
  |               ---- recursive without indirection
  |
  = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable
```

好吧，我不知道你是怎么想的，但我确实觉得被函数式编程社区背叛了。

如果我们真正检查一下错误信息（在我们克服了整个背叛的事情之后），我们可
以看到rustc实际上是在告诉我们如何解决这个问题：

> insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable

好吧，`box`。那是什么？让我们谷歌一下`rust box`...

> [std::boxed::Box - Rust](https://doc.rust-lang.org/std/boxed/struct.Box.html)

贴在这里...

> `pub struct Box<T>(_);`
>
> 一个在堆上分配的指针类型。
> 更多信息参见[module-level documentation](https://doc.rust-lang.org/std/boxed/)。

*点击链接*

> `Box<T>`被随意的称为"盒子"，提供了Rust中最简单的堆分配形式。盒子为这种分配提供所有权，并在它们离开范围时放弃它们的内容。
>
> 举例
>
> 创建盒子：
>
> `let x = Box::new(5);`
>
> 创建一个递归数据结构：
>
```rust
#[derive(Debug)]
enum List<T> {
    Cons(T, Box<List<T>>),
    Nil,
}
```
>
```rust
fn main() {
    let list: List<i32> = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
    println!("{:?}", list);
}
```
>
> 这会打印`Cons(1, Box(Cons(2, Box(Nil))))`。
>
> 递归数据类型一定要是盒子，因为如果他们的Cons的定义向这样：
>
> `Cons(T, List<T>),`
>
> 它不会运行。这因为链表的大小取决于链表里有几个元素，如果我们不知道Cons要分配多少内存。通过引入有已知大小的盒子，我们知道Cons会有多大。

哇，呃。这可能是我看过的最相关最有帮助的文档。字面上文档的最重要的事情是*确切地
说明我们正在尝试写什么，为什么它不能正常运行，如何修复它*。

铛，文档说了算。

好吧，让我们这么做：

```rust ,ignore
pub enum List {
    Empty,
    Elem(i32, Box<List>),
}
```

```text
> cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

哈，他构建了！

...但由于某些原因，实际上这是一个愚蠢的链表定义。

考虑有两个元素的链表：

```text
[] = Stack
() = Heap

[Elem A, ptr] -> (Elem B, ptr) -> (Empty, *junk*)
```

有两个关键问题：

* 我们正分配一个只是说“我不是节点”的节点
* 我们的一个节点不是在堆上分配的。

表面上，这两个似乎是相互矛盾的。我们分配一个额外的节点，但是我们其中一个
节点不需要被分配。然而，考虑我们链表可能的以下布局：

```text
[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
```

在这个布局我们现在不可避免地在堆上分配我们的节点。关键区别是不存在我们第一个
布局的*junk*。这个垃圾是什么？为了理解，我们将需要查看一个枚举在内存里是如何
布局的。

总的来说，如果我们有一个枚举如下：

```rust ,ignore
enum Foo {
    D1(T1),
    D2(T2),
    ...
    Dn(Tn),
}
```

一个Foo会存储一些数字来表示枚举中的哪个*变量*代表了(`D1`, `D2`, .. `Dn`)。这是枚
举的*标签*。它也会需要足够的空间存储*最大的*`T1`, `T2`, .. `Tn`（加上一些额外的
空间来满足对齐的需要）。

这里最大的启示是，尽管`Empty`是一个单一的信息位，但它必然要消耗足够的空间来放置一个
指针和一个元素，因为它要随时准备成为一个`Elem`。因此第一个布局的堆里多分配了一个元
素，就是满满的垃圾，比第二个布局多消耗了一点空间。

我们的一个节点完全不被分配，也许令人惊讶的是，也比总是分配它*更糟糕*。这是因为它给我
们提供了一个*非统一*的节点布局。这对推入和弹出节点没有什么明显的影响，但对拆分和合并
链表有影响。

考虑在两种布局中拆分一个链表：

```text
布局1：

[Elem A, ptr] -> (Elem B, ptr) -> (Elem C, ptr) -> (Empty *junk*)

在C处分割：

[Elem A, ptr] -> (Elem B, ptr) -> (Empty *junk*)
[Elem C, ptr] -> (Empty *junk*)
```

```text
布局2：

[ptr] -> (Elem A, ptr) -> (Elem B, ptr) -> (Elem C, *null*)

在C处分割：

[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
[ptr] -> (Elem C, *null*)
```

布局2的拆分只需要将B的指针复制到堆栈，并将旧的值清空。布局1最终也做同样的事情，
但也要把C从堆中复制到栈中。合并则是反过来的相同过程。

链表的几个好处之一是，你可以在节点本身中构造元素，然后在链表中自由打乱，而无需
移动它。你只需要摆弄一下指针，东西就会被“移动”。布局1破坏了这个属性。

好吧，我有理由相信布局1是坏的。我们如何重写我们的链表？好吧，我们可以做这样的事
情：

```rust ,ignore
pub enum List {
    Empty,
    ElemThenEmpty(i32),
    ElemThenNotEmpty(i32, Box<List>),
}
```

希望这在你看来是一个更糟糕的想法。最值得注意的是，这真的让我们的逻辑复杂化了，因为
现在有一个完全无效的状态：`ElemThenNotEmpty(0, Box(Empty))`。它也*仍然*受到非
统一分配我们元素的影响。

然而它有*一个*有趣的特性：它完全避免了分配为空的情况，使堆分配的总数减少了1，不幸
的是，在这样做的时候，它设法浪费了*更多的空间*！这是因为之前的布局利用了*空指针优
化*。

我们之前看到，每个枚举都必须存储一个标签，以指定其位代表枚举的哪个变体。然而，如果
我们有一种特殊的枚举：

```rust,ignore
enum Foo {
    A,
    B(ContainsANonNullPtr),
}
```


空指针优化来了，从而*消除了标签所需的空间*。如果变体是A，则整个枚举都设置为全0，否则
就是B。这样做的原因是B永远不会是所有的0，因为它包含一个非零指针。聪明!

你能想到其他的枚举和类型可以做到这样的优化吗？其实有很多! 这就是为什么Rust对枚举布局
完全不作规定。有一些更复杂的枚举布局优化，Rust会为我们做，但空指针这个绝对是最重要的!
这意味着`&`、`&mut`、`Box`、`Rc`、`Arc`、`Vec`以及Rust中其他几个重要的类型在放入
`Option`时没有任何开销! (我们会在适当的时候讲到其中的大部分)。

那么，我们如何避免额外的垃圾，统一分配，*并*获得甜蜜的空指针优化呢？我们需要更好地将拥
有一个元素的想法与分配另一个列表分开。要做到这一点，我们必须想得更像C语言：结构体!

枚举让我们声明一个可以包含多个值中的*一个*类型，而结构体让我们声明一个同时包含*许多*值
的类型。让我们把链表分成两种类型。一个链表和一个节点。

和之前一样，一个链表要么是空的，要么是有一个元素跟在另一个链表后面。通过用一个完全独立
的类型来表示“有一个被另一个链表跟随的元素”的情况，我们可以将盒子提升到一个更理想的位置：

```rust ,ignore
struct Node {
    elem: i32,
    next: List,
}

pub enum List {
    Empty,
    More(Box<Node>),
}
```

我们来确认优先级：

* 列表尾部永远不会分配额外的垃圾：检查!
* `enum`是美味的空指针优化形式：检查!
* 所有元素都是统一分配的：检查!

好吧！实际上，我们刚刚构建的布局正是我们用来证明我们的第一个布局（正如官方
Rust文档所建议的那样）是有问题的。

```text
> cargo build

warning: private type `first::Node` in public interface (error E0446)
 --> src/first.rs:8:10
  |
8 |     More(Box<Node>),
  |          ^^^^^^^^^
  |
  = note: #[warn(private_in_public)] on by default
  = warning: this was previously accepted by the compiler but
    is being phased out; it will become a hard error in a future release!
```

:(

Rust又对我们发火了。我们把`List`标记为公共的（因为我们希望人们能够使用它），但`Node`
没有。问题是枚举的内部是完全公开的，我们不允许公开谈论私有类型。我们可以将`Node`的所
有内容完全公开，但一般来说，在Rust中，我们倾向于保持实现细节的私有性。让我们把`List`
做成一个结构体，这样我们就可以隐藏实现细节：

```rust ,ignore
pub struct List {
    head: Link,
}

enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}
```


因为`List`是一个只有一个字段的结构体，所以它的大小与该字段相同。耶，零成本抽象!

```text
> cargo build

warning: field is never used: `head`
 --> src/first.rs:2:5
  |
2 |     head: Link,
  |     ^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: variant is never constructed: `Empty`
 --> src/first.rs:6:5
  |
6 |     Empty,
  |     ^^^^^

warning: variant is never constructed: `More`
 --> src/first.rs:7:5
  |
7 |     More(Box<Node>),
  |     ^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/first.rs:11:5
   |
11 |     elem: i32,
   |     ^^^^^^^^^

warning: field is never used: `next`
  --> src/first.rs:12:5
   |
12 |     next: Link,
   |     ^^^^^^^^^^

```

好了，这下编译好了！Rust很生气，因为据它所知，我们写的所有东西都是完全无用的：我们从
来没有用过`head`，使用我们库的人也不能用，因为它是私有的。反过来说，这意味着`Link`和
`Node`也是无用的。所以我们来解决这个问题吧! 让我们为我们的List实现一些代码吧!
