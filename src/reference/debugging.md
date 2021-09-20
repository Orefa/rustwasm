# 调试 Rust 生成的 WebAssembly

本节包含调试 Rust 生成的 WebAssembly 的技巧。

## 使用调试符号构建

> ⚡ 调试时，请始终确保使用调试符号进行构建！

如果你没有启用调试符号，那么在编译的 `.wasm` 二进制文件中将不会出现 `"name"` 自定义部分，并且堆栈跟踪将具有类似 `wasm-function[42]` 的函数名称，而不是函数的 Rust 名称，比如 `wasm_game_of_life::Universe::live_neighbor_count`。

当使用 "debug" 构建（又名 `wasm-pack build --debug` 或 `cargo build`）时，默认启用调试符号。

对于 "release" 版本，默认情况下不启用调试符号。 要启用调试符号，请确保在 `Cargo.toml` 的 `[profile.release]` 部分中设置 `debug = true`：

```toml
[profile.release]
debug = true
```

## 使用 `console` API 进行日志记录

日志记录是我们拥有的最有效的工具之一，用于证明和反驳关于我们的程序为什么有错误的假设。 在 Web 上，[`console.log` 功能](https://developer.mozilla.org/en-US/docs/Web/API/Console/log) 是将消息记录到浏览器的开发者工具控制台的方式 .

我们可以使用 [the `web-sys` crate][web-sys] 来访问 `console` 日志功能:

```rust
extern crate web_sys;

web_sys::console::log_1(&"Hello, world!".into());
```

或者，[`console.error` 函数](https://developer.mozilla.org/en-US/docs/Web/API/Console/error) 具有与 `console.log` 相同的签名，但开发人员工具 当使用 `console.error` 时，往往还会在记录的消息旁边捕获和显示堆栈跟踪。

### References

* 使用 `console.log` 和 `web-sys` crate:
  * [`web_sys::console::log` 取一个数组的值来记录](https://rustwasm.github.io/wasm-bindgen/api/web_sys/console/fn.log.html)
  * [`web_sys::console::log_1` 记录一个单一的值](https://rustwasm.github.io/wasm-bindgen/api/web_sys/console/fn.log_1.html)
  * [`web_sys::console::log_2` 记录了两个值](https://rustwasm.github.io/wasm-bindgen/api/web_sys/console/fn.log_2.html)
  * Etc...
* 使用 `console.error` 与 `web-sys` crate:
  * [`web_sys::console::error` 取一个数组的值来记录](https://rustwasm.github.io/wasm-bindgen/api/web_sys/console/fn.error.html)
  * [`web_sys::console::error_1` 记录一个单一的值](https://rustwasm.github.io/wasm-bindgen/api/web_sys/console/fn.error_1.html)
  * [`web_sys::console::error_2` 记录了两个值](https://rustwasm.github.io/wasm-bindgen/api/web_sys/console/fn.error_2.html)
  * Etc...
* [MDN上的 `console` 对象](https://developer.mozilla.org/en-US/docs/Web/API/Console)
* [火狐浏览器开发者工具 - 网络控制台](https://developer.mozilla.org/en-US/docs/Tools/Web_Console)
* [微软EDGE开发工具 - 控制台](https://docs.microsoft.com/en-us/microsoft-edge/devtools-guide/console)
* [开始使用 Chrome DevTools 控制台](https://developers.google.com/web/tools/chrome-devtools/console/get-started)

## 记录 Panics

[`console_error_panic_hook` crate 通过 `console.error` 将意外的恐慌记录到开发者控制台。][panic-hook] 而不是获得神秘的、难以调试的 `RuntimeError: unreachable execution` 错误消息，这给你 Rust 的格式化 恐慌信息。

您需要做的就是通过在初始化函数或公共代码路径中调用 `console_error_panic_hook::set_once()` 来安装钩子： 

```rust
#[wasm_bindgen]
pub fn init_panic_hook() {
    console_error_panic_hook::set_once();
}
```

[panic-hook]: https://github.com/rustwasm/console_error_panic_hook

## 使用调试器

不幸的是，WebAssembly 的调试故事仍然不成熟。 在大多数 Unix 系统上，[DWARF][dwarf] 用于对调试器提供正在运行的程序的源代码级检查所需的信息进行编码。 有一种替代格式可以对 Windows 上的类似信息进行编码。 目前，没有 WebAssembly 的等价物。 因此，调试器目前提供的实用程序有限，我们最终会逐步执行编译器发出的原始 WebAssembly 指令，而不是我们编写的 Rust 源文本。

> 有一个[W3C WebAssembly 小组调试子章程][debugging-subcharter]，所以期待这个故事在未来得到改进！ 

[debugging-subcharter]: https://github.com/WebAssembly/debugging
[dwarf]: http://dwarfstd.org/

尽管如此，调试器对于检查与我们的 WebAssembly 交互的 JavaScript 和检查原始 wasm 状态仍然很有用。

### 参考

* [Firefox 开发者工具 — 调试器](https://developer.mozilla.org/en-US/docs/Tools/Debugger)
* [Microsoft Edge 开发者工具 — 调试器](https://docs.microsoft.com/en-us/microsoft-edge/devtools-guide/debugger)
* [在 Chrome DevTools 中开始调试 JavaScript](https://developers.google.com/web/tools/chrome-devtools/javascript/)

## 首先避免调试 WebAssembly

如果该错误特定于与 JavaScript 或 Web API 的交互，则 [使用 `wasm-bindgen-test` 编写测试。][wbg-test]

如果错误*不*涉及与 JavaScript 或 Web API 的交互，那么尝试将其复制为普通的 Rust `#[test]` 函数，您可以在调试时利用操作系统成熟的本机工具。 使用像 [`quickcheck`][quickcheck] 和它的测试用例收缩器这样的测试箱来机械地减少测试用例。 最终，如果您可以在不需要与 JavaScript 交互的较小测试用例中隔离它们，您将更容易找到和修复错误。

请注意，为了在没有编译器和链接器错误的情况下运行本机 `#[test]`，您需要确保 `"rlib"` 包含在 `Cargo. toml` 文件。

```toml
[lib]
crate-type ["cdylib", "rlib"]
```

[quickcheck]: https://crates.io/crates/quickcheck
[web-sys]: https://rustwasm.github.io/wasm-bindgen/web-sys/index.html
[wbg-test]: https://rustwasm.github.io/wasm-bindgen/wasm-bindgen-test/index.html
