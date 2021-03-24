# 可变遍历

老实说，可变遍历是很疯狂的。这本身就像是一个疯狂的说法；它肯定与Iter相同！

从语义上讲，是的，但是当IterMut是合法的巫师魔法时，共享和可变引用的本质意
味着Iter是"微不足道的"。

关键的洞察力来自于我们对Iter的Iterator的实现：

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> { /* stuff */ }
}
```

这可以被解开语法糖为：

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next<'b>(&'b mut self) -> Option<&'a T> { /* stuff */ }
}
```

`next`的签名在输入和输出的生命期之间*没有*建立任何约束! 我们为什么要关心这个？
这意味着我们可以无条件地一次又一次地调用`next`!


```rust ,ignore
let mut list = List::new();
list.push(1); list.push(2); list.push(3);

let mut iter = list.iter();
let x = iter.next().unwrap();
let y = iter.next().unwrap();
let z = iter.next().unwrap();
```

酷！

这对共享引用来说*绝对是好的*，因为整个要点是你可以同时拥有大量的引用。然而，可变的
引用*不能*共存。重点是它们是排他性的。

最终的结果是，使用安全的代码来编写IterMut是非常困难的（我们还没有深入了解这意味着
什么......）。令人惊讶的是，IterMut实际上可以完全安全地实现许多结构!

我们将从Iter的代码开始，把所有的东西都改成可变的：

```rust ,ignore
pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn iter_mut(&self) -> IterMut<'_, T> {
        IterMut { next: self.head.as_deref_mut() }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref_mut();
            &mut node.elem
        })
    }
}
```

```text
> cargo build
error[E0596]: cannot borrow `self.head` as mutable, as it is behind a `&` reference
  --> src/second.rs:95:25
   |
94 |     pub fn iter_mut(&self) -> IterMut<'_, T> {
   |                     ----- help: consider changing this to be a mutable reference: `&mut self`
95 |         IterMut { next: self.head.as_deref_mut() }
   |                         ^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable

error[E0507]: cannot move out of borrowed content
   --> src/second.rs:103:9
    |
103 |         self.next.map(|node| {
    |         ^^^^^^^^^ cannot move out of borrowed content
```

好吧，看起来我们在这里有两个不同的错误。第一个错误看起来非常清楚，它甚至告诉我们如何解决它！你
不能把一个共享的引用升级为一个可变的引用，所以`iter_mut`需要用`&mut self`。只是一个愚蠢的复制
粘贴错误。

```rust ,ignore
pub fn iter_mut(&mut self) -> IterMut<'_, T> {
    IterMut { next: self.head.as_deref_mut() }
}
```

那另一个呢？

哎呀! 其实我在写上一节的`iter` impl时不小心犯了一个错误，我们只是在侥幸地认为它成
功了!

我们刚刚第一次接触到了Copy的魔力。当我们介绍[所有权][ownership]时，我们说当你移动
东西时，你就不能再使用它了。对于某些类型，这是很有意义的。我们的好朋友Box为我们管理
着堆上的分配，我们当然不希望有两段代码认为它们需要释放它的内存。

然而，对于其他类型来说，这就是*垃圾*了。整数没有所有权语义；它们只是无意义的数字！
这就是为什么整数被标记为Copy。众所周知，Copy类型是完全可以通过比特拷贝的方式来复制
的。因此，它们有一个超级能力：当被移动时，旧的值仍然*可以*使用。因此，你甚至可以将
一个Copy类型从一个引用中移出而不需要替换

rust中所有的数字基元（i32, u64, bool, f32, char, etc...）都是Copy。你也可以声明任
何用户定义的类型是Copy，只要它的所有组件都是Copy。

最关键的是，共享引用也是Copy! 因为`&`是Copy，`Option<&>`*也*是Copy。所以当我们做
`self.next.map`的时候就很好，因为Option只是被复制了。现在我们不能这样做了，因为
`&mut`不是Copy（如果你复制了一个&mut，你就会有两个&mut到内存的同一个位置，这是禁止
的）。相反，我们应该正确地`take` Option来获得它。


```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.next.take().map(|node| {
        self.next = node.next.as_deref_mut();
        &mut node.elem
    })
}
```

```text
> cargo build

```

呃......哇。我的妈呀 IterMut跑起来了!

让我们测试一下这个：


```rust ,ignore
#[test]
fn iter_mut() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter_mut();
    assert_eq!(iter.next(), Some(&mut 3));
    assert_eq!(iter.next(), Some(&mut 2));
    assert_eq!(iter.next(), Some(&mut 1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 6 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured

```

是的。它是有效的。

真他妈的。

什么。

好吧，我的意思是它实际上*是*应该工作的，但通常会有一些愚蠢的东西挡住了它的去路! 让
我们在这里说清楚：

我们刚刚实现了一段代码，它接收了一个单链表，并且返回链表中每个元素的可变引用各最多一
次。而且它是经过静态验证的，可以做到这一点。它是完全安全的。我们不需要做任何疯狂的事
情。

如果你问我，这是件大事。有几个原因可以说明这一点：

* 我们`take` `Option<&mut>`，所以我们可以独占地访问可变的引用。不需要担心有人会再看
  它。
* Rust知道把一个可变引用分散到指向的结构的子字段中是可以的，因为没有办法"回到上面"，
  而且它们肯定是不相干的。

事实证明，你也可以应用这个基本逻辑来为数组或树获得一个安全的IterMut！你甚至可以把迭代
器变成一个安全的迭代器。你甚至可以让迭代器变成DoubleEnded，这样你就可以同时从前面*和*
后面消耗迭代器了。呜呼!

[ownership]: first-ownership.md
