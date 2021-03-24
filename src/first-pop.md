# 弹出

和`push`一样，`pop`想要改变列表。与`push`不同的是，我们实际上想返回一些东西。
但`pop`还需要处理一个棘手的角落情况：如果链表是空的怎么办？为了表示这种情况，
我们使用可靠的`Option`类型：

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    // TODO
}
```

`Option<T>`是一个枚举，表示一个可能存在的值。它可以是`Some(T)`，也可以是`None`。
我们可以像对待Link一样，自己做一个枚举，但是我们希望我们的用户能够理解我们的返回类
型到底是什么，Option是如此的无处不在，以至于*每个人*都知道它。事实上，它是如此的基
本，以至于在每个文件中都会隐式地导入到scope中，还有它的变体`Some`和`None`（所以我
们不必说`Option::None`）。

`Option<T>`上的尖头表示Option实际上是在T上*通用*的，这意味着你可以为*任何*类型制
作一个Option!

那么，呃，我们有这个`Link`的东西，我们怎么知道它是空的还是有More的？模式匹配与
`match`!

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
}
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/first.rs:27:30
   |
27 |     pub fn pop(&mut self) -> Option<i32> {
   |            ---               ^^^^^^^^^^^ expected enum `std::option::Option`, found ()
   |            |
   |            this function's body doesn't return
   |
   = note: expected type `std::option::Option<i32>`
              found type `()`
```

糟糕，`pop`必须返回一个值，而我们还没有这样做。我们*其实可以*返回`None`，但在这种
情况下，返回`unimplemented!()`可能是一个更好的主意，以表明我们还没有完成函数的实
现。`unimplemented!()`是一个宏(`!`表示一个宏)，当我们到达它时，它会使程序惊慌失
措(\~以一种可控的方式使它崩溃)。

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

无条件恐慌是[发散函数][diverging]的一个例子。发散函数永远不会返回给调用者，所以它
们可以用在期待任何类型的值的地方。这里，`unimplemented!()`被用来代替一个
`Option<T>`类型的值。

还要注意，我们的程序中不需要写`return`。一个函数中的最后一个表达式（基本上是行）隐
含了它的返回值。这让我们可以更简洁地表达真正简单的东西。你总是可以像其他类C语言一
样，显式地用`return`提前返回。

```text
> cargo build

error[E0507]: cannot move out of borrowed content
  --> src/first.rs:28:15
   |
28 |         match self.head {
   |               ^^^^^^^^^
   |               |
   |               cannot move out of borrowed content
   |               help: consider borrowing here: `&self.head`
...
32 |             Link::More(node) => {
   |                        ---- data moved here
   |
note: move occurs because `node` has type `std::boxed::Box<first::Node>`, which does not implement the `Copy` trait
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^
```

来吧Rust，别烦我们了。一如既往，Rust对我们很生气。值得庆幸的是，这次它也给了我们完
整的信息! 默认情况下，模式匹配会尝试将其内容移动到新的分支中，但我们不能这样做，因
为我们在这里没有拥有self的值。

```text
help: consider borrowing here: `&self.head`
```

Rust说我们应该在`match`中添加一个引用来解决这个问题。🤷‍♀️让我们试试吧：

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match &self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(ref node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

```text
> cargo build

warning: unused variable: `node`
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^ help: consider prefixing with an underscore: `_node`
   |
   = note: #[warn(unused_variables)] on by default

warning: field is never used: `elem`
  --> src/first.rs:13:5
   |
13 |     elem: i32,
   |     ^^^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/first.rs:14:5
   |
14 |     next: Link,
   |     ^^^^^^^^^^
```

万岁，又能编译了! 现在让我们弄清楚这个逻辑。我们想做一个Option，所以让我们为它做一
个变量。如果是Empty，我们需要返回None。如果是More，我们需要返回`Some(i32)`，并改
变列表的头。那么，让我们试着简单做到这一点？

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match &self.head {
        Link::Empty => {
            result = None;
        }
        Link::More(ref node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
> cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:35:29
   |
35 |                 self.head = node.next;
   |                             ^^^^^^^^^ cannot move out of borrowed content

```

*脑袋*

*办公桌*

我们正试图把东西移出`node`，但我们只有它的共享引用。

我们也许应该退一步，想想我们现在到底想做什么。我们想做的是：

* 检查列表是否为空。
* 如果是空的，就返回None
* 如果它*不是*空的
    * 移除链表的头
    * 移除它的`elem`
    * 用它的`next`代替列表的头
    * 返回`Some(elem)`

我们主要的想法是要*删除*事物，这意味着我们要*通过值*来获得列表的头部。我们当然不能通过我们通过
`&self.head`得到的共享引用来做到这一点。我们也"只"有一个对`self`的可变引用，所以我们移动东西
的唯一方法就是*替换它*。看来我们又在跳Empty舞了！

让我们试试吧：

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

O M G

它在编译时没有*任何*警告!!!!!

其实我在这里要套用一下我个人的套路：我们返回了这里的`result`，但其实我们根本不
需要这么做！就像一个函数运算结果是它的最后一个表达式一样，每个块的运算结果也是它
的最后一个表达式。通常我们会用分号阻止这种行为，这样反而会使代码块运算为空元组，
`()`。这实际上是不声明返回值的函数--比如`push`--返回的值。

因此，我们可以将`pop`写成：

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => None,
        Link::More(node) => {
            self.head = node.next;
            Some(node.elem)
        }
    }
}
```

这更简洁、更惯用。请注意，Link::Empty分支完全失去了括号，因为我们只有一个表达式
需要运算。只是一个很好的简写，用于简单的情况。

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

好，仍能运行！



[ownership]: first-ownership.html
[diverging]: https://doc.rust-lang.org/nightly/book/ch19-04-advanced-types.html#the-never-type-that-never-returns
