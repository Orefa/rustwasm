# 发布到 npm

现在我们有了一个有效的、快速的、*和*小的 `wasm-game-of-life` 包，我们可以将它发布到 npm 以便其他 JavaScript 开发人员可以重用它，如果他们需要一个现成的游戏 生活执行。

## 先决条件

首先，[确保您有一个 npm 帐户](https://www.npmjs.com/signup)。

其次，通过运行以下命令确保您在本地登录到您的帐户： 

```
wasm-pack login
```

## 发布

通过在 `wasm-game-of-life` 目录中运行 `wasm-pack`，确保 `wasm-game-of-life/pkg` 构建是最新的： 

```
wasm-pack build
```

现在花点时间查看一下 `wasm-game-of-life/pkg` 的内容，这就是我们下一步要发布到 npm 的内容！

准备好后，运行 `wasm-pack publish` 将包上传到 npm： 

```
wasm-pack publish
```

这就是发布到 npm 所需的全部内容！

...除了其他人也完成了本教程，因此在 npm 上使用了 `wasm-game-of-life` 名称，最后一个命令可能不起作用。

打开 `wasm-game-of-life/Cargo.toml` 并将您的用户名添加到 `name` 的末尾，以独特的方式消除包的歧义：

```toml
[package]
name = "wasm-game-of-life-my-username"
```

然后，重建并再次发布：

```
wasm-pack build
wasm-pack publish
```

这次应该可以了！
