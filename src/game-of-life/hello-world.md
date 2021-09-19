# Hello, World!

本节将告诉你如何构建和运行你的第一个 Rust 和 WebAssembly 的程序：一个提醒 "Hello, World!" 的网页。

在开始之前，请确保你已经遵循了[设置说明](setup.html)。

## 克隆项目模板

该项目模板预先配置了合理的默认值，因此你可以快速构建、集成和打包你的代码用于 Web。

用这个命令克隆项目模板:

```text
cargo generate --git https://github.com/rustwasm/wasm-pack-template
```

这将提示你新项目的名称。我们将使用 **"wasm-game-of-life"**.

```text
wasm-game-of-life
```

## 里面有什么

进入新的 `wasm-game-of-life` 项目

```
cd wasm-game-of-life
```

并让我们看看它的内容。

```text
wasm-game-of-life/
├── Cargo.toml
├── LICENSE_APACHE
├── LICENSE_MIT
├── README.md
└── src
    ├── lib.rs
    └── utils.rs
```

让我们详细看一下其中的几个文件。 

### `wasm-game-of-life/Cargo.toml`

`Cargo.toml` 文件为 `cargo`、Rust 的包管理器和构建工具指定依赖项和元数据。 这个预先配置了一个 `wasm-bindgen` 依赖项，一些我们稍后将深入研究的可选依赖项，以及正确初始化的 `crate-type` 以生成 `.wasm` 库。 

### `wasm-game-of-life/src/lib.rs`

`src/lib.rs` 文件是我们正在编译为 WebAssembly 的 Rust crate 的根目录。 它使用 `wasm-bindgen` 与 JavaScript 交互。 它导入了 `window.alert` JavaScript 函数，并导出了 `greet` Rust 函数，它会提醒一条问候消息。 

```rust
mod utils;

use wasm_bindgen::prelude::*;

// When the `wee_alloc` feature is enabled, use `wee_alloc` as the global
// allocator.
#[cfg(feature = "wee_alloc")]
#[global_allocator]
static ALLOC: wee_alloc::WeeAlloc = wee_alloc::WeeAlloc::INIT;

#[wasm_bindgen]
extern {
    fn alert(s: &str);
}

#[wasm_bindgen]
pub fn greet() {
    alert("Hello, wasm-game-of-life!");
}
```

### `wasm-game-of-life/src/utils.rs`

`src/utils.rs` 模块提供了一些通用的工具来让 Rust 编译成 WebAssembly 变得更容易。 我们将在本教程后面更详细地了解其中一些实用程序，例如当我们查看 [调试我们的 wasm 代码](debugging.html) 时，但我们现在可以忽略此文件。 

## 构建项目 

我们使用 `wasm-pack` 来编排以下构建步骤： 

* 确保我们有 Rust 1.30 或更新版本，并且通过 `rustup` 安装了 `wasm32-unknown-unknown` 目标，
* 通过 `cargo` 将我们的 Rust 源编译为 WebAssembly `.wasm` 二进制文件，
* 使用 `wasm-bindgen` 生成 JavaScript API，以便使用我们的 Rust 生成的 WebAssembly。 

要做到这一切，在项目目录内运行这个命令。

```
wasm-pack build
```

构建完成后，我们可以在 `pkg` 目录中找到它的工件，它应该有以下内容： 

```
pkg/
├── package.json
├── README.md
├── wasm_game_of_life_bg.wasm
├── wasm_game_of_life.d.ts
└── wasm_game_of_life.js
```

`README.md` 文件是从主项目复制的，但其他文件是全新的。 

### `wasm-game-of-life/pkg/wasm_game_of_life_bg.wasm`

`.wasm` 文件是由 Rust 编译器从我们的 Rust 源代码生成的 WebAssembly 二进制文件。 它包含我们所有 Rust 函数和数据的编译到 wasm 版本。 例如，它有一个导出的“问候”功能。 

### `wasm-game-of-life/pkg/wasm_game_of_life.js`

`.js` 文件由 `wasm-bindgen` 生成，包含用于将 DOM 和 JavaScript 函数导入 Rust 并将 WebAssembly 函数的良好 API 暴露给 JavaScript 的 JavaScript 胶水。 例如，有一个 JavaScript 的 `greet` 函数包装了从 WebAssembly 模块导出的 `greet` 函数。 现在，这种粘合剂并没有做太多事情，但是当我们开始在 wasm 和 JavaScript 之间来回传递更多有趣的值时，它将帮助引导这些值跨越边界。 

```js
import * as wasm from './wasm_game_of_life_bg';

// ...

export function greet() {
    return wasm.greet();
}
```

### `wasm-game-of-life/pkg/wasm_game_of_life.d.ts`

`.d.ts` 文件包含 JavaScript 胶水的 [TypeScript][] 类型声明。 如果您使用的是 TypeScript，您可以检查对 WebAssembly 函数的调用类型，并且您的 IDE 可以提供自动完成和建议！ 如果您不使用 TypeScript，则可以放心地忽略此文件。 

```typescript
export function greet(): void;
```

[TypeScript]: http://www.typescriptlang.org/

### `wasm-game-of-life/pkg/package.json`

[`package.json` 文件包含有关生成的 JavaScript 和 WebAssembly 包的元数据。][package.json] npm 和 JavaScript 捆绑器使用它来确定包之间的依赖关系、包名称、版本和一堆其他东西。 它帮助我们与 JavaScript 工具集成，并允许我们将我们的包发布到 npm。 

```json
{
  "name": "wasm-game-of-life",
  "collaborators": [
    "Your Name <your.email@example.com>"
  ],
  "description": null,
  "version": "0.1.0",
  "license": null,
  "repository": null,
  "files": [
    "wasm_game_of_life_bg.wasm",
    "wasm_game_of_life.d.ts"
  ],
  "main": "wasm_game_of_life.js",
  "types": "wasm_game_of_life.d.ts"
}
```

[package.json]: https://docs.npmjs.com/files/package.json

## 将其放入网页 

为了获取我们的 `wasm-game-of-life` 包并在网页中使用它，我们使用 [`create-wasm-app` JavaScript 项目模板][create-wasm-app]。 

[create-wasm-app]: https://github.com/rustwasm/create-wasm-app

在 `wasm-game-of-life` 目录中运行此命令： 

```
npm init wasm-app www
```

这是我们新的 `wasm-game-of-life/www` 子目录包含的内容： 

```
wasm-game-of-life/www/
├── bootstrap.js
├── index.html
├── index.js
├── LICENSE-APACHE
├── LICENSE-MIT
├── package.json
├── README.md
└── webpack.config.js
```

再一次，让我们仔细看看其中的一些文件。 

### `wasm-game-of-life/www/package.json`

这个 `package.json` 预先配置了 `webpack` 和 `webpack-dev-server` 依赖，以及对 `hello-wasm-pack` 的依赖，它是已发布到 npm 的 `wasm-pack-template` 包。 

### `wasm-game-of-life/www/webpack.config.js`

此文件配置 webpack 及其本地开发服务器。 它是预先配置的，你根本不需要调整它来让 webpack 和它的本地开发服务器工作。 

### `wasm-game-of-life/www/index.html`

这是网页的根 HTML 文件。 除了加载 `bootstrap.js `之外，它没有做太多事情，`bootstrap.js` 是一个非常薄的 `index.js` 包装器。 

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Hello wasm-pack!</title>
  </head>
  <body>
    <script src="./bootstrap.js"></script>
  </body>
</html>
```

### `wasm-game-of-life/www/index.js`

`index.js` 是我们网页的 JavaScript 的主要入口点。 它导入 `hello-wasm-pack` npm 包，其中包含默认的 `wasm-pack-template` 编译的 WebAssembly 和 JavaScript 胶水，然后调用 `hello-wasm-pack` 的 `greet` 函数。 

```js
import * as wasm from "hello-wasm-pack";

wasm.greet();
```

### 安装依赖项 

首先，通过在 `wasm-game-of-life/www` 子目录中运行 `npm install` 来确保本地开发服务器及其依赖项已安装： 

```text
npm install
```

这个命令只需要运行一次，就会安装`webpack` JavaScript bundler 和它的开发服务器。

> 请注意，使用 Rust 和 WebAssembly 不需要 `webpack`，
> 这里它只是我们为了方便而选择的打包器和开发服务器
> Parcel 和 Rollup 还应该支持将 WebAssembly 导入为 ECMAScript 模块
> 你也可以使用 Rust 和 WebAssembly [没有捆绑器][without a bundler] 如果你愿意！ 

[without a bundler]: https://rustwasm.github.io/docs/wasm-bindgen/examples/without-a-bundler.html

### 在 `www` 中使用我们本地的 `wasm-game-of-life` 包 

我们不想使用 npm 中的 `hello-wasm-pack` 包，而是想使用我们本地的 `wasm-game-of-life` 包。 这将使我们能够逐步开发我们的生命游戏程序。

打开 `wasm-game-of-life/www/package.json` 并在 `"devDependencies"` 旁边添加 `"dependencies"` 字段，包括一个 `"wasm-game-of-life": "file :../pkg"` 条目： 

```js
{
  // ...
  "dependencies": {                     // Add this three lines block!
    "wasm-game-of-life": "file:../pkg"
  },
  "devDependencies": {
    //...
  }
}
```

接下来，修改 `wasm-game-of-life/www/index.js` 以导入 `wasm-game-of-life` 而不是 `hello-wasm-pack` 包： 

```js
import * as wasm from "wasm-game-of-life";

wasm.greet();
```

由于我们声明了一个新的依赖项，我们需要安装它：

```text
npm install
```

我们的网页现在可以在本地提供服务了！ 

## 本地服务

接下来，为开发服务器打开一个新终端。 在新终端中运行服务器让我们让它在后台运行，同时不会阻止我们运行其他命令。 在新终端中，从 `wasm-game-of-life/www` 目录中运行以下命令： 

```
npm run start
```

将您的 Web 浏览器导航到 [http://localhost:8080/](http://localhost:8080/)，您应该会看到一条警告消息： 

[![Screenshot of the "Hello, wasm-game-of-life!" Web page alert](../images/game-of-life/hello-world.png)](../images/game-of-life/hello-world.png)


任何时候进行更改并希望它们反映在 [http://localhost:8080/](http://localhost:8080/) 上，只需在 `wasm-game-of-life` 目录中运行 `wasm-pack build` 命令.

Anytime you make changes and want them reflected on
[http://localhost:8080/](http://localhost:8080/), just re-run the `wasm-pack
build` command within the `wasm-game-of-life` directory.

## 练习 

* 修改 `wasm-game-of-life/src/lib.rs` 中的 `greet` 函数，采用 `name: &str` 参数来自定义警报消息，并将你的名字从内部传递给 `greet` 函数 `wasm-game-of-life/www/index.js`。 使用 `wasm-pack build` 重新构建 `.wasm` 二进制文件，然后在 Web 浏览器中刷新 [http://localhost:8080/](http://localhost:8080/)，你应该会看到一个自定义的问候语！ 

  <details>
    <summary>答案</summary>

    `wasm-game-of-life/src/lib.rs` 中的 `greet` 函数的新版本:

    ```rust
    #[wasm_bindgen]
    pub fn greet(name: &str) {
        alert(&format!("Hello, {}!", name));
    }
    ```

    `wasm-game-of-life/www/index.js` 中对 `greet` 的新调用：

    ```js
    wasm.greet("Your Name");
    ```

  </details>
