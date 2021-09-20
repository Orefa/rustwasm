# 哪些板块能与WebAssembly一起使用现成的产品?

最简单的方法是列出目前不能与WebAssembly一起使用的东西；避免这些东西的crack往往可以移植到WebAssembly上，并且通常*Just Work*。一个好的经验法则是，如果一个板块支持嵌入式和 `#![no_std]` 的使用，它可能也支持WebAssembly。

## 一个板块可能会做的事情，不会与WebAssembly一起工作。

### C和系统库的依赖性

wasm中没有系统库，所以任何试图与系统库绑定的crate都不会工作。

使用C库也可能无法工作，因为wasm没有一个稳定的ABI用于跨语言通信，而且wasm的跨语言链接非常不稳定。每个人都希望这最终能起作用，特别是由于`clang`现在已经默认发送他们的`wasm32`目标，但故事还没有完全结束。

### 文件I/O

WebAssembly没有访问文件系统的权限，所以假设文件系统存在的crates &mdash; 没有针对WASM的解决方法 &mdash; 将不工作。

### 生成线程

有[计划将线程添加到WebAssembly][wasm-threading]，但它还没有发货。试图在 `wasm32-unknown-unknown`  目标上的线程上生成，会引起恐慌，从而触发wasm陷阱。

[wasm-threading]: https://rustwasm.github.io/2018/10/24/multithreading-rust-and-wasm.html

## 那么，哪些通用工具箱倾向于与WebAssembly一起工作？

### 算法和数据结构

提供特定[算法](https://crates.io/categories/algorithms)或[数据结构](https://crates.io/categories/data-structures)实现的板块，例如A*图搜索或splay树，往往能与WebAssembly很好地配合。

### `#![no_std]`

[不依赖于标准库的 crates](https://crates.io/categories/no-std) 往往与 WebAssembly 配合得很好。

### 解析器

[解析器](https://crates.io/categories/parser-implementations) &mdash; 只要他们只接受输入而不执行自己的 I/O  &mdash; 倾向于与 WebAssembly 配合使用。 

### 文本处理 

[以文本形式表达时处理人类语言复杂性的Crates](https://crates.io/categories/text-processing) 倾向于与 WebAssembly 配合使用。

### Rust 模式

[针对特定于 Rust 编程的特定情况的共享解决方案](https://crates.io/categories/rust-patterns) 倾向于与 WebAssembly 配合使用。
