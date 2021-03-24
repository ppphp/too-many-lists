# 推入

`push`会使链表发生*改变*，所以我们需要使用`&mut self`。我们还需要取一个i32
来推入：

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        // TODO
    }
}
```

首先，我们需要做一个节点来存储我们的元素：

```rust ,ignore
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: ?????
        };
    }
```

`next`是什么？嗯，整个旧链表！我们能不能... ... 就这么做？

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: self.head,
        };
    }
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

不——。Rust正告诉我们正确的事情，但它到底是什么意思，或者对它该怎么做，肯定不太显然：

> 不能移出借用的上下文

我们想把`self.head`字段移到`next`，但Rust不希望我们这么做。这将使`self`在我们结束
借用并将其“还给”它的合法所有者时，只被部分初始化。正如我们之前说过的，这是你不能用
`&mut`做的*一件*事：这将是超级无礼的，而Rust是非常有礼貌的（这也将是难以置信的危险
，但*这*肯定不是它关心的原因）。

如果我们把东西放回去呢？也就是我们正在创建的节点：


```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: self.head,
    });

    self.head = Link::More(new_node);
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

没办法。原则上，这是Rust可以接受的，但它不会接受（出于各种原因--最严重的是[例外安全][]）
。我们需要一些方法在Rust没有注意到它消失的情况下得到头。为了寻求建议，我们求助于臭名昭著
的Rust黑客Indiana Jones：

![Indy Prepares to mem::replace](img/indy.gif)

啊，对了，Indy建议使用`mem::replace`手法。这个非常有用的函数可以让我们通过用另一个值*替
换*来从一个借用中偷取出来一个值。我们在文件的顶部拉入`std::mem`，让`mem`在本地作用域内：

```rust ,ignore
use std::mem;
```

并适当地使用它：

```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: mem::replace(&mut self.head, Link::Empty),
    });

    self.head = Link::More(new_node);
}
```

在这里，我们用Link::Empty暂时`replace`到self.head，然后再把self.head换成列表的新头。
我不会撒谎：这是一件非常不幸的事情。很遗憾，我们必须这么做（目前）。

但是，嘿，`push`全部完成! 可能吧。我们也许应该测试一下，说实话。现在，最简单的方法
可能是写`pop`，并确保它产生正确的结果。





[例外安全]: https://doc.rust-lang.org/nightly/nomicon/exception-safety.html
