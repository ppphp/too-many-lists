# 遍历

好吧，让我们试着实现Iter。这一次，我们将不能依靠List提供我们想要的所有功能。我
们需要靠我们自己。我们想要的基本逻辑是持有一个指向当前节点的指针，我们想要产生
下一个节点。因为这个节点可能不存在（列表是空的或者我们已经完成了迭代），我们希
望这个引用是一个 Option。当我们产出一个元素时，我们希望继续到当前节点的下一个
节点。

好吧，我们来试试：

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:72:18
   |
72 |     next: Option<&Node<T>>,
   |                  ^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:82:17
   |
82 |     type Item = &T;
   |                 ^ expected lifetime parameter
```

哦，上帝。生命周期。我听说过这些事情。我听说它们是一场恶梦。

让我们试试新的东西：看到那个`error[E0106]`的东西吗？那是一个编译器错误代码。我们可
以让rustc用`--explain`来解释这些：

```text
> rustc --explain E0106
This error indicates that a lifetime is missing from a type. If it is an error
inside a function signature, the problem may be with failing to adhere to the
lifetime elision rules (see below).

Here are some simple examples of where you'll run into this error:

struct Foo { x: &bool }        // error
struct Foo<'a> { x: &'a bool } // correct

enum Bar { A(u8), B(&bool), }        // error
enum Bar<'a> { A(u8), B(&'a bool), } // correct

type MyStr = &str;        // error
type MyStr<'a> = &'a str; //correct
...

```

这......这并没有真正说明什么（这些文档假设我们比现在更了解Rust）。但看起来我们应该把
这些`'a`添加到我们的结构中去？让我们试试吧。

```
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:83:22
   |
83 | impl<T> Iterator for Iter<T> {
   |                      ^^^^^^^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:84:17
   |
84 |     type Item = &T;
   |                 ^ expected lifetime parameter

error: aborting due to 2 previous errors
```

好吧，我开始看到一个模式......让我们把这些小家伙添加到我们能做的所有事情中：

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<'a, T> List<T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &'a node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&'a mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &'a node);
            &'a node.elem
        })
    }
}
```

```text
> cargo build

error: expected `:`, found `node`
  --> src/second.rs:77:47
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ---- while parsing this struct        ^^^^ expected `:`

error: expected `:`, found `node`
  --> src/second.rs:85:50
   |
85 |             self.next = node.next.map(|node| &'a node);
   |                                                  ^^^^ expected `:`

error[E0063]: missing field `next` in initializer of `second::Iter<'_, _>`
  --> src/second.rs:77:9
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ^^^^ missing `next`
```

哦，上帝。我们搞坏了Rust。

也许我们应该搞清楚这个`'a`生命周期到底是什么意思。

生命期会吓跑很多人，因为它是对我们从编程之初就知道并喜爱的东西的改变。到目前为止，
我们实际上已经成功地避开了生命期，尽管它们一直在我们的程序中纠缠不清。

生命期在垃圾收集语言中是不必要的，因为垃圾收集器会确保所有的东西都能神奇地活到它需
要的时间。Rust中的大多数数据都是*人工*管理的，所以这些数据需要另一种解决方案。C和
C++给了我们一个清晰的例子，如果你只是让人们在堆栈上拿指针去随机数据会发生什么：普
遍的不可管理的不安全。这可以粗略地分为两类错误：

* 持有一个超出范围的东西的指针
* 持有一个指向被改变的东西的指针

生命期可以解决这两个问题，而且99%的情况下，它们是以一种完全透明的方式进行的。

那么什么是生命期呢？

很简单，生命期是程序中某个区域（\~块/范围）代码的名称。就是这样。当一个引用被标记为生
命期时，我们是说它必须对*整个*区域有效。不同的东西对一个引用必须和可以在多长时间内有
效提出了要求。整个生命期系统又只是一个约束解决系统，它试图最小化每个引用的区域。如果
它成功地找到了一组满足所有约束条件的生命期，你的程序就可以编译了！否则你就会得到一个
错误反馈，说什么东西的生命期不够长。

在一个函数体中，你一般不能谈论生命期，而且也不想谈论。编译器有完整的信息，可以推断出
所有的限制条件来找到最小的生命期。然而在类型和API层面，编译器*并没有*所有的信息。它需
要你告诉它不同生命期之间的关系，这样它才能弄清楚你在做什么。

原则上说，这些生命期也*可以*不写，但这样一来，检查所有的借用将是一个巨大的全程序分析
，会产生令人难以置信的非本地错误。Rust的系统意味着所有的借用检查都可以在每个函数体中
独立完成，你的所有错误都应该是相当局部的（或者你的类型有错误的签名）。

但是我们以前也在函数签名中写过引用，而且很好！为什么？这是因为有一些情况非常普遍，
Rust会自动为你挑选生命期。这就是*寿命消除*。

特别地：

```rust ,ignore
// Only one reference in input, so the output must be derived from that input
fn foo(&A) -> &B; // sugar for:
fn foo<'a>(&'a A) -> &'a B;

// Many inputs, assume they're all independent
fn foo(&A, &B, &C); // sugar for:
fn foo<'a, 'b, 'c>(&'a A, &'b B, &'c C);

// Methods, assume all output lifetimes are derived from `self`
fn foo(&self, &B, &C) -> &D; // sugar for:
fn foo<'a, 'b, 'c>(&'a self, &'b B, &'c C) -> &'a D;
```

那么，`fn foo<'a>(&'a A)->&'a B`是什么*意思*？在实践中，它的意思是，输入必须至少活得
与输出一样长。因此，如果你把输出保留很长一段时间，这将扩大输入必须有效的区域。一旦你
停止使用输出，编译器就会知道输入也可以变得无效了。

由于这个系统的设置，Rust可以确保没有东西会在释放之后被使用，而且在未完成的引用存在时
没有任何东西被改变。它只是确保这些约束条件都能得到解决。

好的。那么。Iter。

让我们回到没有生命期的状态：

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

我们只需要在函数和类型的签名中添加生命期：

```rust ,ignore
// Iter is generic over *some* lifetime, it doesn't care
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

// No lifetime here, List doesn't have any associated lifetimes
impl<T> List<T> {
    // We declare a fresh lifetime here for the *exact* borrow that
    // creates the iter. Now &self needs to be valid as long as the
    // Iter is around.
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

// We *do* have a lifetime here, because Iter has one that we need to define
impl<'a, T> Iterator for Iter<'a, T> {
    // Need it here too, this is a type declaration
    type Item = &'a T;

    // None of this needs to change, handled by the above.
    // Self continues to be incredibly hype and amazing
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

好吧，我想这次我们得到了它，你们都。

```text
cargo build

error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`
```

(╯°□°)╯︵ ┻━┻

好的。所以。我们修复了我们的生命期错误，但现在我们得到了一些新的类型错误。

我们想存储`&Node`的，但我们得到的是`&Box<Node>`的。好吧，这很容易，我们只需
要在引用之前取消对Box的引用：

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:77:43
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                                           ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                      ^^^^^^^^^ cannot move out of borrowed content

error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:85:46
   |
85 |             self.next = node.next.map(|node| &*node);
   |                                              ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &*node);
   |                         ^^^^^^^^^ cannot move out of borrowed content
```

(ﾉಥ益ಥ）ﾉ﻿ ┻━┻

我们忘记了要`as_ref`，所以我们要把盒子移到`map`里，这意味着它将被丢弃，这意味着
我们的引用将被悬空：

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_ref().map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_ref().map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.as_ref().map(|node| &*node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.as_ref().map(|node| &*node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

```

😭

`as_ref`又增加了一层我们需要去除的间接性：


```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
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

```text
cargo build

```

🎉 🎉 🎉

as_deref和as_deref_mut函数在Rust 1.40中已经稳定。在此之前，你需要做`map(|node| &**node)`
和`map(|node| &mut**node)`。你可能会想"哇，那个`&**`的东西真的很古怪"，你没有错，但就像美
酒一样，Rust会随着时间的推移变得更好，我们不再需要这样做。通常情况下，Rust非常善于隐式地进
行这种转换，通过一个叫做*解引用强制转换*的过程，基本上它可以在你的代码中插入\*，使其进行类
型检查。它可以做到这一点，因为我们有借用检查器来确保我们不会弄乱指针！

但是闭包同时结合了我们有一个`Option<&T>`而不是`&T`的情况，对它来说有点太复杂了，所以我们需
要通过显式的方式来帮助它。值得庆幸的是，根据我的经验，这种情况相当少见。

为了完整起见，我们*可以*用*涡轮鱼*给它一个*不同*的提示：

```rust ,ignore
    self.next = node.next.as_ref().map::<&Node<T>, _>(|node| &node);
```

看，map是一个泛型函数：

```rust ,ignore
pub fn map<U, F>(self, f: F) -> Option<U>
```

涡轮鱼，`:<>`，让我们告诉编译器我们认为这些泛型的类型应该是什么。在这个例子中
`::<&Node<T>, _>`说 "它应该返回一个`&Node<T>`，而我不知道/不关心其他类型"。

这反过来又让编译器知道`&node`应该被应用于解引用强制转换，所以我们不需要手动应用所有这
些\*!

但在这种情况下，我不认为这真的是一种改进，这只是一个薄薄的借口，来展示解引用强制转换和
有时有用的涡轮鱼。😅

让我们写一个测试，以确定我们没有拒绝它或任何东西：

```rust ,ignore
#[test]
fn iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

全对。

最后，应该指出的是，我们实际上可以在这里应用生命期消除：

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref() }
    }
}
```

等同于：

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.as_deref() }
    }
}
```

更少的生命期!

或者，如果你不愿意"隐藏"一个结构包含一个生命期，你可以使用Rust 2018的“
显式省略生命期”语法，`'_`：

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}
```
