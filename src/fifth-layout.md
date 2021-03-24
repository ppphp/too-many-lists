# 布局

那么，单链队列是什么样的呢？好吧，当我们有一个单链接的堆栈时，我们推到列表的一端，
然后从同一端弹出。栈和队列的唯一区别是，队列会从*另*一端弹出。所以从我们的堆栈实现
来看，我们有：

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

stack push X:
[Some(ptr)] -> (X, Some(ptr)) -> (A, Some(ptr)) -> (B, None)

stack pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

要制作一个队列，我们只需要决定将哪个操作移到列表的末尾：推，还是弹？由于我们的列表
是单链的，我们实际上可以用同样的努力把*任何一个*操作移到最后。

要把`push`移到最后，我们只需一路走到`None`，并把它和新元素一起设置为Some。

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

flipped push X:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)
```

要把`pop`移到最后，我们只需一路走到None*之前*的节点，然后`take`它：

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)

flipped pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

我们今天可以做这个，然后就不干了，但那会很臭！。这两种操作都是在*整个*列表上行走。有
些人认为，这样的队列实现确实是一个队列，因为它暴露了正确的接口。然而我认为性能保证是
接口的一部分。我不关心精确的渐近界线，只关心“快”与“慢”。队列保证推送和弹出是快速的，
而在整个列表上行走肯定是*不*快的。

一个关键的观察是，我们在重复做*同样的事情*，浪费了大量的工作。我们能不能把这项工作记
忆化？为什么，是的？我们可以存储一个指向列表末尾的指针，然后直接跳到那里去！

事实证明，只有一种反转的`push`和`pop`方式可以使用。要反转`pop`，我们必须将“尾部”指针
向后移动，但由于我们的列表是单链的，我们无法有效地做到这一点。如果我们反转`push`，我
们只需要将“头部”指针向前移动，这很容易。

让我们试试吧：

```rust ,ignore
use std::mem;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // swap the old tail to point to the new tail
        let old_tail = mem::replace(&mut self.tail, Some(new_tail));

        match old_tail {
            Some(mut old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
            }
        }
    }
}
```

由于我们对这种事情应该很熟悉，所以我现在在实现细节方面走得快一点。并不是说你应该期望
在第一次尝试时就能产生这样的代码。我只是跳过了一些我们以前不得不处理的试验和错误。实
际上，我在写这段代码时犯了很多错误，我没有显示出来。你只能看到我漏掉了`mut`或`;`这么
多次，然后它就不再具有指导意义了。别担心，我们会看到很多*其他的*错误信息的！

```text
> cargo build

error[E0382]: use of moved value: `new_tail`
  --> src/fifth.rs:38:38
   |
26 |         let new_tail = Box::new(Node {
   |             -------- move occurs because `new_tail` has type `std::boxed::Box<fifth::Node<T>>`, which does not implement the `Copy` trait
...
33 |         let old_tail = mem::replace(&mut self.tail, Some(new_tail));
   |                                                          -------- value moved here
...
38 |                 old_tail.next = Some(new_tail);
   |                                      ^^^^^^^^ value used here after move
```

发现了！

> 使用了移动过的值： `new_tail`

盒子没有实现复制，所以我们不能直接把它分配到两个位置。更重要的是，Box*拥有*它所指向
的东西，当它被丢弃时，会尝试释放它。如果我们的`push`实现被编译了，我们就会把列表的
尾部释放出来。实际上，按照写法，我们的代码会在每次推送时释放 old_tail。呀! 🙀

好了，我们知道如何制作一个不能拥有的指针。这只是一个引用！

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
    tail: Option<&mut Node<T>>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // Put the box in the right place, and then grab a reference to its Node
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
                old_tail.next.as_deref_mut()
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
                self.head.as_deref_mut()
            }
        };

        self.tail = new_tail;
    }
}
```

这里没有什么太棘手的问题。与前面的代码的基本思路相同，只是我们使用了一些隐含返回的
好处，从我们放置实际Box的地方提取尾部引用。

```text
> cargo build

error[E0106]: missing lifetime specifier
 --> src/fifth.rs:3:18
  |
3 |     tail: Option<&mut Node<T>>, // NEW!
  |                  ^ expected lifetime parameter
```

哦，对了，我们需要在类型的生命期中给出参考。嗯......这个引用的生命期是多少？嗯，
这好像是IterMut，对吗？让我们试试我们为IterMut所做的，只是添加一个通用的`'a`：

```rust ,ignore
pub struct List<'a, T> {
    head: Link<T>,
    tail: Option<&'a mut Node<T>>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<'a, T> List<'a, T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // Put the box in the right place, and then grab a reference to its Node
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
                old_tail.next.as_deref_mut()
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
                self.head.as_deref_mut()
            }
        };

        self.tail = new_tail;
    }
}
```

```text
cargo build

error[E0495]: cannot infer an appropriate lifetime for autoref due to conflicting requirements
  --> src/fifth.rs:35:27
   |
35 |                 self.head.as_deref_mut()
   |                           ^^^^^^^^^^^^
   |
note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the method body at 18:5...
  --> src/fifth.rs:18:5
   |
18 | /     pub fn push(&mut self, elem: T) {
19 | |         let new_tail = Box::new(Node {
20 | |             elem: elem,
21 | |             // When you push onto the tail, your next is always None
...  |
39 | |         self.tail = new_tail;
40 | |     }
   | |_____^
note: ...so that reference does not outlive borrowed content
  --> src/fifth.rs:35:17
   |
35 |                 self.head.as_deref_mut()
   |                 ^^^^^^^^^
note: but, the lifetime must be valid for the lifetime 'a as defined on the impl at 13:6...
  --> src/fifth.rs:13:6
   |
13 | impl<'a, T> List<'a, T> {
   |      ^^
   = note: ...so that the expression is assignable:
           expected std::option::Option<&'a mut fifth::Node<T>>
              found std::option::Option<&mut fifth::Node<T>>


```

哇，这是一个非常详细的错误信息。这有点令人担忧，因为它表明我们正在做一些非常混乱的事
情。这里有一个有趣的部分：

> 生命期必须对内联中定义的 "a "的生命期有效。

我们借用了`self`，但编译器希望我们的生命期和`'a`一样长，如果我们告诉它`self`*确实*
能持续那么久呢……？

```rust ,ignore
    pub fn push(&'a mut self, elem: T) {
```

```text
cargo build

warning: field is never used: `elem`
 --> src/fifth.rs:9:5
  |
9 |     elem: T,
  |     ^^^^^^^
  |
  = note: #[warn(dead_code)] on by default
```

哦，嘿，成功了! 很好!

我们也来做`pop`吧：

```rust ,ignore
pub fn pop(&'a mut self) -> Option<T> {
    // Grab the list's current head
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        // If we're out of `head`, make sure to set the tail to `None`.
        if self.head.is_none() {
            self.tail = None;
        }

        head.elem
    })
}
```

并为此写一个快速测试：

```rust ,ignore
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);
    }
}
```

```text
cargo test

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:68:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
68 |         list.push(1);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:69:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
69 |         list.push(2);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:70:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
70 |         list.push(3);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here


....

** WAY MORE LINES OF ERRORS **

....

error: aborting due to 11 previous errors
```

🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀

哦，我的天哪。

编译器在我们身上呕吐是没有错的。我们刚刚犯了一个Rust的大罪：我们在*自己内部*存储了
一个对自己的引用。不知何故，我们成功地说服了Rust，在我们的`push`和`pop`实现中，这
完全是有意义的（我真的很震惊，我们做到了）。我相信原因是Rust还不能从`push`和`pop`
中分辨出引用是进入自己的--或者说，Rust根本就没有这个概念。引用到自己身上不能生效只
是一种突发行为。

当我们试图*使用*我们的列表时，一切都迅速崩溃了。当我们调用`push`或`pop`时，我们立
即在自己身上存储了一个对自己的引用，并被*困住*。我们实际上是在借用我们自己。

我们的`pop`实现暗示了为什么这可能是非常危险的：

```rust ,ignore
// ...
if self.head.is_none() {
    self.tail = None;
}
```

如果我们忘记这样做呢？那么我们的尾巴就会指向*某个已经从列表中删除的节点*。这样的节
点会立即被释放，我们就会有一个悬空的指针，而Rust应该保护我们免受其害！

事实上，Rust正在保护我们远离这种危险。只是以一种非常...**迂回**的方式。

那么我们能做什么呢？回到`Rc<RefCell>>`地狱？

拜托了。不，不。

不，我们要离开轨道，使用*原始指针*。我们的布局将看起来像这样：

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // DANGER DANGER
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```
就这样了。没有这种懦弱的参考-计算-动态-借贷-检查的废话! 真实。硬的。未经检查的。
指针。

让我们都成为C语言。让我们整天都是C。

我回来了。我准备好了。

你好，`unsafe`。

