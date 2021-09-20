# 添加交互性

我们将继续探索 JavaScript 和 WebAssembly 界面，为我们的 Game of Life 实现添加一些交互功能。 我们将使用户能够通过单击来切换单元格是活的还是死的，并允许暂停游戏，这使得绘制单元格图案变得更加容易。 

## 暂停和恢复游戏

让我们添加一个按钮来切换游戏是正在播放还是暂停。 在 `wasm-game-of-life/www/index.html` 中，在 `<canvas>` 正上方添加按钮：

```html
<button id="play-pause"></button>
```

在 `wasm-game-of-life/www/index.js` JavaScript 中，我们将进行以下更改：

* 跟踪最近一次调用`requestAnimationFrame` 返回的标识符，以便我们可以通过使用该标识符调用`cancelAnimationFrame` 来取消动画。

* 当播放/暂停按钮被点击时，检查我们是否有排队动画帧的标识符。如果我们这样做，那么游戏当前正在播放，我们想要取消动画帧，以便不再调用 `renderLoop`，从而有效地暂停游戏。如果我们没有排队动画帧的标识符，那么我们当前处于暂停状态，我们想调用`requestAnimationFrame` 来恢复游戏。

因为 JavaScript 正在驱动 Rust 和 WebAssembly，这就是我们需要做的所有事情，我们不需要更改 Rust 源。

我们引入了`animationId` 变量来跟踪`requestAnimationFrame` 返回的标识符。当没有排队的动画帧时，我们将此变量设置为 `null`。

```js
let animationId = null;

// This function is the same as before, except the
// result of `requestAnimationFrame` is assigned to
// `animationId`.
const renderLoop = () => {
  drawGrid();
  drawCells();

  universe.tick();

  animationId = requestAnimationFrame(renderLoop);
};
```

在任何时候，我们都可以通过检查 `animationId` 的值来判断游戏是否暂停：

```js
const isPaused = () => {
  return animationId === null;
};
```

现在，当点击播放/暂停按钮时，我们检查游戏当前是否暂停或播放，并分别恢复`renderLoop`动画或取消下一动画帧。 此外，我们更新按钮的文本图标以反映按钮在下一次单击时将采取的操作。

```js
const playPauseButton = document.getElementById("play-pause");

const play = () => {
  playPauseButton.textContent = "⏸";
  renderLoop();
};

const pause = () => {
  playPauseButton.textContent = "▶";
  cancelAnimationFrame(animationId);
  animationId = null;
};

playPauseButton.addEventListener("click", event => {
  if (isPaused()) {
    play();
  } else {
    pause();
  }
});
```

最后，我们之前通过直接调用 `requestAnimationFrame(renderLoop)` 来启动游戏及其动画，但我们想用对 `play` 的调用替换它，以便按钮获得正确的初始文本图标。

```diff
// This used to be `requestAnimationFrame(renderLoop)`.
play();
```

刷新 [http://localhost:8080/](http://localhost:8080/) 我们现在应该可以通过点击按钮暂停和恢复游戏了！ 

## 在 `"click"` 事件上切换单元格的状态

现在我们可以暂停游戏了，是时候添加通过单击单元格来变异单元格的功能了。

切换一个单元格就是将它的状态从活着变为死亡或从死亡变为活着。 向 `wasm-game-of-life/src/lib.rs` 中的 `Cell` 添加 `toggle` 方法：

```rust
impl Cell {
    fn toggle(&mut self) {
        *self = match *self {
            Cell::Dead => Cell::Alive,
            Cell::Alive => Cell::Dead,
        };
    }
}
```

要在给定的行和列处切换单元格的状态，我们将行和列对转换为单元格向量的索引，并在该索引处的单元格上调用 toggle 方法：

```rust
/// Public methods, exported to JavaScript.
#[wasm_bindgen]
impl Universe {
    // ...

    pub fn toggle_cell(&mut self, row: u32, column: u32) {
        let idx = self.get_index(row, column);
        self.cells[idx].toggle();
    }
}
```

这个方法被定义在`impl`块中，该块被注释为`#[wasm_bindgen]`，这样它就可以被JavaScript调用。

在`wasm-game-of-life/www/index.js`中，我们监听`<canvas>`元素上的点击事件，将点击事件的页面相对坐标转换为画布相对坐标，然后转换为行和列，调用`toggle_cell`方法，最后重新绘制场景。

```js
canvas.addEventListener("click", event => {
  const boundingRect = canvas.getBoundingClientRect();

  const scaleX = canvas.width / boundingRect.width;
  const scaleY = canvas.height / boundingRect.height;

  const canvasLeft = (event.clientX - boundingRect.left) * scaleX;
  const canvasTop = (event.clientY - boundingRect.top) * scaleY;

  const row = Math.min(Math.floor(canvasTop / (CELL_SIZE + 1)), height - 1);
  const col = Math.min(Math.floor(canvasLeft / (CELL_SIZE + 1)), width - 1);

  universe.toggle_cell(row, col);

  drawGrid();
  drawCells();
});
```

在`wasm-game-of-life`中用`wasm-pack build`重新构建，然后再次刷新[http://localhost:8080/](http://localhost:8080/)，现在我们可以通过点击单元格并切换其状态来绘制我们自己的图案。s

## 练习

* 引入一个[`<input type="range">`][input-range]小部件来控制每一帧动画发生多少次。

* 添加一个按钮，当点击时将宇宙重置到一个随机的初始状态。另一个按钮可以将宇宙重设为所有死单元。

* 在 `Ctrl + Click` 时，插入一个 [glider](https://en.wikipedia.org/wiki/Glider_(Conway%27s_Life))，以目标单元为中心。在`Shift + Click`上，插入一个脉冲星。

[input-range]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/range
