# 构建

好吧，我们先从建立链表开始。在这个新的系统中，这是很直接的事情。`new`仍然简单，
所有的字段都是None。另外，因为它变得有点不方便，我们也来做一个Node构造函数：

```rust ,ignore
impl<T> Node<T> {
    fn new(elem: T) -> Rc<RefCell<Self>> {
        Rc::new(RefCell::new(Node {
            elem: elem,
            prev: None,
            next: None,
        }))
    }
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }
}
```

```text
> cargo build

**A BUNCH OF DEAD CODE WARNINGS BUT IT BUILT**
```

耶!

现在让我们试着把推入写到链表的前面。因为双链表明显更复杂，我们需要做更多的工作。
单链表的操作可以简化为一个简单的单行代码，而双链表的操作则相当复杂。

特别是我们现在需要特别处理一些围绕空列表的边界情况。大多数操作只涉及`head`或
`tail`的指针。然而当过渡到空列表或从空列表过渡时，我们需要*同时*编辑两个列表。

如果我们保持以下的不变性，我们验证我们的方法是否有意义的一个简单方法是：每个节
点应该正好有两个指向它的指针。列表中间的每个节点都被其前任和后任所指向，而两端
的节点则被列表本身所指向。

让我们来试一试：

```rust ,ignore
pub fn push_front(&mut self, elem: T) {
    // new node needs +2 links, everything else should be +0
    let new_head = Node::new(elem);
    match self.head.take() {
        Some(old_head) => {
            // non-empty list, need to connect the old_head
            old_head.prev = Some(new_head.clone()); // +1 new_head
            new_head.next = Some(old_head);         // +1 old_head
            self.head = Some(new_head);             // +1 new_head, -1 old_head
            // total: +2 new_head, +0 old_head -- OK!
        }
        None => {
            // empty list, need to set the tail
            self.tail = Some(new_head.clone());     // +1 new_head
            self.head = Some(new_head);             // +1 new_head
            // total: +2 new_head -- OK!
        }
    }
}
```

```text
cargo build

error[E0609]: no field `prev` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:39:26
   |
39 |                 old_head.prev = Some(new_head.clone()); // +1 new_head
   |                          ^^^^ unknown field

error[E0609]: no field `next` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:40:26
   |
40 |                 new_head.next = Some(old_head);         // +1 old_head
   |                          ^^^^ unknown field
```

好吧。编译器错误。好的开始。好的开始。

为什么我们不能访问我们节点上的`prev`和`next`字段？以前当我们只有一个`Rc<Node>`的
时候，它是可以工作的。似乎`RefCell`妨碍了它。

我们也许应该检查一下文档。

*谷歌一下"rust refcell"*

*[点击第一个链接](https://doc.rust-lang.org/std/cell/struct.RefCell.html)*
> 一个具有动态检查的借用规则的可变的内存位置
>
> 参见[模块级文档](https://doc.rust-lang.org/std/cell/index.html)以了解更多。

*点击链接*

> 可共享的可变异容器。
>
> `Cell<T>`和`RefCell<T>`类型的值可以通过共享引用（即常见的`&T`类型）进行改变
> ，而大多数Rust类型只能通过唯一的（`&mut T`）引用进行变异。我们说`Cell<T>`和
> `RefCell`<T>提供了 "内部可变性"，而典型的Rust类型则表现为 "继承的可变性"。
> 
> 细胞类型有两种风格：`Cell<T>`和`RefCell<T>`。`Cell<T>`提供了`get`和`set`方
> 法，只需调用一个方法就可以改变内部值。不过`Cell<T>`只与实现Copy的类型兼容。对
> 于其他类型，必须使用`RefCell<T>`类型，在改变之前获得一个写锁。
>
> `RefCell`<T>使用Rust的生命周期来实现 "动态借用"，在这个过程中，我们可以要求对
> 内部值进行临时的、独占的、可改变的访问。`RefCell<T>`的借用是在"运行时"跟踪的，
> 而不像Rust的本地引用类型那样完全是在编译时静态跟踪的。因为`RefCell<T>`的借用是
> 动态的，所以有可能试图借用一个已经被变异借用的值；当这种情况发生时，会导致线程恐
> 慌。
>
> # 何时选择内部可变性
>
> 更常见的继承可变性，即必须有唯一的访问权才能改变一个值，这是一个关键的语言元素，
> 使Rust能够强有力地推理指针别名，静态地防止崩溃的错误。正因为如此，继承的可变性是
> 首选，而内部可变性是最后的手段。由于单元格类型能够在原本不允许的地方进行改变，所
> 以在某些情况下，内部变异可能是合适的，甚至是*必须*使用的，比如说
>
> * 为共享类型引入继承的可变性根源。
> * 逻辑上不可变的方法的实现细节。
> * `Clone`的可变实现。
>
> ## 为共享类型引入继承的可变性根基
>
> 共享的智能指针类型，包括`Rc<T>`和`Arc<T>`，提供了可以在多方之间克隆和共享的容器
> 。因为所包含的值可能是多重别名的，所以它们只能作为共享引用被借用，而不是可变型引用
> 。如果没有单元格，就根本不可能对共享盒内的数据进行变异！
>
> 因此，在共享指针类型内放置一个`RefCell<T>`以重新引入可变性是非常常见的：
>
> ```rust ,ignore
> use std::collections::HashMap;
> use std::cell::RefCell;
> use std::rc::Rc;
>
> fn main() {
>     let shared_map: Rc<RefCell<_>> = Rc::new(RefCell::new(HashMap::new()));
>     shared_map.borrow_mut().insert("africa", 92388);
>     shared_map.borrow_mut().insert("kyoto", 11837);
>     shared_map.borrow_mut().insert("piccadilly", 11826);
>     shared_map.borrow_mut().insert("marbles", 38);
> }
> ```
>
> 注意这个例子使用了`Rc<T>`而不是`Arc<T>`。`RefCell<T>`是用于单线程的情况。如果你在
> 多线程情况下需要共享可变性，请考虑使用`Mutex<T>`。

嘿，Rust的文档仍然是令人难以置信的棒。

我们关心的主要内容是这一行：

```rust ,ignore
shared_map.borrow_mut().insert("africa", 92388);
```

特别是`borrow_mut`的事情。似乎我们需要明确地借用一个RefCell。`.`运算符不会为我
们这样做。很奇怪。让我们试试：

```rust ,ignore
pub fn push_front(&mut self, elem: T) {
    let new_head = Node::new(elem);
    match self.head.take() {
        Some(old_head) => {
            old_head.borrow_mut().prev = Some(new_head.clone());
            new_head.borrow_mut().next = Some(old_head);
            self.head = Some(new_head);
        }
        None => {
            self.tail = Some(new_head.clone());
            self.head = Some(new_head);
        }
    }
}
```


```text
> cargo build

warning: field is never used: `elem`
  --> src/fourth.rs:12:5
   |
12 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default
```

嘿，它构建了！文档又赢了。
