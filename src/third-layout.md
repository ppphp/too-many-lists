# 布局

好了，回到布局的画板上。

持久列表最重要的一点是，你可以完全没有成本地操作列表的尾部：

例如，在持久列表中，这并不是一个不常见的工作情况：

```text
list1 = A -> B -> C -> D
list2 = tail(list1) = B -> C -> D
list3 = push(list2, X) = X -> B -> C -> D
```

但在最后，我们希望内存看起来像这样：

```text
list1 -> A ---+
              |
              v
list2 ------> B -> C -> D
              ^
              |
list3 -> X ---+
```

这对Boxes来说就是不行的，因为`B`的所有权是*共享*的。谁应该释放它？如果我放弃list2，
它会释放B吗？对于盒子，我们当然希望如此。

函数式语言--事实上几乎所有其他语言--都通过使用*垃圾回收*来解决这个问题。有了垃圾收集
的魔力，只有在每个人都不再看B的时候，它才会被释放。万幸！

Rust没有像这些语言那样的垃圾收集器。他们有*追踪*的GC，它将挖掘所有在运行时闲置的内
存，并自动找出哪些是垃圾。相反，Rust现在有的只是*引用计数*。引用计数可以被认为是一个
非常简单的GC。对于许多工作情况来说，它的吞吐量明显低于跟踪收集器，而且如果你建立一个
循环，它就会完全崩溃。但是，这就是我们所能得到的一切了。值得庆幸的是，对于我们的用例
来说，我们永远不会遇到循环问题（请随时尝试向自己证明这一点 -- 我肯定不会）。

那么，我们如何进行引用计数的垃圾收集呢？`Rc`! Rc就像Box一样，但我们可以复制它，而且
它的内存*只有*在*所有*从它派生出来的Rc被丢弃时才会被释放。不幸的是，这种灵活性有一个
严重的代价：我们只能对其内部进行共享引用。这意味着我们永远不能从我们的一个列表中取出
数据，也不能对它们进行改变。

那么，我们的布局会是什么样子的呢？嗯，之前我们有：


```rust ,ignore
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

我们能不能把Box改成Rc？

```rust ,ignore
// in third.rs

pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Rc<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

```text
cargo build

error[E0412]: cannot find type `Rc` in this scope
 --> src/third.rs:5:23
  |
5 | type Link<T> = Option<Rc<Node<T>>>;
  |                       ^^ not found in this scope
help: possible candidate is found in another module, you can import it into scope
  |
1 | use std::rc::Rc;
  |
```

哦，该死，坏起来了。与我们用于可变列表的一切不同，Rc是如此蹩脚，以至于它甚至没有被
隐式地导入到每一个Rust程序中。*真是个失败者。*

```rust ,ignore
use std::rc::Rc;
```

```text
cargo build

warning: field is never used: `head`
 --> src/third.rs:4:5
  |
4 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

似乎是合法的。Rust依然写起来是*完全*琐碎的。我打赌我们会在某一天可以用Rc查找和替
换Box，然后就可以了。

...

不，不，我们不能。
