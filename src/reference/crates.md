# 你应该知道的Crates

这是您在进行 Rust 和 WebAssembly 开发时应该了解的很棒的 crate 的精选列表。 

[您还可以在 WebAssembly 类别中浏览所有发布到 crates.io 的 crate.][wasm-category]

## 与 JavaScript 和 DOM 交互

### `wasm-bindgen` | [crates.io](https://crates.io/crates/wasm-bindgen) | [repository](https://github.com/rustwasm/wasm-bindgen)

`wasm-bindgen` 促进了 Rust 和 JavaScript 之间的高级交互。 它允许将 JavaScript 内容导入 Rust 并将 Rust 内容导出到 JavaScript。 

### `wasm-bindgen-futures` | [crates.io](https://crates.io/crates/wasm-bindgen-futures) | [repository](https://github.com/rustwasm/wasm-bindgen/tree/master/crates/futures)

`wasm-bindgen-futures` 是连接 JavaScript `Promise`s 和 Rust `Future`s 的桥梁。 它可以双向转换，在 Rust 中处理异步任务时很有用，并允许与 DOM 事件和 I/O 操作进行交互。

### `js-sys` | [crates.io](https://crates.io/crates/js-sys) | [repository](https://github.com/rustwasm/wasm-bindgen/tree/master/crates/js-sys)

所有 JavaScript 全局类型和方法的原始 `wasm-bindgen` 导入，例如 `Object`、`Function`、`eval` 等。这些 API 可以在所有标准 ECMAScript 环境中移植，而不仅仅是 Web，例如 Node.js。

### `web-sys` | [crates.io](https://crates.io/crates/web-sys) | [repository](https://github.com/rustwasm/wasm-bindgen/tree/master/crates/web-sys)

所有 Web API 的原始 `wasm-bindgen` 导入，例如 DOM 操作、`setTimeout`、Web GL、Web Audio 等。

## 错误报告和日志记录

### `console_error_panic_hook` | [crates.io](https://crates.io/crates/console_error_panic_hook) | [repository](https://github.com/rustwasm/console_error_panic_hook)

这个 crate 允许你通过提供一个将 panic 消息转发到 `console.error` 的 panic 钩子来调试 `wasm32-unknown-unknown` 上的 panics。

### `console_log` | [crates.io](https://crates.io/crates/console_log) | [repository](https://github.com/iamcodemaker/console_log)

此 crate 为 [the `log` crate](https://crates.io/crates/log) 提供后端，将记录的消息路由到 devtools 控制台。

## 动态分配

### `wee_alloc` | [crates.io](https://crates.io/crates/wee_alloc) | [repository](https://github.com/rustwasm/wee_alloc)

**W**asm-**E**nabled，**E**lfin 分配器。 当代码大小比分配性能更受关注时，一个小的（~1K 未压缩的`.wasm`）分配器实现。 

## 解析和生成 `.wasm` 二进制文件

### `parity-wasm` | [crates.io](https://crates.io/crates/parity-wasm) | [repository](https://github.com/paritytech/parity-wasm)

用于序列化、反序列化和构建 `.wasm` 二进制文件的低级 WebAssembly 格式库。 对众所周知的自定义部分的良好支持，例如 "names" 部分和 "reloc.WHATEVER" 部分。 

### `wasmparser` | [crates.io](https://crates.io/crates/wasmparser) | [repository](https://github.com/yurydelendik/wasmparser.rs)

一个简单的、事件驱动的库，用于解析 WebAssembly 二进制文件。 提供每个已解析事物的字节偏移量，例如在解释 relocs 时这是必需的。

## 解释和编译 WebAssembly

### `wasmi` | [crates.io](https://crates.io/crates/wasmi) | [repository](https://github.com/paritytech/wasmi)

Parity 的可嵌入 WebAssembly 解释器。

### `cranelift-wasm` | [crates.io](https://crates.io/crates/cranelift-wasm) | [repository](https://github.com/bytecodealliance/wasmtime/tree/master/cranelift)

将 WebAssembly 编译为本机主机的机器代码。 Cranelift (né Cretonne) 代码生成器项目的一部分。 

[wasm-category]: https://crates.io/categories/wasm
