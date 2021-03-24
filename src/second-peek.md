# 选择

有一件事我们上次甚至懒得实施，就是选择。让我们继续做这个。我们所要做的就是返回一
个对列表头部元素的引用，如果它存在的话。听起来很简单，让我们试试吧：

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.map(|node| {
        &node.elem
    })
}
```


```text
> cargo build

error[E0515]: cannot return reference to local data `node.elem`
  --> src/second.rs:37:13
   |
37 |             &node.elem
   |             ^^^^^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:36:9
   |
36 |         self.head.map(|node| {
   |         ^^^^^^^^^ cannot move out of borrowed content


```

*叹息*。现在怎么办，Rust？

map通过值来获取`self`，这将会把Option移出它所在的地方。以前这很好，因为我们只是把它
取出来，但是现在我们实际上想把它留在原处。处理这个问题的*正确*方法是使用Option的
`as_ref`方法，它的定义如下：

```rust ,ignore
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

它将 Option<T> 降级为对其内部的引用的 Option。我们可以通过显式匹配来实现这一点，但
是*不行*。这确实意味着我们需要做一个额外的取引用来切断额外的指示，但幸运的是`.`操作符
为我们处理了这个问题。


```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

```text
cargo build

    Finished dev [unoptimized + debuginfo] target(s) in 0.32s
```


钉住了。

我们也可以用`as_mut`制作这个方法的*可变*版本：

```rust ,ignore
pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| {
        &mut node.elem
    })
}
```

```text
lists::cargo build

```

EZ

别忘了测试一下：

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```
这很好，但我们并没有真正测试是否可以改变那个`peek_mut`的返回值，不是吗？如果一个引
用是可变的，但是没有人改变它，我们真的测试了可变性吗？让我们试试在这个
`Option<&mut T>`上使用`map`，放一个奥妙的值：

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
    list.peek_mut().map(|&mut value| {
        value = 42
    });

    assert_eq!(list.peek(), Some(&42));
    assert_eq!(list.pop(), Some(42));
}
```

```text
> cargo test

error[E0384]: cannot assign twice to immutable variable `value`
   --> src/second.rs:100:13
    |
99  |         list.peek_mut().map(|&mut value| {
    |                                   -----
    |                                   |
    |                                   first assignment to `value`
    |                                   help: make this binding mutable: `mut value`
100 |             value = 42
    |             ^^^^^^^^^^ cannot assign twice to immutable variable          ^~~~~
```
编译器抱怨说`value`是不可变的，但我们很清楚地写了`&mut value`；这又是怎么回事呢？事
实证明，这样写闭包的参数并没有说明`value`是一个可变的引用。相反，它创建了一个将与闭
包的参数相匹配的模式；`|&mut value|`意味着 "参数是一个可变的引用，但是请将它指向的值
复制到`value`中"。如果我们只使用`|value|`，`value`的类型将是`&mut i32`，我们就可以
真正地对头部进行改变：

```rust ,ignore
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));

        list.peek_mut().map(|value| {
            *value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

好多了！
