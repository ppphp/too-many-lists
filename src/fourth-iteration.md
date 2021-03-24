# 迭代

让我们来试着迭代这个坏小子。

## IntoIter

IntoIter和往常一样，是最简单的。只是包装了栈并调用`pop`：

```rust ,ignore
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<T> {
        self.0.pop_front()
    }
}
```

但我们有一个有趣的新发展。以前，我们的列表只有一种"自然"的迭代顺序，而双向队列本
身就是双向的。从前到后有什么特别之处？如果有人想从另一个方向迭代呢？

Rust实际上对此有一个答案：`DoubleEndedIterator`。DoubleEndedIterator*继承*自
Iterator（意味着所有的DoubleEndedIterator都是Iterators），并且需要一个新方法：
`next_back`。它的签名与`next`完全相同，但它应该从另一端产生元素。
DoubleEndedIterator的语义对我们来说是非常方便的：迭代器成为一个双向队列。你可以
从前面和后面消耗元素，直到两端汇合，这时迭代器是空的。

与Iterator和`next`一样，事实证明`next_back`并不是DoubleEndedIterator的消费者
真正关心的东西。相反，这个接口最好的部分是它暴露了`rev`方法，它将迭代器包装起来，
形成一个新的迭代器，以相反的顺序产生元素。这方面的语义是相当直接的：对反向迭代器
的`next`的调用只是对`next_back`的调用。

总之，因为我们已经是一个双向队列，所以提供这个API是非常容易的：

```rust ,ignore
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        self.0.pop_back()
    }
}
```

然后让我们测试它：

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push_front(1); list.push_front(2); list.push_front(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next_back(), Some(1));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next_back(), None);
    assert_eq!(iter.next(), None);
}
```


```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 11 tests
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test fourth::test::into_iter ... ok
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test third::test::iter ... ok
test third::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 11 passed; 0 failed; 0 ignored; 0 measured

```

好。

## Iter

Iter的宽容度会低一些。我们将不得不再次处理那些可怕的`Ref`的东西 由于Refs的存在，
我们不能像以前那样存储`&Nodes`。相反，让我们尝试存储`Ref<Node>`s：

```rust ,ignore
pub struct Iter<'a, T>(Option<Ref<'a, Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.borrow()))
    }
}
```

```text
> cargo build

```

到目前为止还不错。`next`的实现会有点棘手，但我认为它与旧的堆栈IterMut的基本逻辑
是一样的，只是多了RefCell的疯狂：

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = Ref<'a, T>;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node_ref| {
            self.0 = node_ref.next.as_ref().map(|head| head.borrow());
            Ref::map(node_ref, |node| &node.elem)
        })
    }
}
```

```text
cargo build

error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:155:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   -------- borrow is only valid in the closure body
    |             |
    |             reference to `node_ref` escapes the closure body here

error[E0505]: cannot move out of `node_ref` because it is borrowed
   --> src/fourth.rs:156:22
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- lifetime `'1` appears in the type of `self`
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ------   -------- borrow of `node_ref` occurs here
    |             |
    |             assignment requires that `node_ref` is borrowed for `'1`
156 |             Ref::map(node_ref, |node| &node.elem)
    |                      ^^^^^^^^ move out of `node_ref` occurs here
```

完成。

`node_ref`的生命周期不够长。与普通的引用不同，Rust不允许我们像这样把Ref拆开。我们从
`head.borrow()`中得到的Ref只允许活得和`node_ref`一样长，但我们最终在`Ref::map`
的调用中破坏了它。

巧合的是，就在我写这篇文章的时候，我们想要的函数实际上在两天前就已经稳定了。这意味着它
将在几个月后进入稳定版。所以让我们继续使用最新的每夜构建：

```rust ,ignore
pub fn map_split<U, V, F>(orig: Ref<'b, T>, f: F) -> (Ref<'b, U>, Ref<'b, V>) where
    F: FnOnce(&T) -> (&U, &V),
    U: ?Sized,
    V: ?Sized,
```

芜湖。让我们试一试...

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.0.take().map(|node_ref| {
        let (next, elem) = Ref::map_split(node_ref, |node| {
            (&node.next, &node.elem)
        });

        self.0 = next.as_ref().map(|head| head.borrow());

        elem
    })
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:159:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
...
159 |             self.0 = next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   ---- borrow is only valid in the closure body
    |             |
    |             reference to `next` escapes the closure body here
```

Ergh. 我们需要再次使用`Ref::Map`来使我们的生命周期正确。但是`Ref::Map`返回一个
`Ref`，我们需要一个`Option<Ref>`，但是我们需要通过Ref来映射我们的Option......

**盯着远方看了很久**

??????

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.0.take().map(|node_ref| {
        let (next, elem) = Ref::map_split(node_ref, |node| {
            (&node.next, &node.elem)
        });

        self.0 = if next.is_some() {
            Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
        } else {
            None
        };

        elem
    })
}
```

```text
error[E0308]: mismatched types
   --> src/fourth.rs:162:22
    |
162 |                 Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
    |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `fourth::Node`, found struct `std::cell::RefCell`
    |
    = note: expected type `std::cell::Ref<'_, fourth::Node<_>>`
               found type `std::cell::Ref<'_, std::cell::RefCell<fourth::Node<_>>>`
```

哦。对。这里有多个RefCell。我们在列表中走得越深，每个RefCell下的嵌套就越多。我们需要
维护，比如说，一个堆栈的Refs来代表我们持有的所有未偿借贷，因为如果我们停止查看一个元
素，我们需要递减它之前的每个RefCell的借用计数.................

我不认为我们在这里有什么可以做的。这是个死胡同。让我们试着离开RefCell。

那我们的`Rc`s呢？谁说我们甚至需要存储引用？为什么我们不能克隆整个Rc，以获得一个漂亮的
自己的解决方案到列表的中间？

```rust
pub struct Iter<T>(Option<Rc<Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.clone()))
    }
}

impl<T> Iterator for Iter<T> {
    type Item =
```
呃......等等，我们现在返回什么？ `&T`？`Ref<T>`？

不，这些都不行......我们的Iter已经没有生命周期了！`&T`和`Ref<T>`都要求我们在进
入下一步之前先声明一些生命周期。但是我们设法从Rc中得到的任何东西都会借用迭代
器......脑子......受伤了......啊啊啊啊啊啊

也许我们可以......映射......Rc......得到一个`Rc<T>`？这是个问题吗？Rc的文档似乎
没有这样的东西。事实上，有人做了一个[板条箱][own-ref]，可以让你这样做。

但是等等，即使我们*这样*做了，我们也有一个更大的问题：如幽灵般可怕的迭代器失效。
以前我们对迭代器失效完全免疫，因为Iter借用了列表，使其完全不可改变。然而，如果我
们的 Iter 产生了 Rcs，他们就根本不会借用列表了！这意味着人们可以在持有指向列表的
指针时开始调用`push`和`pop`。

哦，天哪，那会怎么样？

好吧，推入其实是可以的。我们已经得到了一个进入列表的某个子范围的视图，而列表只是在
我们的视线之外增长。没什么大不了的。

然而`pop`是另一个故事。如果他们在我们的范围之外弹出元素，应该*还*是可以的。我们看
不到那些节点，所以不会发生什么。然而，如果他们试图从我们指向的节点上弹出......一切
都会被炸毁！特别是当他们去`unwrap` `try_unwrap`的结果时，实际上会失败，整个程序
会恐慌。

这实际上是非常酷的。我们可以把大量的内部拥有的指针放入列表中，并同时对其进行改变，
它就*会一直工作*，直到他们试图删除我们所指向的节点。即使如此，我们也不会出现悬空指
针或任何东西，程序会确定地发生恐慌！

但是在映射Rc的基础上还要处理迭代器失效的问题，这似乎......很糟糕。`Rc<RefCell>`
真的真的终于让我们失望了。有趣的是，我们经历了持久化堆栈案例的反转。持久堆栈努力地
收回数据的所有权，但每天都能得到引用，而我们的列表在获得所有权方面没有问题，但在借
出引用方面却非常困难。

尽管公平地说，我们的大部分挣扎都是围绕着想要隐藏实现细节和拥有一个体面的API。如果我
们想在所有地方传递节点，我们*可以*做得很好。

见鬼，我们可以制作多个并发的IterMuts，这些IterMuts在运行时被检查为不可改变地访问同
一个元素！这就是我们的设计。

实际上，这种设计更适合于内部数据结构，因为它永远不会让API的消费者看到。内部可变性对
于编写安全的*应用程序*是很好的。但对于安全的*库*来说就不是那么回事了。

总之，我放弃了Iter和IterMut。我们可以做它们，但是*唉*。

[own-ref]: https://crates.io/crates/owning_ref
