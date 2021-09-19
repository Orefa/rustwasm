# 设置

本节描述了如何设置工具链，将 Rust 程序编译为 WebAssembly，并将其集成到 JavaScript 中。到 WebAssembly，并将其集成到 JavaScript 中。

## Rust 工具链

你将需要标准的 Rust 工具链，包括 `rustup`, `rustc`, 和 `cargo`.

[按照这些说明来安装Rust工具链。][rust-install]

Rust 和 WebAssembly 的经验是乘着 Rust 发布的列车到了稳定期! 这意味着我们不需要任何实验性功能标志。然而，我们确实 需要 Rust 1.30 或更新版本。

## `wasm-pack`

`wasm-pack` 是您构建、测试和发布 Rust 生成的 WebAssembly 的一站式商店。 

[Get `wasm-pack` here!][wasm-pack-install]

## `cargo-generate`

[`cargo-generate` 通过利用预先存在的 git 存储库作为模板，帮助您快速启动并运行新的 Rust 项目。][cargo-generate]

使用如下命令安装 `cargo-generate`:

```
cargo install cargo-generate
```

## `npm`
`npm` 是 JavaScript 的包管理器。 我们将使用它来安装和运行 JavaScript 打包器和开发服务器。 在教程结束时，我们将把我们编译的 `.wasm` 发布到 `npm` 仓库。 

[按照这些说明进行安装 `npm`.][npm-install]

如果您已经安装了 `npm`，请使用以下命令确保它是最新的： 

```
npm install npm@latest -g
```

[rust-install]: https://www.rust-lang.org/tools/install
[npm-install]: https://www.npmjs.com/get-npm
[wasm-pack]: https://github.com/rustwasm/wasm-pack
[cargo-generate]: https://github.com/ashleygwilliams/cargo-generate
[wasm-pack-install]: https://rustwasm.github.io/wasm-pack/installer/
