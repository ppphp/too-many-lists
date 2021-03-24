# 基本

我们现在已经知道了很多Rust的基础知识，所以我们可以再次做很多简单的事情。

对于构造函数，我们又可以直接复制粘贴：

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }
}
```

`push`和`pop`已经不太能表达含义了。我们可以提供`append`和`tail`来代替它们，它们提
供的东西大致相同。

让我们从append开始。它接受一个列表和一个元素，并返回一个列表。就像可变列表的情况一样，
我们要做一个新的节点，它的`next`值是旧列表。唯一新奇的是如何获得下一个值，因为我们不
允许改变任何东西。

我们祈祷的答案就是Clone特性。几乎所有的类型都实现了克隆，它提供了一种通用的方法来获得
"像这样的另一个"，而这个逻辑上是不相干的，只给了一个共享的引用。它就像C++中的复制构造
函数，但它不会被隐式调用。

特别地，Rc使用Clone作为增加引用计数的方法。因此，与其移动一个Box到子列表中，不如直接
克隆旧列表的头部。我们甚至不需要在头部进行匹配，因为Option暴露了一个Clone的实现，它所
做的正是我们想要的事情。

好了，让我们试一试吧：

```rust ,ignore
pub fn append(&self, elem: T) -> List<T> {
    List { head: Some(Rc::new(Node {
        elem: elem,
        next: self.head.clone(),
    }))}
}
```

```text
> cargo build

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

哇，Rust对实际使用字段真的很苛刻。它可以告诉没有消费者可以真正观察到这些字段的使用情
况! 不过，到目前为止，我们似乎还不错。

`tail`是这个操作的逻辑逆运算。它接收一个列表并返回除去第一个元素的整个列表。所有这些
都是克隆列表中的*第二个*元素（如果它存在的话）。我们来试试这个：

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().map(|node| node.next.clone()) }
}
```

```text
cargo build

error[E0308]: mismatched types
  --> src/third.rs:27:22
   |
27 |         List { head: self.head.as_ref().map(|node| node.next.clone()) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `std::rc::Rc`, found enum `std::option::Option`
   |
   = note: expected type `std::option::Option<std::rc::Rc<_>>`
              found type `std::option::Option<std::option::Option<std::rc::Rc<_>>>`
```

嗯，我们搞砸了。`map`希望我们返回一个Y，但是在这里我们要返回一个`Option<Y>`。值得庆
幸的是，这是另一个常见的Option模式，我们可以直接使用`and_then`来让我们返回一个Option。

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().and_then(|node| node.next.clone()) }
}
```

```text
> cargo build

```

很好。

现在我们有了`tail`，我们也许应该提供`head`，它返回对第一个元素的引用。这只是从可
变列表中`peek`：

```rust ,ignore
pub fn head(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem )
}
```

```text
> cargo build

```

很好。

这已经有足够的功能，我们可以测试它：


```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let list = List::new();
        assert_eq!(list.head(), None);

        let list = list.append(1).append(2).append(3);
        assert_eq!(list.head(), Some(&3));

        let list = list.tail();
        assert_eq!(list.head(), Some(&2));

        let list = list.tail();
        assert_eq!(list.head(), Some(&1));

        let list = list.tail();
        assert_eq!(list.head(), None);

        // Make sure empty tail works
        let list = list.tail();
        assert_eq!(list.head(), None);

    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured

```

很完美!

Iter也与我们的可变链表的情况相同：

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.elem
        })
    }
}
```

```rust ,ignore
#[test]
fn iter() {
    let list = List::new().append(1).append(2).append(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 7 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured

```

谁曾说过动态类型更容易？

(笨蛋说的)

注意，我们不能为这个类型实现IntoIter或IterMut。我们只有对元素的共享访问。
