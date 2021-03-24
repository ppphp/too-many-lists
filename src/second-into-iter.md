# 进入遍历

在Rust中使用*Iterator*特性对集合进行迭代。这比`Drop`要复杂一点：

```rust ,ignore
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

块里的新东西是`type Item`。这声明了每个迭代器的实现都有一个*关联类型*，叫做Item。在
这种情况下，当你调用`next`时，它可以吐出这个类型。

Iterator产生`Option<Self::Item>`的原因是该接口集成了`has_next`和`get_next`的概念。
当你有下一个值时，你产生`Some(value)`，而当你没有时，你产生`None`。这使得API在使用
和实现上更加符合人体工程学和安全，同时避免了`has_next`和`get_next`之间多余的检查和
逻辑。很好!

遗憾的是，Rust（现在还）没有类似`yield`语句的东西，所以我们必须自己实现这个逻辑。另
外，每个集合应该努力实现3种不同的迭代器：

* IntoIter - `T`
* IterMut - `&mut T`
* Iter - `&T`

实际上，我们已经有了使用List接口实现IntoIter的所有工具：只需不断地调用`pop`。因此，
我们只需将IntoIter作为List的一个新类型包装器来实现：


```rust ,ignore
// Tuple structs are an alternative form of struct,
// useful for trivial wrappers around other types.
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // access fields of a tuple struct numerically
        self.0.pop()
    }
}
```

然后让我们写一个测试：

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next(), Some(1));
    assert_eq!(iter.next(), None);
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 4 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

棒！
