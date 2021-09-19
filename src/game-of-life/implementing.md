# 实现康威的生命游戏

## 设计

在我们深入研究之前，我们需要考虑一些设计选择。 

### 无限宇宙 

生命游戏是在一个无限的宇宙中进行的，但我们没有无限的内存和计算能力。 解决这个相当烦人的限制通常有以下三种方式之一： 

1. 跟踪宇宙的哪个子集发生了有趣的事情，并根据需要扩展该区域。 在最坏的情况下，这种扩展是无界的，实现会越来越慢，最终会耗尽内存。 

2. 创建一个固定大小的宇宙，其中边缘的单元格比中间的单元格具有更少的邻居。 这种方法的缺点是，像滑翔机一样到达宇宙尽头的无限模式被扼杀了。 

3. 创建一个固定大小的周期性宇宙，其中边缘的细胞有环绕宇宙另一侧的邻居。 因为邻居环绕着宇宙的边缘，所以滑翔机可以永远运行。 

我们将实施第三个选项。

### Rust 和 JavaScript 的接口 

> ⚡ 这是本教程中需要理解和掌握的最重要的概念之一！

JavaScript 的垃圾收集堆——其中分配了“对象”、“数组”和 DOM 节点——与 WebAssembly 的线性内存空间不同，我们的 Rust 值存在于其中。 WebAssembly 目前无法直接访问垃圾收集堆（截至 2018 年 4 月，这预计会随着 [“接口类型”提案][interface-types] 的出现而改变）。 另一方面，JavaScript可以读写WebAssembly的线性内存空间，但只能作为标量值的[`ArrayBuffer`][array-buf]（`u8`，`i32`，`f64`，等等）。 WebAssembly 函数也接受和返回标量值。 这些是构成所有 WebAssembly 和 JavaScript 通信的构建块。 

[interface-types]: https://github.com/WebAssembly/interface-types/blob/master/proposals/interface-types/Explainer.md
[array-buf]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer

`wasm_bindgen` 定义了如何跨这个边界处理复合结构的共同理解。 它涉及装箱 Rust 结构，并将指针包装在 JavaScript 类中以提高可用性，或者从 Rust 索引到 JavaScript 对象表。 `wasm_bindgen` 非常方便，但它并没有消除考虑我们的数据表示的需要，以及跨越这个边界传递的值和结构。 相反，将其视为实现您选择的界面设计的工具。

在设计 WebAssembly 和 JavaScript 之间的接口时，我们希望针对以下属性进行优化： 

1. **最大限度地减少进出 WebAssembly 线性内存的复制。** 不必要的副本会带来不必要的开销。 

2. **最小化序列化和反序列化。** 与副本类似，序列化和反序列化也会产生开销，并且通常也会产生复制。 如果我们可以将不透明的句柄传递给数据结构——而不是在一侧序列化它，将它复制到 WebAssembly 线性内存中的某个已知位置，然后在另一侧反序列化——我们通常可以减少很多开销。 `wasm_bindgen` 帮助我们定义和使用 JavaScript 对象或盒装 Rust 结构的不透明句柄。 

### 在我们的生活游戏中连接 Rust 和 JavaScript 

让我们首先列举一些要避免的危险。 我们不想在每个滴答声中将整个宇宙复制到 WebAssembly 线性内存中。 我们不想为宇宙中的每个单元格分配对象，也不想强加跨界调用来读取和写入每个单元格。

这让我们何去何从？ 我们可以将宇宙表示为一个平面数组，它存在于 WebAssembly 线性内存中，每个单元格都有一个字节。 `0` 是死细胞，`1` 是活细胞。

以下是 4 x 4 宇宙在内存中的样子： 

![Screenshot of a 4 by 4 universe](../images/game-of-life/universe.png)

为了找到宇宙中某一行和某一列的单元格的阵列索引，我们可以使用这个公式：

```text
index(row, column, universe) = row * width(universe) + column
```

我们有几种方法可以将宇宙的单元格暴露给 JavaScript。首先，我们将为 "宇宙 "实现[`std::fmt::Display`][`Display`]，我们可以用它来生成一个渲染为文本字符的单元格的Rust`String`。然后这个Rust字符串从WebAssembly的线性内存中复制到JavaScript的垃圾收集堆中的一个JavaScript字符串，然后通过设置HTML`textContent`来显示。在本章的后面，我们将发展这个实现，以避免在堆之间复制宇宙的单元，并渲染到`<canvas>`。

*另一个可行的设计方案是让Rust在每次打勾后返回一个改变状态的单元格的列表，而不是将整个宇宙暴露给JavaScript。这样一来，JavaScript就不需要在渲染时遍历整个宇宙，只需要遍历相关的子集。这样做的好处是，这种基于delta的设计在实现上稍显困难。*

## Rust 实现

在上一章中，我们克隆了一个初始项目模板。 我们现在将修改该项目模板。

让我们首先从 `wasm-game-of-life/src/lib.rs` 中删除 `alert` 导入和 `greet` 函数，并将它们替换为单元格的类型定义： 

```rust
#[wasm_bindgen]
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Cell {
    Dead = 0,
    Alive = 1,
}
```

重要的是我们有`#[repr(u8)]`，这样每个单元格都表示为一个字节。 同样重要的是，`Dead` 变体为 `0`，`Alive` 变体为 `1`，以便我们可以轻松地通过加法计算单元格的活邻居。

接下来，让我们定义宇宙。 宇宙有宽度和高度，以及长度为 `width * height` 的单元向量。 

```rust
#[wasm_bindgen]
pub struct Universe {
    width: u32,
    height: u32,
    cells: Vec<Cell>,
}
```

为了访问给定行和列的单元格，我们将行和列转换为单元格向量的索引，如前所述：

```rust
impl Universe {
    fn get_index(&self, row: u32, column: u32) -> usize {
        (row * self.width + column) as usize
    }

    // ...
}
```

为了计算一个细胞的下一个状态，我们需要计算它的邻居有多少是活着的。 让我们编写一个 `live_neighbor_count` 方法来做到这一点！ 

```rust
impl Universe {
    // ...

    fn live_neighbor_count(&self, row: u32, column: u32) -> u8 {
        let mut count = 0;
        for delta_row in [self.height - 1, 0, 1].iter().cloned() {
            for delta_col in [self.width - 1, 0, 1].iter().cloned() {
                if delta_row == 0 && delta_col == 0 {
                    continue;
                }

                let neighbor_row = (row + delta_row) % self.height;
                let neighbor_col = (column + delta_col) % self.width;
                let idx = self.get_index(neighbor_row, neighbor_col);
                count += self.cells[idx] as u8;
            }
        }
        count
    }
}
```

`live_neighbor_count` 方法使用 deltas 和 modulo 来避免使用 `if` 对宇宙边缘进行特殊外壳。 当应用 `-1` 的增量时，我们*添加* `self.height - 1` 并让模数做它的事情，而不是尝试减去 `1`。 `row` 和 `column` 可以是 `0`，如果我们试图从它们中减去 `1`，就会出现一个无符号整数下溢。

现在我们拥有了从当前计算下一代所需的一切！ 游戏的每条规则都直接转换为“匹配”表达式的条件。 此外，因为我们希望 JavaScript 控制滴答发生的时间，我们将把这个方法放在一个 `#[wasm_bindgen]` 块中，以便它暴露给 JavaScript。 

```rust
/// Public methods, exported to JavaScript.
#[wasm_bindgen]
impl Universe {
    pub fn tick(&mut self) {
        let mut next = self.cells.clone();

        for row in 0..self.height {
            for col in 0..self.width {
                let idx = self.get_index(row, col);
                let cell = self.cells[idx];
                let live_neighbors = self.live_neighbor_count(row, col);

                let next_cell = match (cell, live_neighbors) {
                    // Rule 1: Any live cell with fewer than two live neighbours
                    // dies, as if caused by underpopulation.
                    (Cell::Alive, x) if x < 2 => Cell::Dead,
                    // Rule 2: Any live cell with two or three live neighbours
                    // lives on to the next generation.
                    (Cell::Alive, 2) | (Cell::Alive, 3) => Cell::Alive,
                    // Rule 3: Any live cell with more than three live
                    // neighbours dies, as if by overpopulation.
                    (Cell::Alive, x) if x > 3 => Cell::Dead,
                    // Rule 4: Any dead cell with exactly three live neighbours
                    // becomes a live cell, as if by reproduction.
                    (Cell::Dead, 3) => Cell::Alive,
                    // All other cells remain in the same state.
                    (otherwise, _) => otherwise,
                };

                next[idx] = next_cell;
            }
        }

        self.cells = next;
    }

    // ...
}
```

So far, the state of the universe is represented as a vector of cells. To make
this human readable, let's implement a basic text renderer. The idea is to write
the universe line by line as text, and for each cell that is alive, print the
Unicode character `◼` ("black medium square"). For dead cells, we'll print `◻`
(a "white medium square").

By implementing the [`Display`] trait from Rust's standard library, we can add a
way to format a structure in a user-facing manner. This will also automatically
give us a [`to_string`] method.

[`Display`]: https://doc.rust-lang.org/1.25.0/std/fmt/trait.Display.html
[`to_string`]: https://doc.rust-lang.org/1.25.0/std/string/trait.ToString.html

```rust
use std::fmt;

impl fmt::Display for Universe {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        for line in self.cells.as_slice().chunks(self.width as usize) {
            for &cell in line {
                let symbol = if cell == Cell::Dead { '◻' } else { '◼' };
                write!(f, "{}", symbol)?;
            }
            write!(f, "\n")?;
        }

        Ok(())
    }
}
```

Finally, we define a constructor that initializes the universe with an
interesting pattern of live and dead cells, as well as a `render` method:

```rust
/// Public methods, exported to JavaScript.
#[wasm_bindgen]
impl Universe {
    // ...

    pub fn new() -> Universe {
        let width = 64;
        let height = 64;

        let cells = (0..width * height)
            .map(|i| {
                if i % 2 == 0 || i % 7 == 0 {
                    Cell::Alive
                } else {
                    Cell::Dead
                }
            })
            .collect();

        Universe {
            width,
            height,
            cells,
        }
    }

    pub fn render(&self) -> String {
        self.to_string()
    }
}
```

With that, the Rust half of our Game of Life implementation is complete!

Recompile it to WebAssembly by running `wasm-pack build` within the
`wasm-game-of-life` directory.

## Rendering with JavaScript

First, let's add a `<pre>` element to `wasm-game-of-life/www/index.html` to
render the universe into, just above the `<script>` tag:

```html
<body>
  <pre id="game-of-life-canvas"></pre>
  <script src="./bootstrap.js"></script>
</body>
```

Additionally, we want the `<pre>` centered in the middle of the Web page. We can
use CSS flex boxes to accomplish this task. Add the following `<style>` tag
inside `wasm-game-of-life/www/index.html`'s `<head>`:

```html
<style>
  body {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
  }
</style>
```

At the top of `wasm-game-of-life/www/index.js`, let's fix our import to bring in
the `Universe` rather than the old `greet` function:

```js
import { Universe } from "wasm-game-of-life";
```

Also, let's get that `<pre>` element we just added and instantiate a new
universe:

```js
const pre = document.getElementById("game-of-life-canvas");
const universe = Universe.new();
```

The JavaScript runs in [a `requestAnimationFrame`
loop][requestAnimationFrame]. On each iteration, it draws the current universe
to the `<pre>`, and then calls `Universe::tick`.

[requestAnimationFrame]: https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame

```js
const renderLoop = () => {
  pre.textContent = universe.render();
  universe.tick();

  requestAnimationFrame(renderLoop);
};
```

To start the rendering process, all we have to do is make the initial call for
the first iteration of the rendering loop:

```js
requestAnimationFrame(renderLoop);
```

Make sure your development server is still running (run `npm run start` inside
`wasm-game-of-life/www`) and this is what
[http://localhost:8080/](http://localhost:8080/) should look like:

[![Screenshot of the Game of Life implementation with text rendering](../images/game-of-life/initial-game-of-life-pre.png)](../images/game-of-life/initial-game-of-life-pre.png)

## Rendering to Canvas Directly from Memory

Generating (and allocating) a `String` in Rust and then having `wasm-bindgen`
convert it to a valid JavaScript string makes unnecessary copies of the
universe's cells. As the JavaScript code already knows the width and
height of the universe, and can read WebAssembly's linear memory that make up
the cells directly, we'll modify the `render` method to return a pointer to the
start of the cells array.

Also, instead of rendering Unicode text, we'll switch to using the [Canvas
API]. We will use this design in the rest of the tutorial.

[Canvas API]: https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API

Inside `wasm-game-of-life/www/index.html`, let's replace the `<pre>` we added
earlier with a `<canvas>` we will render into (it too should be within the
`<body>`, before the `<script>` that loads our JavaScript):

```html
<body>
  <canvas id="game-of-life-canvas"></canvas>
  <script src='./bootstrap.js'></script>
</body>
```

To get the necessary information from the Rust implementation, we'll need to add
some more getter functions for a universe's width, height, and pointer to its
cells array. All of these are exposed to JavaScript as well. Make these
additions to `wasm-game-of-life/src/lib.rs`:

```rust
/// Public methods, exported to JavaScript.
#[wasm_bindgen]
impl Universe {
    // ...

    pub fn width(&self) -> u32 {
        self.width
    }

    pub fn height(&self) -> u32 {
        self.height
    }

    pub fn cells(&self) -> *const Cell {
        self.cells.as_ptr()
    }
}
```

Next, in `wasm-game-of-life/www/index.js`, let's also import `Cell` from
`wasm-game-of-life`, and define some constants that we will use when rendering
to the canvas:

```js
import { Universe, Cell } from "wasm-game-of-life";

const CELL_SIZE = 5; // px
const GRID_COLOR = "#CCCCCC";
const DEAD_COLOR = "#FFFFFF";
const ALIVE_COLOR = "#000000";
```

Now, let's rewrite the rest of this JavaScript code to no longer write to the
`<pre>`'s `textContent` but instead draw to the `<canvas>`:

```js
// Construct the universe, and get its width and height.
const universe = Universe.new();
const width = universe.width();
const height = universe.height();

// Give the canvas room for all of our cells and a 1px border
// around each of them.
const canvas = document.getElementById("game-of-life-canvas");
canvas.height = (CELL_SIZE + 1) * height + 1;
canvas.width = (CELL_SIZE + 1) * width + 1;

const ctx = canvas.getContext('2d');

const renderLoop = () => {
  universe.tick();

  drawGrid();
  drawCells();

  requestAnimationFrame(renderLoop);
};
```

To draw the grid between cells, we draw a set of equally-spaced horizontal
lines, and a set of equally-spaced vertical lines. These lines criss-cross to
form the grid.

```js
const drawGrid = () => {
  ctx.beginPath();
  ctx.strokeStyle = GRID_COLOR;

  // Vertical lines.
  for (let i = 0; i <= width; i++) {
    ctx.moveTo(i * (CELL_SIZE + 1) + 1, 0);
    ctx.lineTo(i * (CELL_SIZE + 1) + 1, (CELL_SIZE + 1) * height + 1);
  }

  // Horizontal lines.
  for (let j = 0; j <= height; j++) {
    ctx.moveTo(0,                           j * (CELL_SIZE + 1) + 1);
    ctx.lineTo((CELL_SIZE + 1) * width + 1, j * (CELL_SIZE + 1) + 1);
  }

  ctx.stroke();
};
```

We can directly access WebAssembly's linear memory via `memory`, which is
defined in the raw wasm module `wasm_game_of_life_bg`. To draw the cells, we
get a pointer to the universe's cells, construct a `Uint8Array` overlaying the
cells buffer, iterate over each cell, and draw a white or black rectangle
depending on whether the cell is dead or alive, respectively. By working with
pointers and overlays, we avoid copying the cells across the boundary on every
tick.

```js
// Import the WebAssembly memory at the top of the file.
import { memory } from "wasm-game-of-life/wasm_game_of_life_bg";

// ...

const getIndex = (row, column) => {
  return row * width + column;
};

const drawCells = () => {
  const cellsPtr = universe.cells();
  const cells = new Uint8Array(memory.buffer, cellsPtr, width * height);

  ctx.beginPath();

  for (let row = 0; row < height; row++) {
    for (let col = 0; col < width; col++) {
      const idx = getIndex(row, col);

      ctx.fillStyle = cells[idx] === Cell.Dead
        ? DEAD_COLOR
        : ALIVE_COLOR;

      ctx.fillRect(
        col * (CELL_SIZE + 1) + 1,
        row * (CELL_SIZE + 1) + 1,
        CELL_SIZE,
        CELL_SIZE
      );
    }
  }

  ctx.stroke();
};
```

To start the rendering process, we'll use the same code as above to start the
first iteration of the rendering loop:

```js
drawGrid();
drawCells();
requestAnimationFrame(renderLoop);
```

Note that we call `drawGrid()` and `drawCells()` here _before_ we call
`requestAnimationFrame()`. The reason we do this is so that the _initial_ state
of the universe is drawn before we make modifications. If we instead simply
called `requestAnimationFrame(renderLoop)`, we'd end up with a situation where
the first frame that was drawn would actually be _after_ the first call to
`universe.tick()`, which is the second "tick" of the life of these cells.

## It Works!

Rebuild the WebAssembly and bindings glue by running this command from within
the root `wasm-game-of-life` directory:

```
wasm-pack build
```

Make sure your development server is still running. If it isn't, start it again
from within the `wasm-game-of-life/www` directory:

```
npm run start
```

If you refresh [http://localhost:8080/](http://localhost:8080/), you should be
greeted with an exciting display of life!

[![Screenshot of the Game of Life implementation](../images/game-of-life/initial-game-of-life.png)](../images/game-of-life/initial-game-of-life.png)

As an aside, there is also a really neat algorithm for implementing the Game of
Life called [hashlife](https://en.wikipedia.org/wiki/Hashlife). It uses
aggressive memoizing and can actually get *exponentially faster* to compute
future generations the longer it runs! Given that, you might be wondering why we
didn't implement hashlife in this tutorial. It is out of scope for this text,
where we are focusing on Rust and WebAssembly integration, but we highly
encourage you to go learn about hashlife on your own!

## Exercises

* Initialize the universe with a single space ship.

* Instead of hard-coding the initial universe, generate a random one, where each
  cell has a fifty-fifty chance of being alive or dead.

  *Hint: use [the `js-sys` crate](https://crates.io/crates/js-sys) to import
  [the `Math.random` JavaScript
  function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random).*

  <details>
    <summary>Answer</summary>
    *First, add `js-sys` as a dependency in `wasm-game-of-life/Cargo.toml`:*

    ```toml
    # ...
    [dependencies]
    js-sys = "0.3"
    # ...
    ```

    *Then, use the `js_sys::Math::random` function to flip a coin:*

    ```rust
    extern crate js_sys;

    // ...

    if js_sys::Math::random() < 0.5 {
        // Alive...
    } else {
        // Dead...
    }
    ```
  </details>

* Representing each cell with a byte makes iterating over cells easy, but it
  comes at the cost of wasting memory. Each byte is eight bits, but we only
  require a single bit to represent whether each cell is alive or dead. Refactor
  the data representation so that each cell uses only a single bit of space.

  <details>
    <summary>Answer</summary>

    In Rust, you can use [the `fixedbitset` crate and its `FixedBitSet`
    type](https://crates.io/crates/fixedbitset) to represent cells instead of
    `Vec<Cell>`:

    ```rust
    // Make sure you also added the dependency to Cargo.toml!
    extern crate fixedbitset;
    use fixedbitset::FixedBitSet;

    // ...

    #[wasm_bindgen]
    pub struct Universe {
        width: u32,
        height: u32,
        cells: FixedBitSet,
    }
    ```

    The Universe constructor can be adjusted the following way:

    ```rust
    pub fn new() -> Universe {
        let width = 64;
        let height = 64;

        let size = (width * height) as usize;
        let mut cells = FixedBitSet::with_capacity(size);

        for i in 0..size {
            cells.set(i, i % 2 == 0 || i % 7 == 0);
        }

        Universe {
            width,
            height,
            cells,
        }
    }
    ```

    To update a cell in the next tick of the universe, we use the `set` method
    of `FixedBitSet`:

    ```rust
    next.set(idx, match (cell, live_neighbors) {
        (true, x) if x < 2 => false,
        (true, 2) | (true, 3) => true,
        (true, x) if x > 3 => false,
        (false, 3) => true,
        (otherwise, _) => otherwise
    });
    ```

    To pass a pointer to the start of the bits to JavaScript, you can convert
    the `FixedBitSet` to a slice and then convert the slice to a pointer:

    ```rust
    #[wasm_bindgen]
    impl Universe {
        // ...

        pub fn cells(&self) -> *const u32 {
            self.cells.as_slice().as_ptr()
        }
    }
    ```

    In JavaScript, constructing a `Uint8Array` from Wasm memory is the same as
    before, except that the length of the array is not `width * height` anymore,
    but `width * height / 8` since we have a cell per bit rather than per byte:

    ```js
    const cells = new Uint8Array(memory.buffer, cellsPtr, width * height / 8);
    ```

    Given an index and `Uint8Array`, you can determine whether the
    *n<sup>th</sup>* bit is set with the following function:

    ```js
    const bitIsSet = (n, arr) => {
      const byte = Math.floor(n / 8);
      const mask = 1 << (n % 8);
      return (arr[byte] & mask) === mask;
    };
    ```

    Given all that, the new version of `drawCells` looks like this:

    ```js
    const drawCells = () => {
      const cellsPtr = universe.cells();

      // This is updated!
      const cells = new Uint8Array(memory.buffer, cellsPtr, width * height / 8);

      ctx.beginPath();

      for (let row = 0; row < height; row++) {
        for (let col = 0; col < width; col++) {
          const idx = getIndex(row, col);

          // This is updated!
          ctx.fillStyle = bitIsSet(idx, cells)
            ? ALIVE_COLOR
            : DEAD_COLOR;

          ctx.fillRect(
            col * (CELL_SIZE + 1) + 1,
            row * (CELL_SIZE + 1) + 1,
            CELL_SIZE,
            CELL_SIZE
          );
        }
      }

      ctx.stroke();
    };
    ```

  </details>
