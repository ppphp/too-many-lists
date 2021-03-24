# 丢弃

我们可以做一个堆栈，对它推入，弹出，我们甚至已经测试过，这一切都正确！

我们需要担心清理我们的列表吗？从技术上讲，不，完全不需要！Rust和C++一样，使用
destructors来自动清理列表。像C++一样，Rust使用析构器来自动清理资源。如果一个
类型实现了一个叫做Drop的*特质*，它就有一个析构器。特质是Rust对接口的花哨称呼。
Drop特质有以下接口：

```rust ,ignore
pub trait Drop {
    fn drop(&mut self);
}
```

基本上就是，“当你离开作用域时，我会给你一点时间来清理你的事务”。

如果你包含了实现Drop的类型，你其实并不需要实现Drop，你要做的只是调用*它们*的析构函
数。在List的情况下，它所要做的就是丢弃它的头部，而头部又*可能*会尝试丢弃一个
`Box<Node>`。所有这些都会自动为我们处理......但有一个问题。

自动处理的效果会很差。

让我们考虑一个简单的列表：


```text
list -> A -> B -> C
```

当`list`被丢弃时，它会尝试丢弃A，会尝试丢弃B，会尝试丢弃C，你们中的一些人可能会正确
地开始紧张。这是递归代码，而递归代码会炸掉堆栈!

你们中的一些人可能会想："这显然是尾部递归，任何体面的语言都会确保这种代码不会炸堆"。
事实上，这是不正确的! 为了了解原因，让我们试着写出编译器要做的事情，为我们的List手动
实现Drop，就像编译器一样：


```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        // NOTE: you can't actually explicitly call `drop` in real Rust code;
        // we're pretending to be the compiler!
        self.head.drop(); // tail recursive - good!
    }
}

impl Drop for Link {
    fn drop(&mut self) {
        match *self {
            Link::Empty => {} // Done!
            Link::More(ref mut boxed_node) => {
                boxed_node.drop(); // tail recursive - good!
            }
        }
    }
}

impl Drop for Box<Node> {
    fn drop(&mut self) {
        self.ptr.drop(); // uh oh, not tail recursive!
        deallocate(self.ptr);
    }
}

impl Drop for Node {
    fn drop(&mut self) {
        self.next.drop();
    }
}
```

我们*不能*在回收内存*之后*丢弃Box的内容，所以没有办法以尾部递归的方式丢弃！而我们要
手动为`List`写一个迭代丢弃，将节点从它们的盒子里吊出来。


```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);
        // `while let` == "do this thing until this pattern doesn't match"
        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
            // boxed_node goes out of scope and gets dropped here;
            // but its Node's `next` field has been set to Link::Empty
            // so no unbounded recursion occurs.
        }
    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

```

棒！

----------------------

<span style="float:left">![Bonus](img/profbee.gif)</span>

## 提前优化的奖励部分！

我们对drop的实现其实和`while let Some(_) = self.pop() { }`*很*相似，当然更简单。
它有什么不同，一旦我们开始泛化我们的列表来存储整数以外的东西，会导致什么性能问题？

<details>
  <summary>Click to expand for answer</summary>

Pop返回`Option<i32>`，而我们的实现只操作Links（`Box<Node>`）。所以我们的实现只移动节点的指针，而基于pop的实现将移动我们存储在节点中的值。如果我们通用我们的列表，有人用它来存储很大的有丢弃实现的东西(VBTWADI)的实例，这可能会非常昂贵。Box能够在原地运行其内容的drop实现，所以它不会受到这个问题的影响。由于VBTWADI正是实际上使用链表比使用数组更可取，所以在这种情况下表现不佳会让人有点失望。

如果你希望同时拥有两种实现的优点，你可以添加一个新的方法，`fn pop_node(&mut self) -> Link`，`pop`和`drop`都可以干净利落地从它派生出来。

</details>
