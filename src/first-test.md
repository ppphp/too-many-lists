# 测试

好了，我们已经写好了`push`和`pop`，现在我们可以真正测试我们的堆栈了!Rust和cargo支
持测试作为一级功能，所以这将是超级简单的。我们要做的就是写一个函数，然后用`#[test]`
来对它注解。

一般来说，在Rust社区中，我们尽量把测试放在它测试的代码旁边。然而我们通常会为测试做
一个新的命名空间，以避免与"真正的"代码冲突。就像我们使用`mod`来指定`first.rs`应该
包含在`lib.rs`中一样，我们可以使用`mod`来简单地创建一个全新的*内联*文件：


```rust ,ignore
// in first.rs

mod test {
    #[test]
    fn basics() {
        // TODO
    }
}
```

我们用`cargo test`调用它。

```text
> cargo test
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.00s
     Running /Users/ABeingessner/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

耶，我们的什么都没做的测试通过了! 让我们把它变得做一些事情。我们将通过`assert_eq!`
宏来实现。这不是什么特殊的测试魔法。它所做的就是比较你给它的两个东西，如果它们不匹配
，程序就会惊慌失措。是的，你通过惊慌失措向测试套装表示失败!

```rust ,ignore
mod test {
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
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

```text
> cargo test

error[E0433]: failed to resolve: use of undeclared type or module `List`
  --> src/first.rs:43:24
   |
43 |         let mut list = List::new();
   |                        ^^^^ use of undeclared type or module `List`


```

糟糕！因为我们做了一个新的模块，所以我们需要明确地拉入List来使用。

```rust ,ignore
mod test {
    use super::List;
    // everything else the same
}
```

```text
> cargo test

warning: unused import: `super::List`
  --> src/first.rs:45:9
   |
45 |     use super::List;
   |         ^^^^^^^^^^^
   |
   = note: #[warn(unused_imports)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running /Users/ABeingessner/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```
耶！

不过那个警告是怎么回事...? 我们在测试中明明用的是List!

...但只在测试时使用! 为了安抚编译器(并对我们的消费者友好)，我们应该说明整个测试模块只有
在运行测试时才应该被编译。


```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    // everything else the same
}
```

这就是测试的一切!
