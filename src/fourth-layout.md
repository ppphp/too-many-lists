# 布局

我们设计的关键是`RefCell`类型。RefCell的核心是一对方法：

```rust ,ignore
fn borrow(&self) -> Ref<'_, T>;
fn borrow_mut(&self) -> RefMut<'_, T>;
```

`borrow`和`borrow_mut`的规则正是`&`和`&mut`的规则：你可以随意调用`borrow`，但
`borrow_mut`需要独占性。

RefCell不是静态地执行这些规则，而是在运行时执行这些规则。如果你违反了这些规则，
RefCell就会惊慌失措，使程序崩溃。为什么它会返回这些Ref和RefMut的东西？好吧，它们的
行为基本上和`Rc`s一样，只是用于借用。它们也保持RefCell的借用，直到它们超出范围。我
们稍后会讨论这个问题。

现在有了Rc和RefCell，我们可以成为......一种令人难以置信的冗长的普遍可变异的垃圾收集
语言，它不能收集循环 呀呀呀...

好了，我们要做*双向链接*的。这意味着每个节点都有一个指向上一个和下一个节点的指针。同
时，列表本身也有一个指向第一个和最后一个节点的指针。这样我们就可以在列表的两端快速插
入和删除。

所以我们可能需要这样的东西：

```rust ,ignore
use std::rc::Rc;
use std::cell::RefCell;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>,
}

type Link<T> = Option<Rc<RefCell<Node<T>>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
    prev: Link<T>,
}
```

```text
> cargo build

warning: field is never used: `head`
 --> src/fourth.rs:5:5
  |
5 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fourth.rs:6:5
  |
6 |     tail: Link<T>,
  |     ^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fourth.rs:12:5
   |
12 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/fourth.rs:13:5
   |
13 |     next: Link<T>,
   |     ^^^^^^^^^^^^^

warning: field is never used: `prev`
  --> src/fourth.rs:14:5
   |
14 |     prev: Link<T>,
   |     ^^^^^^^^^^^^^
```

嘿，它建好了! 很多死代码的警告，但它构建了！让我们试着使用它。
