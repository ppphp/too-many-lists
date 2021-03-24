# 使用Option

细心的读者可能已经注意到，我们实际上重新发明了一个非常糟糕的Option版本：

```rust ,ignore
enum Link {
    Empty,
    More(Box<Node>),
}
```

Link只是`Option<Box<Node>>`。现在，可以不用到处写`Option<Box<Node>>`了，
真不错。而且与`pop`不同的是，我们并没有把它暴露给外部世界，所以也许这很好。
然而Option有一些*非常好*的方法，我们一直在手动实现。我们*不要*这样做，而是用
Options代替一切。首先，我们简单地使用Some和None重命名所有的东西：

```rust ,ignore
use std::mem;

pub struct List {
    head: Link,
}

// 耶，类型别名！
type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, None),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, None) {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, None);
        while let Some(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, None);
        }
    }
}
```

这稍微好一点，但最大的胜利来自于Option的方法。

首先，`mem::replace(&mut option, None)`是一个非常常见的习语，Option实际上只是把它
作为一个方法：`take`。

```rust ,ignore
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match self.head.take() {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

第二，`match option { None => None, Some(x) => Some(y) }`是一个非常惯用的写法，
以至于它被称为`map`。`map`需要一个函数来对`Some(x)`中的`x`执行，以产生`Some(y)`
中的`y`。我们可以写一个适当的`fn`并把它传递给`map`，但我们更愿意写出*内联*要做什
么。

做到这一点的方法是使用一个*闭包*。闭包是匿名函数，有一个额外的超级能力：它们可以
引用闭包*之外*的局部变量。这使得它们在做各种条件逻辑时超级有用。我们唯一进行
`match`的地方是在`pop`中，所以让我们重写一下：

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    self.head.take().map(|node| {
        self.head = node.next;
        node.elem
    })
}
```

啊，好多了。让我们确保我们没有破坏任何东西：

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

很好! 让我们继续实际改进代码的*行为*。
