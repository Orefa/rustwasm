# 如何为通用Crate添加 WebAssembly 支持

本节适用于想要支持 WebAssembly 的通用 crate 作者。

## 也许你的 Crate 已经支持 WebAssembly！

查看有关 [什么样的东西可以使 WebAssembly 的通用 crate *不可移植*](./which-crates-work-with-wasm.html) 的信息。 如果您的 crate 没有这些东西，它可能已经支持 WebAssembly！

您始终可以通过为 WebAssembly 目标运行 `cargo build` 来检查： 

```
cargo build --target wasm32-unknown-unknown
```

如果该命令失败，那么您的 crate 现在不支持 WebAssembly。 如果它没有失败，那么你的箱子 * 可能 * 支持 WebAssembly。 您可以通过 [为 wasm 添加测试并在 CI 中运行这些测试](#维护对-webassembly-的持续支持)来 100% 确定它会（并继续这样做！） 

## 添加对 WebAssembly 的支持

### 避免直接执行 I/O

在 Web 上，I/O 始终是异步的，并且没有文件系统。 将 I/O 排除在您的库之外，让用户执行 I/O，然后将输入切片传递给您的库。

例如，重构这个：

```rust
use std::fs;
use std::path::Path;

pub fn parse_thing(path: &Path) -> Result<MyThing, MyError> {
    let contents = fs::read(path)?;
    // ...
}
```

进入这个：

```rust
pub fn parse_thing(contents: &[u8]) -> Result<MyThing, MyError> {
    // ...
}
```

### 添加 `wasm-bindgen` 作为依赖

如果您需要与外部世界交互（即您不能让图书馆消费者为您驱动该交互），那么您需要添加 `wasm-bindgen`（以及 `js-sys` 和 `web-sys`，如果 您需要它们）作为编译针对 WebAssembly 时的依赖项：

```toml
[target.'cfg(target_arch = "wasm32")'.dependencies]
wasm-bindgen = "0.2"
js-sys = "0.3"
web-sys = "0.3"
```

### 避免同步 I/O

如果您必须在库中执行 I/O，则它不能是同步的。 Web 上只有异步 I/O。 使用 [`futures` 包](https://crates.io/crates/futures) 和 [`wasm-bindgen-futures` 包]((https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen_futures/) 来管理异步 I/O。 如果您的库函数在某个未来类型 `F` 上是通用的，那么该未来可以通过 Web 上的 `fetch` 或通过操作系统提供的非阻塞 I/O 来实现。

```rust
pub fn do_stuff<F>(future: F) -> impl Future<Item = MyOtherThing>
where
    F: Future<Item = MyThing>,
{
    // ...
}
```

您还可以为 WebAssembly 和 Web 以及本机目标定义一个特征并实现它：

```rust
trait ReadMyThing {
    type F: Future<Item = MyThing>;
    fn read(&self) -> Self::F;
}

#[cfg(target_arch = "wasm32")]
struct WebReadMyThing {
    // ...
}

#[cfg(target_arch = "wasm32")]
impl ReadMyThing for WebReadMyThing {
    // ...
}

#[cfg(not(target_arch = "wasm32"))]
struct NativeReadMyThing {
    // ...
}

#[cfg(not(target_arch = "wasm32"))]
impl ReadMyThing for NativeReadMyThing {
    // ...
}
```

### 避免产生线程

Wasm 尚不支持线程（但 [实验工作正在进行](https://rustwasm.github.io/2018/10/24/multithreading-rust-and-wasm.html)），因此尝试在 wasm 会恐慌。

您可以使用`#[cfg(..)]`s 来启用线程和非线程代码路径，具体取决于目标是否为 WebAssembly：

```rust
#![cfg(target_arch = "wasm32")]
fn do_work() {
    // Do work with only this thread...
}

#![cfg(not(target_arch = "wasm32"))]
fn do_work() {
    use std::thread;

    // Spread work to helper threads....
    thread::spawn(|| {
        // ...
    });
}
```

另一种选择是从您的库中提取线程，并允许用户“自带线程”，类似于提取文件 I/O 并允许用户自带 I/O。 这具有与想要拥有自己的自定义线程池的应用程序配合良好的副作用。 

## 维护对 WebAssembly 的持续支持

### 在 CI 中构建 `wasm32-unknown-unknown` 

通过让您的 CI 脚本运行以下命令，确保在针对 WebAssembly 时编译不会失败：

```
rustup target add wasm32-unknown-unknown
cargo check --target wasm32-unknown-unknown
```

例如，您可以将其添加到 Travis CI 的 `.travis.yml` 配置中：

```yaml

matrix:
  include:
    - language: rust
      rust: stable
      name: "check wasm32 support"
      install: rustup target add wasm32-unknown-unknown
      script: cargo check --target wasm32-unknown-unknown
```

### 在 Node.js 和无头浏览器中进行测试 

您可以使用 `wasm-bindgen-test` 和 `wasm-pack test` 子命令在 Node.js 或无头浏览器中运行 wasmntests。 您甚至可以将这些测试集成到您的 CI 中。

[在此处了解有关测试 wasm 的更多信息。](https://rustwasm.github.io/wasm-bindgen/wasm-bindgen-test/index.html)
