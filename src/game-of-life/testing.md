# 实现康威的生命游戏

现在我们已经使用 JavaScript 在浏览器中实现了生命游戏渲染的 Rust 实现，让我们来谈谈测试我们的 Rust 生成的 WebAssembly 函数。

我们将测试我们的 `tick` 函数以确保它提供我们期望的输出。

接下来，我们将要在 `wasm_game_of_life/src/lib.rs` 文件中现有的 `impl Universe` 块中创建一些 setter 和 getter 函数。 我们将创建一个 `set_width` 和一个 `set_height` 函数，以便我们可以创建不同大小的 `Universe`。 

```rust
#[wasm_bindgen]
impl Universe { 
    // ...

    /// Set the width of the universe.
    ///
    /// Resets all cells to the dead state.
    pub fn set_width(&mut self, width: u32) {
        self.width = width;
        self.cells = (0..width * self.height).map(|_i| Cell::Dead).collect();
    }

    /// Set the height of the universe.
    ///
    /// Resets all cells to the dead state.
    pub fn set_height(&mut self, height: u32) {
        self.height = height;
        self.cells = (0..self.width * height).map(|_i| Cell::Dead).collect();
    }

}
```

我们将在我们的 `wasm_game_of_life/src/lib.rs` 文件中创建另一个不带 `#[wasm_bindgen]` 属性的 `impl Universe` 块。 有一些我们需要测试的函数我们不想暴露给我们的 JavaScript。 Rust 生成的 WebAssembly 函数不能返回借用的引用。 尝试使用属性编译 Rust 生成的 WebAssembly 并查看您得到的错误。

我们将编写 `get_cells` 的实现来获取 `Universe` 的 `cells` 的内容。 我们还将编写一个 `set_cells` 函数，以便我们可以将 `Universe` 的特定行和列中的 `cells` 设置为 `Alive`。

```rust
impl Universe {
    /// Get the dead and alive values of the entire universe.
    pub fn get_cells(&self) -> &[Cell] {
        &self.cells
    }

    /// Set cells to be alive in a universe by passing the row and column
    /// of each cell as an array.
    pub fn set_cells(&mut self, cells: &[(u32, u32)]) {
        for (row, col) in cells.iter().cloned() {
            let idx = self.get_index(row, col);
            self.cells[idx] = Cell::Alive;
        }
    }

}
```

现在我们将在 `wasm_game_of_life/tests/web.rs` 文件中创建我们的测试。

在我们这样做之前，文件中已经有一个工作测试。 您可以通过在 `wasm-game-of-life` 目录中运行 `wasm-pack test --chrome --headless` 来确认 Rust 生成的 WebAssembly 测试正在运行。 你还可以使用 `--firefox`、`--safari` 和 `--node` 选项在这些浏览器中测试你的代码。

在 `wasm_game_of_life/tests/web.rs` 文件中，我们需要导出 `wasm_game_of_life` crate 和 `Universe` 类型。

```rust
extern crate wasm_game_of_life;
use wasm_game_of_life::Universe;
```

在 `wasm_game_of_life/tests/web.rs` 文件中，我们将要创建一些飞船建造器功能。

我们需要一个用于我们的输入宇宙飞船，我们将调用 `tick` 函数，我们希望在一个刻度后我们将获得预期的宇宙飞船。 我们选择了要初始化为 `Alive` 的单元格，以在 `input_spaceship` 函数中创建我们的飞船。 `input_spaceship` 勾选后，`expected_spaceship` 函数中飞船的位置是手动计算的。 您可以自己确认一下，输入飞船的单元格在一个tick后与预期的飞船相同。

```rust
#[cfg(test)]
pub fn input_spaceship() -> Universe {
    let mut universe = Universe::new();
    universe.set_width(6);
    universe.set_height(6);
    universe.set_cells(&[(1,2), (2,3), (3,1), (3,2), (3,3)]);
    universe
}

#[cfg(test)]
pub fn expected_spaceship() -> Universe {
    let mut universe = Universe::new();
    universe.set_width(6);
    universe.set_height(6);
    universe.set_cells(&[(2,1), (2,3), (3,2), (3,3), (4,2)]);
    universe
}
```

现在我们将为我们的`test_tick` 函数编写实现。 首先，我们创建了一个 `input_spaceship()` 和 `expected_spaceship()` 的实例。 然后，我们在 `input_universe` 上调用 `tick`。 最后，我们使用 `assert_eq!` 宏调用 `get_cells()` 以确保 `input_universe` 和 `expected_universe` 具有相同的 `Cell` 数组值。 我们将 `#[wasm_bindgen_test]` 属性添加到我们的代码块中，以便我们可以测试 Rust 生成的 WebAssembly 代码并使用 `wasm-pack test` 来测试 WebAssembly 代码。

```rust
#[wasm_bindgen_test]
pub fn test_tick() {
    // Let's create a smaller Universe with a small spaceship to test!
    let mut input_universe = input_spaceship();

    // This is what our spaceship should look like
    // after one tick in our universe.
    let expected_universe = expected_spaceship();

    // Call `tick` and then see if the cells in the `Universe`s are the same.
    input_universe.tick();
    assert_eq!(&input_universe.get_cells(), &expected_universe.get_cells());
}
```
通过在 `wasm-game-of-life` 目录中运行 `wasm-pack test --firefox --headless` 进行测试。 

