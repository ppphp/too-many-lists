# 崩溃

`pop_front`应该是与`push_front`相同的基本逻辑，但要倒过来。我们来试试：

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    // need to take the old head, ensuring it's -2
    self.head.take().map(|old_head| {                         // -1 old
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {                               // -1 new
                // not emptying list
                new_head.borrow_mut().prev.take();            // -1 old
                self.head = Some(new_head);                   // +1 new
                // total: -2 old, +0 new
            }
            None => {
                // emptying list
                self.tail.take();                             // -1 old
                // total: -2 old, (no new)
            }
        }
        old_head.elem
    })
}
```

```text
> cargo build

error[E0609]: no field `elem` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:64:22
   |
64 |             old_head.elem
   |                      ^^^^ unknown field
```

啊。*RefCells*. 我想，又得`borrow_mut`了...

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    self.head.take().map(|old_head| {
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {
                new_head.borrow_mut().prev.take();
                self.head = Some(new_head);
            }
            None => {
                self.tail.take();
            }
        }
        old_head.borrow_mut().elem
    })
}
```

```text
cargo build

error[E0507]: cannot move out of borrowed content
  --> src/fourth.rs:64:13
   |
64 |             old_head.borrow_mut().elem
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^ cannot move out of borrowed content
```

*叹息*

> 不能移出借用的内容

嗯... 看来盒子真的是宠坏了我们。`borrow_mut`只能得到一个`&mut Node<T>`，但我
们不能从那里移出！

我们需要一个能接收`RefCell<T>`并给我们一个`T`的东西，让我们看看[文档][refcell]
中是否有这样的东西：

> `fn into_inner(self) -> T`
>
> 消费RefCell，返回被包装的值。

这看起来很有希望！

```rust ,ignore
old_head.into_inner().elem
```

```text
> cargo build

error[E0507]: cannot move out of an `Rc`
  --> src/fourth.rs:64:13
   |
64 |             old_head.into_inner().elem
   |             ^^^^^^^^ cannot move out of an `Rc`
```

啊偶。`into_inner`想要移出RefCell，但是我们不能，因为它在一个`Rc`中。正如我们在上一章看到的，
`Rc<T>`只允许我们把共享的引用移到它的内部。这是有道理的，因为这就是引用计数指针的*全部意义*：
它们是共享的！

当我们想为我们的引用计数列表实现Drop时，这是一个问题，解决方案也是一样的：`Rc::try_unwrap`，
如果一个Rc的refcount是1，它将移出其内容。

```rust ,ignore
Rc::try_unwrap(old_head).unwrap().into_inner().elem
```

`Rc::try_unwrap`返回一个`Result<T, Rc<T>>`。结果基本上是一个广义的`Option`，
其中`None`的情况下有数据与之相关。在这种情况下，就是你试图解包的`Rc`。由于我们并
不关心它失败的情况（如果我们的程序写得正确，它一定会成功），我们只是对它调用
`unwrap`。

总之，让我们看看接下来会出现什么编译器错误（让我们面对现实吧，一定会有的）。

```text
> cargo build

error[E0599]: no method named `unwrap` found for type `std::result::Result<std::cell::RefCell<fourth::Node<T>>, std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>>` in the current scope
  --> src/fourth.rs:64:38
   |
64 |             Rc::try_unwrap(old_head).unwrap().into_inner().elem
   |                                      ^^^^^^
   |
   = note: the method `unwrap` exists but the following trait bounds were not satisfied:
           `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>> : std::fmt::Debug`
```

UGH.Result上的`unwrap`要求你能调试打印出错的情况。`RefCell<T>`只有在`T`实现时才实现
`Debug`。`Node`并没有实现Debug。

与其这样做，不如通过将Result转换为一个带有`ok`的Option来解决这个问题：

```rust ,ignore
Rc::try_unwrap(old_head).ok().unwrap().into_inner().elem
```

请。

```text
cargo build

```

对了。

*呼*

我们做到了。

我们应用了`push`和`pop`。

让我们通过窃取旧的`stack`基本测试来进行测试（因为这就是我们到目前为止所实现的全
部）：

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop_front(), None);

        // Populate list
        list.push_front(1);
        list.push_front(2);
        list.push_front(3);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(3));
        assert_eq!(list.pop_front(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push_front(4);
        list.push_front(5);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(5));
        assert_eq!(list.pop_front(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop_front(), Some(1));
        assert_eq!(list.pop_front(), None);
    }
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 9 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::basics ... ok
test fifth::test::iter_mut ... ok
test third::test::basics ... ok
test second::test::iter ... ok
test third::test::iter ... ok
test second::test::into_iter ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured

```

*钉住了*。

现在我们可以正确地从列表中删除东西，我们可以实现Drop。这次的Drop在概念上更加有趣。
以前我们费力地为我们的堆栈实现Drop只是为了避免无界递归，现在我们需要实现Drop才能
让*任何事情*发生。

`Rc`不能处理循环。如果有一个循环，所有的东西都会让其他的东西活着。事实证明，一个
双向链接的列表只是一个由微小的循环组成的大链子！因此，当我们放弃我们的列表时，两
个末端节点的refcounts将被递减到1......然后就不会再发生什么了。好吧，如果我们的列
表正好包含一个节点，我们就可以了。但理想情况下，如果一个列表包含多个元素，它就应
该正常工作。也许这只是我的问题。

正如我们所看到的，删除元素是有点痛苦的。所以对我们来说，最简单的事情就是`pop`，直
到我们得到None：

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while self.pop_front().is_some() {}
    }
}
```

```text
cargo build

```

(我们实际上可以用我们的可变堆栈做到这一点，但捷径是给那些理解事物的人的！)

我们可以考虑实现`push`和`pop`的`_back`版本，但这只是复制粘贴的工作，我们将在本章的
后面进行讨论。现在，让我们看看更有趣的事情吧！


[refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[multirust]: https://github.com/brson/multirust
[downloads]: https://www.rust-lang.org/install.html
