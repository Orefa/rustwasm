# 你应该知道的工具

这是在进行 Rust 和 WebAssembly 开发时你应该知道的一些很棒的工具的精选列表。

## 开发、构建和工作流编排

### `wasm-pack` | [repository](https://github.com/rustwasm/wasm-pack)

`wasm-pack` 旨在成为构建和使用 Rust 生成的 WebAssembly 的一站式商店，您希望与 JavaScript、Web 或 Node.js 进行互操作。 `wasm-pack` 可帮助您构建 Rust 生成的 WebAssembly 并将其发布到 npm 仓库，以便与您已经使用的工作流中的任何其他 JavaScript 包一起使用。 

## 优化和操作 `.wasm` 二进制文件

### `wasm-opt` | [repository](https://github.com/WebAssembly/binaryen)

`wasm-opt` 工具将 WebAssembly 读取为输入，对其运行转换、优化和/或检测传递，然后将转换后的 WebAssembly 作为输出发出。 在 LLVM 通过 `rustc` 生成的 `.wasm` 二进制文件上运行它通常会创建更小且执行速度更快的 `.wasm` 二进制文件。 这个工具是`binaryen` 项目的一部分。 

### `wasm2js` | [repository](https://github.com/WebAssembly/binaryen)

`wasm2js` 工具将 WebAssembly 编译成“几乎是 asm.js”。 这非常适合支持没有 WebAssembly 实现的浏览器，例如 Internet Explorer 11。这个工具是 `binaryen` 项目的一部分。

### `wasm-gc` | [repository](https://github.com/alexcrichton/wasm-gc)

一个垃圾收集 WebAssembly 模块并删除所有不需要的导出、导入、函数等的小工具。这实际上是 WebAssembly 的一个 `--gc-sections` 链接器标志。

由于两个原因，您通常不需要自己使用此工具：

1. `rustc` 现在有一个足够新的 `lld` 版本，它支持 WebAssembly 的 `--gc-sections` 标志。 这会自动为 LTO 构建启用。
2. `wasm-bindgen` CLI 工具会自动为你运行 `wasm-gc`。

### `wasm-snip` | [repository](https://github.com/rustwasm/wasm-snip)

`wasm-snip` 用 `unreachable` 指令替换了 WebAssembly 函数的主体。

也许您知道某些函数在运行时永远不会被调用，但是编译器无法在编译时证明这一点？ 剪吧！ 然后再次运行`wasm-gc`，它传递调用的所有函数（在运行时也永远不会被调用）也将被删除。

这对于在非调试生产版本中强行删除 Rust 的恐慌基础架构很有用。

## 检查 `.wasm` 二进制文件

### `twiggy` | [repository](https://github.com/rustwasm/twiggy)

`twiggy` 是 `.wasm` 二进制文件的代码大小分析器。 它分析二进制的调用图来回答以下问题：

* 为什么这个函数首先包含在二进制文件中？ IE。 哪些导出的函数正在传递调用它？
* 这个函数的保留大小是多少？ IE。 如果我删除它以及删除后成为死代码的所有函数，将节省多少空间。

使用 `twiggy` 使你的二进制文件变得苗条！

### `wasm-objdump` | [repository](https://github.com/WebAssembly/wabt)

打印有关 `.wasm` 二进制文件及其每个部分的低级详细信息。 还支持反汇编成 WAT 文本格式。 它就像`objdump`，但用于WebAssembly。 这是 WABT 项目的一部分。

### `wasm-nm` | [repository](https://github.com/fitzgen/wasm-nm)

列出在 `.wasm` 二进制文件中定义的导入、导出和私有函数符号。 它就像 `nm` 但对于 WebAssembly。 

