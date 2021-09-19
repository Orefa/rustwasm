# 教程: 康威生命游戏

这是一个用 Rust 和 WebAssembly 实现[康威生命游戏][gol]的教程。

[gol]: https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life

## 这个教程是为谁准备的？

本教程适用于已经有基本 Rust 和 JavaScript 经验的人，并希望学习如何一起使用 Rust、WebAssembly 和 JavaScript。

你应该能够自如地阅读和编写基本的Rust、JavaScript 和 HTML。你绝对不需要是一个专家。


## 我将学到什么？

* 如何设置Rust工具链以编译成WebAssembly。

* 一个用于开发由Rust、WebAssembly、JavaScript、HTML和CSS组成的多语言程序的工作流程。

* 如何设计 API 以最大限度地利用 Rust 和 WebAssembly 的优势，同时也是 JavaScript 的优势。

* 如何调试由Rust编译的WebAssembly模块。

* 如何对Rust和WebAssembly程序进行时间剖析以使其更快。

* 如何确定 Rust 和 WebAssembly 程序的大小，使`.wasm`二进制文件更小，更快通过网络下载。