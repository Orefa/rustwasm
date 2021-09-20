# 项目模板

Rust 和 WebAssembly 工作组策划和维护各种项目模板，以帮助您启动新项目并开始运行。

## `wasm-pack-template`

[此模板][wasm-pack-template] 用于启动 Rust 和 WebAssembly 项目，与 [`wasm-pack`][wasm-pack] 一起使用。

使用 `cargo generate` 来克隆这个项目模板：

```
cargo install cargo-generate
cargo generate --git https://github.com/rustwasm/wasm-pack-template.git
```

## `create-wasm-app`

[此模板][create-wasm-app] 适用于使用来自 npm 的包的 JavaScript 项目，这些包是使用 [`wasm-pack`][wasm-pack] 从 Rust 创建的。

与 `npm init` 一起使用：

```
mkdir my-project
cd my-project/
npm init wasm-app
```

该模板通常与 `wasm-pack-template` 一起使用，其中 `wasm-pack-template` 项目通过 `npm link` 安装在本地，并作为 `create-wasm-app` 项目的依赖项引入。

## `rust-webpack-template`

[此模板][rust-webpack-template] 预配置了所有样板，用于将 Rust 编译为 WebAssembly，并使用 Webpack 的 [`rust-loader`][rust-loader] 将其直接挂接到 Webpack 构建管道中。

与 `npm init` 一起使用：

```
mkdir my-project
cd my-project/
npm init rust-webpack
```

[wasm-pack]: https://github.com/rustwasm/wasm-pack
[wasm-pack-template]: https://github.com/rustwasm/wasm-pack-template
[create-wasm-app]: https://github.com/rustwasm/create-wasm-app
[rust-webpack-template]: https://github.com/rustwasm/rust-webpack-template
[rust-loader]: https://github.com/wasm-tool/rust-loader/
