# 让他全变成泛型

我们已经用 Option 和 Box 触及了一些泛型。然而到目前为止，我们一直设法避免声明任
何新的类型，这些类型实际上是对任意元素的泛型。

事实证明，这其实很容易。让我们现在就把我们所有的类型都变成泛型：

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

你只要把所有的东西都加上两个尖角，你的代码就会突然间变得很通用。当然，我们不能
*仅仅*这样做，否则编译器就会变得超级疯狂。


```text
> cargo test

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:14:6
   |
14 | impl List {
   |      ^^^^ expected 1 type argument

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:36:15
   |
36 | impl Drop for List {
   |               ^^^^ expected 1 type argument

```

问题很明显：我们一直在谈论这个`List`的事情，但那已经不是真的了。像 Option 和 Box 
一样，我们现在必须要谈论 `List<Something>`。

但是我们在所有这些impl中使用的Something是什么？就像List一样，我们希望我们的实现能与
*所有*的T一起工作。所以，就像List一样，让我们的`impl`变得尖尖的：


```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            node.elem
        })
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

......就这样了!

```
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

我们所有的代码现在在任意的T值上都是完全通用的，铛，Rust很*容易*。我想对甚至没有改
变的`new`做一个特别的叫喊：

```rust ,ignore
pub fn new() -> Self {
    List { head: None }
}
```

沐浴在Self的光辉中，它是重构和复制粘贴编码的守护者。同样有趣的是，当我们构造一个
链表的实例时，我们并没有写 `List<T>`。这一部分是根据我们从一个期望有 `List<T>`
的函数中返回的事实来推断的。

好了，让我们继续讨论全新的*行为*吧！
