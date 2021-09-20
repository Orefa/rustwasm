# 时间分析

本节介绍如何使用 Rust 和 WebAssembly 分析网页，其目标是提高吞吐量或延迟。

> ⚡ 始终确保在分析时使用优化的构建！ `wasm-pack build` 将默认进行优化构建。 

## 可用工具

### `window.performance.now()` 计时器

[`performance.now()` 函数][perf-now] 返回自网页加载以来以毫秒为单位的单调时间戳。

调用 `performance.now` 的开销很小，因此我们可以从中创建简单、细粒度的测量，而不会扭曲系统其余部分的性能并对我们的测量造成偏差。

我们可以用它来为各种操作计时，我们可以通过[`web-sys` crate][web-sys] 访问`window.performance.now()`:

```rust
extern crate web_sys;

fn now() -> f64 {
    web_sys::window()
        .expect("should have a Window")
        .performance()
        .expect("should have a Performance")
        .now()
}
```

* [The `web_sys::window` function](https://rustwasm.github.io/wasm-bindgen/api/web_sys/fn.window.html)
* [The `web_sys::Window::performance` method](https://rustwasm.github.io/wasm-bindgen/api/web_sys/struct.Window.html#method.performance)
* [The `web_sys::Performance::now` method](https://rustwasm.github.io/wasm-bindgen/api/web_sys/struct.Performance.html#method.now)

[perf-now]: https://developer.mozilla.org/en-US/docs/Web/API/Performance/now

### 开发者工具分析器

所有 Web 浏览器的内置开发人员工具都包含一个分析器。 这些分析器通过调用树和火焰图等常见类型的可视化显示哪些函数占用的时间最多。

如果您 [使用调试符号构建][symbols] 以便“名称”自定义部分包含在 wasm 二进制文件中，那么这些分析器应该显示 Rust 函数名称，而不是像 `wasm-function[123]` 这样不透明的东西。

请注意，这些分析器*不会*显示内联函数，并且由于 Rust 和 LLVM 非常依赖内联，因此结果可能仍然有点令人困惑。

[symbols]: ./debugging.html#building-with-debug-symbols

[![Screenshot of profiler with Rust symbols](../images/game-of-life/profiler-with-rust-names.png)](../images/game-of-life/profiler-with-rust-names.png)

#### 资源

* [Firefox 开发者工具——性能](https://developer.mozilla.org/en-US/docs/Tools/Performance)
* [Microsoft Edge 开发者工具——性能](https://docs.microsoft.com/en-us/microsoft-edge/devtools-guide/performance)
* [Chrome DevTools JavaScript Profiler](https://developers.google.com/web/tools/chrome-devtools/rendering-tools/js-execution)

### `console.time` 和 `console.timeEnd` 函数 

[`console.time` 和 `console.timeEnd` 函数][console-time] 允许您将命名操作的时间记录到浏览器的开发者工具控制台。 你在操作开始时调用`console.time("some operation")`，在操作结束时调用`console.timeEnd("some operation")`。 命名操作的字符串标签是可选的。

您可以直接通过 [`web-sys` crate][web-sys] 使用这些函数：

* [`web_sys::console::time_with_label("some operation")`](https://rustwasm.github.io/wasm-bindgen/api/web_sys/console/fn.time_with_label.html)
* [`web_sys::console::time_end_with_label("some operation")`](https://rustwasm.github.io/wasm-bindgen/api/web_sys/console/fn.time_end_with_label.html)

这是浏览器控制台中 `console.time` 日志的屏幕截图：

[![Screenshot of console.time logs](../images/game-of-life/console-time.png)](../images/game-of-life/console-time.png)

此外，`console.time` 和 `console.timeEnd` 日志将显示在浏览器分析器的“时间轴”或“瀑布”视图中： 

[![Screenshot of console.time logs](../images/game-of-life/console-time-in-profiler.png)](../images/game-of-life/console-time-in-profiler.png)

[console-time]: https://developer.mozilla.org/en-US/docs/Web/API/Console/time

### 在本机代码中使用 `#[bench]`

与我们通常可以通过编写 `#[test]`s 而不是在 Web 上调试来利用操作系统的本机代码调试工具一样，我们可以通过编写 `#[bench]` 函数来利用操作系统的本机代码分析工具。

在 crate 的 `benches` 子目录中写入你的基准。 确保你的 `crate-type` 包含 `"rlib"`，否则 bench 二进制文件将无法链接你的主库。

然而！ 在投入大量精力进行本机代码分析之前，请确保您知道瓶颈在 WebAssembly 中！ 使用浏览器的分析器来确认这一点，否则您可能会浪费时间优化不热门的代码。 

#### 资源

* [在Linux上使用`perf`剖析器](http://www.brendangregg.com/perf.html)
* [在MacOS上使用Instruments.app分析器](https://help.apple.com/instruments/mac/current/)
* [VTune剖析器支持Windows和Linux](https://software.intel.com/en-us/vtune)

[web-sys]: https://rustwasm.github.io/wasm-bindgen/web-sys/index.html
