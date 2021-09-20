# 缩小`.wasm` 大小

对于我们通过网络发送给客户端的 `.wasm` 二进制文件，例如我们的 Game of Life Web 应用程序，我们希望关注代码大小。 我们的 `.wasm` 越小，我们的页面加载速度就越快，我们的用户就越开心。

## 我们可以通过构建配置获得我们的 Game of Life `.wasm` 二进制文件有多小？

[花点时间查看我们可以调整的构建配置选项以获得更小的 `.wasm` 代码大小。](../reference/code-size.html#针对代码大小优化构建)

使用默认的发布构建配置（没有调试符号），我们的 WebAssembly 二进制文件是 29,410 字节： 

```
$ wc -c pkg/wasm_game_of_life_bg.wasm
29410 pkg/wasm_game_of_life_bg.wasm
```

在启用 LTO、设置 `opt-level = "z"` 并运行 `wasm-opt -Oz` 后，生成的 `.wasm` 二进制文件缩小到只有 17,317 个字节： 

```
$ wc -c pkg/wasm_game_of_life_bg.wasm
17317 pkg/wasm_game_of_life_bg.wasm
```

如果我们用 `gzip` 压缩它（几乎每个 HTTP 服务器都会这样做），我们会减少到 9,045 字节！ 

```
$ gzip -9 < pkg/wasm_game_of_life_bg.wasm | wc -c
9045
```

## 练习

* 使用 [`wasm-snip` 工具](../reference/code-size.html#使用-wasm-snip-工具) 从我们的生命游戏的 `.wasm` 二进制文件中删除恐慌的基础设施功能。 它节省了多少字节？

* 使用和不使用 [`wee_alloc` 作为其全局分配器](https://github.com/rustwasm/wee_alloc) 构建我们的游戏生命箱。 我们克隆来启动这个项目的 `rustwasm/wasm-pack-template` 模板有一个“wee_alloc”货物特性，您可以通过将其添加到 `wasm-game-of-life/Cargo.toml` 的 `[features]` 部分中的 `default` 键来启用：

  ```toml
  [features]
  default = ["wee_alloc"]
  ```

  使用 `wee_alloc` 可以减少 `.wasm` 二进制文件的大小吗？ 

* 我们只实例化一个 `Universe`，因此我们可以导出操作单个 `static mut` 全局实例的操作，而不是提供构造函数。 如果这个全局实例也使用了前面章节中讨论过的双缓冲技术，我们可以使这些缓冲区也成为 `static mut` 全局变量。 这从我们的生命游戏实现中删除了所有动态分配，我们可以使它成为一个不包含分配器的 `#![no_std]` crate。 通过完全删除分配器依赖，从 `.wasm` 中删除了多少大小？ 
