# 基础

好了，回到基本的问题上。我们如何构建我们的链表？

之前我们只是做了：

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }
}
```

但我们不再为`tail`使用Option：

```text
> cargo build

error[E0308]: mismatched types
  --> src/fifth.rs:15:34
   |
15 |         List { head: None, tail: None }
   |                                  ^^^^ expected *-ptr, found enum `std::option::Option`
   |
   = note: expected type `*mut fifth::Node<T>`
              found type `std::option::Option<_>`
```

我们*可以*使用一个Option，但是与Box不同，`*mut`是可以置空的。这意味着它不能受益于
空指针的优化。相反，我们将使用`null`来表示无。

那么，我们怎样才能得到一个空指针呢？有几种方法，但我更喜欢使用
`std::ptr::null_mut()`。如果你愿意，你也可以用`0 as *mut _`，但这似乎*太乱了*。

```rust ,ignore
use std::ptr;

// defns...

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: ptr::null_mut() }
    }
}
```

```text
cargo build

warning: field is never used: `head`
 --> src/fifth.rs:4:5
  |
4 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fifth.rs:5:5
  |
5 |     tail: *mut Node<T>,
  |     ^^^^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `head`
  --> src/fifth.rs:12:5
   |
12 |     head: Link<T>,
   |     ^^^^^^^^^^^^^
```

*噓*编译器，我们很快就会用到它们。

好了，让我们继续写`push`。这一次，我们不是在插入后抓取一个`Option<&mut Node<T>>`
，而是直接抓取一个`*mut Node<T>`到Box的内部。我们知道我们可以稳妥地这样做，因为盒子
里的内容有一个稳定的地址，即使我们把盒子移来移去。当然，这并不*安全*，因为如果我们直
接丢掉盒子，我们就会有一个指向已释放内存的指针。

我们如何从一个普通的指针变成一个原始指针呢？强制的! 如果一个变量被声明为一个原始指针，
那么一个普通的引用就会被强制放到它里面：

```rust ,ignore
let raw_tail: *mut _ = &mut *new_tail;
```

我们有我们需要的所有信息。我们可以把我们的代码翻译成，大约是以前的参考版本：

```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    // .is_null checks for null, equivalent to checking for None
    if !self.tail.is_null() {
        // If the old tail existed, update it to point to the new tail
        self.tail.next = Some(new_tail);
    } else {
        // Otherwise, update the head to point to it
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

```text
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             self.tail.next = Some(new_tail);
   |             ----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; try dereferencing it: `(*self.tail).next`
```

嗯？我们有一个指向Node的指针，为什么我们不能得到`next`字段？

当你使用原始指针时，Rust是个混蛋。要访问一个原始指针的内容，它坚持要求我们手动地取消
引用它们，因为这是一个不安全的操作。所以让我们来做这个：

```rust ,ignore
*self.tail.next = Some(new_tail);
```

```text
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             *self.tail.next = Some(new_tail);
   |             -----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; try dereferencing it: `(*self.tail).next`
```

呜运算符优先级。

```rust ,ignore
(*self.tail).next = Some(new_tail);
```

```text
> cargo build

error[E0133]: dereference of raw pointer is unsafe and requires unsafe function or block
  --> src/fifth.rs:31:13
   |
31 |             (*self.tail).next = Some(new_tail);
   |             ^^^^^^^^^^^^^^^^^ dereference of raw pointer
   |
   = note: raw pointers may be NULL, dangling or unaligned; they can violate aliasing rules and cause data races: all of these are undefined behavior
```
这不应该是这么难的事。

还记得我说过不安全的Rust就像是安全Rust的FFI语言吗？那么，编译器希望我们明确地划分出
我们正在做的FFI-ing的地方。我们有两个选择。首先，我们可以把*整个*函数标记为不安全的
，在这种情况下，它就变成了一个不安全的Rust函数，只能在`unsafe`的上下文中调用。这不
是很好，因为我们希望我们的列表可以安全使用。其次，我们可以在我们的函数中写一个
`unsafe`的块，来划定FFI的边界。这就宣告了整个函数是安全的。让我们来做这件事：


```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    // Put the box in the right place, and then grab a reference to its Node
    if !self.tail.is_null() {
        // If the old tail existed, update it to point to the new tail
        unsafe {
            (*self.tail).next = Some(new_tail);
        }
    } else {
        // Otherwise, update the head to point to it
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

```text
> cargo build
warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default
```
耶！

有意思的是，这是迄今为止我们*唯一*需要写不安全块的地方。我们到处都在做原始指针的事
情，这到底是怎么回事？

事实证明，当涉及到`unsafe`的时候，Rust是一个巨大的规则法学家的迂腐者。我们有理由
希望最大化安全的Rust程序集，因为那些程序是我们可以更有信心的。为了达到这个目的，
Rust仔细地划出了一个最小的不安全区域。请注意，我们在处理原始指针的所有其他地方都是
在*分配*它们，或者只是观察它们是否为空。

如果你从来没有真正解除过一个原始指针的引用，*那么这些事情是完全安全的*。你只是在读
写一个整数而已。唯一的一次，你可能会因为一个原始指针而陷入困境，那就是你真的解除了
对它的引用。所以Rust说*只有*这种操作是不安全的，而其他的都是完全安全的。

超强。迂腐。但在技术上是正确的。

然而，这引起了一个有趣的问题：尽管我们应该用`unsafe`块来限定不安全的范围，但它实际
上依赖于在块之外建立的状态。甚至是在函数之外！

这就是我所说的不安全*污点*。只要你在一个模块中使用`unsafe`，整个模块就会被不安全所
玷污。所有的东西都必须正确编写，以确保不安全代码的不变性得到维护。

这种污点是可以管理的，因为有*隐藏性*。在我们的模块之外，我们所有的结构字段都是完全私
有的，所以其他人不能以任意的方式乱用我们的状态。只要我们公开的API的组合没有导致坏事
发生，就外部观察者而言，我们所有的代码都是安全的 实际上，这与FFI的情况没有什么不同。
只要它暴露了一个安全的接口，没有人需要关心某个Python数学库是否向C语言脱壳。

总之，让我们继续讨论`pop`，它几乎是逐字逐句的参考版本：

```rust ,ignore
pub fn pop(&mut self) -> Option<T> {
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        if self.head.is_none() {
            self.tail = ptr::null_mut();
        }

        head.elem
    })
}
```

我们再次看到另一种情况，安全是有状态的。如果我们在*这个*函数中未能将尾巴指针清空，
我们就不会看到任何问题。然而，随后对`push`的调用将开始写到悬空的尾巴上！

让我们来测试一下：

```rust ,ignore
#[cfg(test)]
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

        // Check the exhaustion case fixed the pointer right
        list.push(6);
        list.push(7);

        // Check normal removal
        assert_eq!(list.pop(), Some(6));
        assert_eq!(list.pop(), Some(7));
        assert_eq!(list.pop(), None);
    }
}
```

这只是堆栈测试，但预期的`pop`结果被翻转过来。我还在最后添加了一些额外的步骤，以确保
`pop`中的尾部指针损坏情况不会发生。

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured
```

完美！


