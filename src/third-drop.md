# 丢弃

像可变列表一样，我们有一个递归的析构器问题。诚然，对于不可变的列表来说，这并不是一
个糟糕的问题：如果我们遇到*某处*另一个节点是另一个列表的头，我们就不会递归地放弃它。
然而，这仍然是一个我们应该关心的问题，而且如何处理并不那么明确。下面是我们之前解决这
个问题的方法：

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

问题在于循环的主体：

```rust ,ignore
cur_link = boxed_node.next.take();
```

这是在盒子里改变节点，但我们不能用Rc做这个；它只给我们共享访问权，因为任何数量的其他
Rc都可能指向它。

但是如果我们知道我们是最后一个知道这个节点的列表，那么将节点移出Rc其实是可以的。那么
我们也可以知道什么时候停止：只要我们*不能*把Node吊出来。

看看这个，Rc有一个方法正是这样做的：`try_unwrap`：

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut head = self.head.take();
        while let Some(node) = head {
            if let Ok(mut node) = Rc::try_unwrap(node) {
                head = node.next.take();
            } else {
                break;
            }
        }
    }
}
```

```text
cargo test
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/too-many-lists/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.10s
     Running /Users/ABeingessner/dev/too-many-lists/lists/target/debug/deps/lists-86544f1d97438f1f

running 8 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

好!
棒。
