# 一个不好的单链栈

这章将是*至今*最长的一章，因为我们需要介绍所有Rust的基础知识，而且将以“艰难的方式”构建一些东西来
更好地理解这门语言。

我们将把我们的第一个链表放在`src/first.rs`。我们需要告诉Rust`first.rs`是我们的库用的东西。需
要做的事是我们把它放在`src/lib.rs`（Cargo已经给我们创建了）最上面：

```rust ,ignore
// in lib.rs
pub mod first;
```

