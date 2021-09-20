# 调试

在我们写更多的代码之前，我们会想在我们的腰带上有一些调试工具，以备出错时使用。花点时间回顾一下[参考页列出了可用来调试Rust生成的WebAssembly的工具和方法][reference-debugging].

[reference-debugging]: ../reference/debugging.html

## 为 Panics 启用日志记录

[如果我们的代码出现混乱，我们希望在开发者控制台中显示信息性错误消息。](../reference/debugging.html#记录-panics)

我们的 `wasm-pack-template` 带有一个可选的、默认启用的对 [`console_error_panic_hook` crate][panic-hook] 的依赖，该依赖被配置在 `wasm-game-of-life/src/utils.rs` 中。我们需要做的就是在初始化函数或公共代码路径中安装这个钩子。我们可以在 `wasm-ame-of-life/src/lib.rs` 中的 `Universe::new` 构造函数中调用它。

```rust
pub fn new() -> Universe {
    utils::set_panic_hook();

    // ...
}
```

[panic-hook]: https://github.com/rustwasm/console_error_panic_hook

## 在我们的生活游戏中加入记录的内容

让我们[通过`web-sys` crate 使用 `console.log` 函数来添加一些关于我们 `Universe::tick` 函数中每个单元的日志][logging]。

首先，添加 `web-sys` 作为依赖，并在 `wasm-game-of-life/Cargo.toml` 中启用其`"console"`功能:

```toml
[dependencies]

# ...

[dependencies.web-sys]
version = "0.3"
features = [
  "console",
]
```

为了符合人体工程学，我们将把`console.log`函数包装成一个`println!`风格的宏:

[logging]: ../reference/debugging.html#logging-with-the-console-apis

```rust
extern crate web_sys;

// A macro to provide `println!(..)`-style syntax for `console.log` logging.
macro_rules! log {
    ( $( $t:tt )* ) => {
        web_sys::console::log_1(&format!( $( $t )* ).into());
    }
}
```

现在，我们可以通过在Rust代码中插入对`log`的调用来开始将信息记录到控制台。例如，为了记录每个单元格的状态、活的邻居数和下一个状态，我们可以这样修改`wasm-game-of-life/src/lib.rs`:

```diff
diff --git a/src/lib.rs b/src/lib.rs
index f757641..a30e107 100755
--- a/src/lib.rs
+++ b/src/lib.rs
@@ -123,6 +122,14 @@ impl Universe {
                 let cell = self.cells[idx];
                 let live_neighbors = self.live_neighbor_count(row, col);

+                log!(
+                    "cell[{}, {}] is initially {:?} and has {} live neighbors",
+                    row,
+                    col,
+                    cell,
+                    live_neighbors
+                );
+
                 let next_cell = match (cell, live_neighbors) {
                     // Rule 1: Any live cell with fewer than two live neighbours
                     // dies, as if caused by underpopulation.
@@ -140,6 +147,8 @@ impl Universe {
                     (otherwise, _) => otherwise,
                 };

+                log!("    it becomes {:?}", next_cell);
+
                 next[idx] = next_cell;
             }
         }
```

## 使用调试器在每个 Tick 之间进行停顿

[浏览器的步进调试器对于检查我们的 Rust 生成的 WebAssembly 与之交互的 JavaScript 很有用。](../reference/debugging.html#using-a-debugger)

例如，我们可以使用调试器通过将 [一个 JavaScript `debugger;` 语句][dbg-stmt] 放在我们对 `universe.tick()` 的调用上方来暂停 `renderLoop` 函数的每次迭代。

```js
const renderLoop = () => {
  debugger;
  universe.tick();

  drawGrid();
  drawCells();

  requestAnimationFrame(renderLoop);
};
```

这为我们提供了一个方便的检查点，用于检查记录的消息，并将当前渲染的帧与前一帧进行比较。

[dbg-stmt]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/debugger

[![Screenshot of debugging the Game of Life](../images/game-of-life/debugging.png)](../images/game-of-life/debugging.png)

## 练习

* 向 `tick` 函数添加日志记录，记录每个单元格的行和列，这些单元格将状态从活动状态转换为死状态，反之亦然。

* 在 `Universe::new` 方法中引入 `panic!()`。 在 Web 浏览器的 JavaScript 调试器中检查恐慌的回溯。 禁用调试符号，在没有 `console_error_panic_hook` 可选依赖项的情况下重建，并再次检查堆栈跟踪。 不是那么有用吗？ 
