# 选择

好了，我们用`push`和`pop`做。不瞒你说，我在那里有点情绪化了。编译时的正确性是
一种可怕的毒药。

让我们做一些简单的事情来冷静一下：让我们实现`peek_front`。这在以前是非常容易
的。应该还是很容易的，对吗？

对吗？

事实上，我想我可以直接复制粘贴它！"。

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

等等。不是这时候。

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        // BORROW!!!!
        &node.borrow().elem
    })
}
```

哈。

```text
cargo build

error[E0515]: cannot return value referencing temporary value
  --> src/fourth.rs:66:13
   |
66 |             &node.borrow().elem
   |             ^   ----------^^^^^
   |             |   |
   |             |   temporary value created here
   |             |
   |             returns a value referencing data owned by the current function
```

好吧，我只是在烧我的电脑。

这与我们的单链栈的逻辑完全相同。为什么事情会不同。为什么。

答案其实就是本章的全部寓意。RefCells使一切变得悲伤。到现在为止，RefCells只是一个讨
厌的东西。现在它们将成为一场噩梦。

那么发生了什么？为了理解这一点，我们需要回到`borrow`的定义上：

```rust ,ignore
fn borrow<'a>(&'a self) -> Ref<'a, T>
fn borrow_mut<'a>(&'a self) -> RefMut<'a, T>
```

在布局部分，我们说：

> RefCell不是静态地执行这些规则，而是在运行时执行它们。
> 如果你违反了这些规则，RefCell就会惊慌失措，使程序崩溃。
> 为什么它会返回这些Ref和RefMut的东西？好吧，它们的行为基本上和`Rc`s一样，只是用于借用。
> 另外，它们保持RefCell的借用，直到它们超出范围。**我们稍后会讨论这个问题。**

现在就是那个稍后。

`Ref`和`RefMut`分别实现了`Deref`和`DerefMut`。因此，就大多数意图和目的而言，它们
的行为与`&T`和`&mut T`*完全*一样。然而，由于这些特性的工作方式，被返回的引用与引用
的生命周期有关，而不是与实际的RefCell有关。这意味着，只要我们保留引用，Ref就必须一直
存在。

事实上，这对于正确性是必要的。当一个Ref被放弃时，它告诉RefCell它不再被借用。因此，如
果我们*确实*设法保持我们的引用比引用存在的时间长，我们就可以在引用四处游荡时得到一个
RefMut，并完全破坏Rust的类型系统的一半。

那么，这让我们怎么办呢？我们只想返回一个引用，但我们需要保持这个引用的存在。但是一旦
我们从`peek`中返回引用，函数就结束了，而`Ref`就出了范围。

😖

据我所知，我们在这里实际上已经完全陷入了困境。你不能像这样把RefCells的使用完全封装起来。

但是......如果我们放弃了完全隐藏我们的实现细节呢？如果我们返回Refs呢？

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        node.borrow()
    })
}
```

```text
> cargo build

error[E0412]: cannot find type `Ref` in this scope
  --> src/fourth.rs:63:40
   |
63 |     pub fn peek_front(&self) -> Option<Ref<T>> {
   |                                        ^^^ not found in this scope
help: possible candidates are found in other modules, you can import them into scope
   |
1  | use core::cell::Ref;
   |
1  | use std::cell::Ref;
   |
```

啊这。要导入一些东西。


```rust ,ignore
use std::cell::{Ref, RefCell};
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/fourth.rs:64:9
   |
64 | /         self.head.as_ref().map(|node| {
65 | |             node.borrow()
66 | |         })
   | |__________^ expected type parameter, found struct `fourth::Node`
   |
   = note: expected type `std::option::Option<std::cell::Ref<'_, T>>`
              found type `std::option::Option<std::cell::Ref<'_, fourth::Node<T>>>`
```

嗯...这就对了。我们有一个`Ref<Node<T>>`，但是我们想要一个`Ref<T>`。我们可以放弃所
有封装的希望，直接返回它。我们也可以把事情搞得更复杂，把`Ref<Node<T>>`包装成一个新
的类型，只暴露对`&T`的访问。

这两种方法都*有点*蹩脚。

相反，我们要更深入地去做。让我们找点*乐子*。我们的乐趣之源是这只*野兽*：

```rust ,ignore
map<U, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
    where F: FnOnce(&T) -> &U,
          U: ?Sized
```

> 为借来的数据的一个组成部分做一个新的参考。

是的：就像你可以映射一个Option，你也可以映射一个Ref。

我相信某些人真的很兴奋，因为*单体*或其他什么，但我并不关心这些。我也不认为这是一个合
适的单体，因为没有None-like的情况，但我离题了。

它很酷，这对我来说才是最重要的。*我需要这个*。

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        Ref::map(node.borrow(), |node| &node.elem)
    })
}
```

```text
> cargo build
```

啊耶。

让我们通过从我们的堆栈中插入测试来确保它的工作。我们需要做一些处理，以应对Refs不实
现比较的事实。

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert!(list.peek_front().is_none());
    list.push_front(1); list.push_front(2); list.push_front(3);

    assert_eq!(&*list.peek_front().unwrap(), &3);
}
```


```
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 10 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test fourth::test::peek ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test third::test::basics ... ok
test second::test::peek ... ok
test second::test::iter ... ok
test third::test::iter ... ok

test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured

```

棒！
