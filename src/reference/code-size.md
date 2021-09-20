# 缩小`.wasm` 代码大小

本节将教你如何优化你的 `.wasm` 构建以减少代码大小，以及如何识别更改 Rust 源代码的机会，以便发出更少的 `.wasm` 代码。

## 为什么要关心代码大小？

当通过网络提供 `.wasm` 文件时，它越小，客户端下载它的速度就越快。更快的 `.wasm` 下载会导致更快的页面加载时间，从而带来更快乐的用户。

但是，重要的是要记住，尽管代码大小可能不是您感兴趣的最终指标，而是更模糊且难以衡量的指标，例如“首次交互时间”。虽然代码大小在这个衡量中扮演着重要的角色（如果你还没有所有的代码，就什么也做不了！）它不是唯一的因素。

WebAssembly 通常提供给使用 gzip 压缩的用户，因此您需要确保比较 gzip 大小的差异，以便通过网络传输时间。还要记住，WebAssembly 二进制格式非常适合 gzip 压缩，通常可以减少 50% 以上的大小。

此外，WebAssembly 的二进制格式针对非常快速的解析和处理进行了优化。现在的浏览器有“基线编译器”，它解析 WebAssembly 并以最快的速度发出已编译的代码，因为它可以通过网络传入。这意味着[如果你正在使用`instantiateStreaming`][hacks] Web 请求完成后，WebAssembly 模块可能已经准备就绪。另一方面，JavaScript 通常需要更长的时间，不仅要解析，还要跟上 JIT 编译等的速度。

最后，请记住，WebAssembly 在执行速度方面也比 JavaScript 优化得多。您需要确保测量 JavaScript 和 WebAssembly 之间的运行时比较，以将其考虑到代码大小的重要性。

如果你的 `.wasm` 文件比预期的大，基本上不要立即沮丧！代码大小最终可能只是端到端故事中的众多因素之一。只看代码大小的 JavaScript 和 WebAssembly 之间的比较缺少树的森林。

[hacks]: https://hacks.mozilla.org/2018/01/making-webassembly-even-faster-firefoxs-new-streaming-and-tiering-compiler/

## 针对代码大小优化构建

有很多配置选项可以用来让`rustc`制作更小的`.wasm`二进制文件。在某些情况下，我们用较长的编译时间来换取较小的`.wasm'尺寸。在其他情况下，我们要用WebAssembly的运行速度来换取更小的代码大小。我们应该认识到每个选项的权衡，在用运行时间速度换取代码大小的情况下，要进行剖析和测量，以做出一个明智的决定，确定这种交易是否值得。

### 使用链接时间优化（LTO）进行编译

在`Cargo.toml`中，在`[profile.release]`部分添加`lto = true`：

```toml
[profile.release]
lto = true
```

这给了LLVM更多的机会来内联和修剪函数。这不仅会使`.wasm`更小，而且会使它在运行时更快！缺点是编译时间更长。缺点是编译时间会更长。

### 告诉LLVM对大小而不是速度进行优化

LLVM的优化通道被调整为提高速度，而不是默认的大小。我们可以通过修改`Cargo.toml`中的`[profile.release]`部分，将目标改为代码大小：

```toml
[profile.release]
opt-level = 's'
```

或者，更积极地优化尺寸，以进一步的潜在速度成本：

```toml
[profile.release]
opt-level = 'z'
```

请注意，令人惊讶的是，`opt-level = "s"` 有时会产生比 `opt-level = "z"` 更小的二进制文件。 经常测量！ 

### 使用 `wasm-opt` 工具

[Binaryen][] 工具包是一组特定于 WebAssembly 的编译器工具。 它比 LLVM 的 WebAssembly 后端走得更远，使用它的 `wasm-opt` 工具对 LLVM 生成的 `.wasm` 二进制文件进行后处理，通常可以再节省 15-20% 的代码大小。 它通常会同时产生运行时加速！

```bash
# Optimize for size.
wasm-opt -Os -o output.wasm input.wasm

# Optimize aggressively for size.
wasm-opt -Oz -o output.wasm input.wasm

# Optimize for speed.
wasm-opt -O -o output.wasm input.wasm

# Optimize aggressively for speed.
wasm-opt -O3 -o output.wasm input.wasm
```

[Binaryen]: https://github.com/WebAssembly/binaryen

### 关于调试信息的注意事项 

wasm 二进制文件大小的最大贡献者之一可能是调试信息和 wasm 二进制文件的“名称”部分。 然而，`wasm-pack` 工具默认会删除 debuginfo。 此外，除非还指定了 `-g`，否则 `wasm-opt` 会默认删除 `names` 部分。

这意味着，如果您按照上述步骤操作，则默认情况下，wasm 二进制文件中不应有 debuginfo 或 names 部分。 但是，如果您手动以其他方式在 wasm 二进制文件中保留此调试信息，请务必注意这一点！

## 尺寸分析

如果调整构建配置以优化代码大小并没有产生足够小的 `.wasm` 二进制文件，那么是时候进行一些分析以查看剩余代码大小的来源。

> ⚡ 就像我们如何让时间分析指导我们加速工作一样，我们希望让大小分析指导我们的代码大小缩减工作。
>  如果不这样做，您可能会浪费自己的时间！ 

### `twiggy` 代码大小分析器 

[`twiggy` 是一个代码大小分析器][twiggy] 支持 WebAssembly 作为输入。 它分析二进制的调用图来回答以下问题：

* 为什么这个函数首先包含在二进制文件中？

* 这个函数的*保留大小*是多少？ 即 如果我删除它以及删除后成为死代码的所有函数，会节省多少空间？ 

<style>
/* For whatever reason, the default mdbook fonts fonts break with the
   following box-drawing characters, hence the manual style. */
pre, code {
  font-family: "SFMono-Regular",Consolas,"Liberation Mono",Menlo,Courier,monospace;
}
</style>

```text
$ twiggy top -n 20 pkg/wasm_game_of_life_bg.wasm
 Shallow Bytes │ Shallow % │ Item
───────────────┼───────────┼────────────────────────────────────────────────────────────────────────────────────────
          9158 ┊    19.65% ┊ "function names" subsection
          3251 ┊     6.98% ┊ dlmalloc::dlmalloc::Dlmalloc::malloc::h632d10c184fef6e8
          2510 ┊     5.39% ┊ <str as core::fmt::Debug>::fmt::he0d87479d1c208ea
          1737 ┊     3.73% ┊ data[0]
          1574 ┊     3.38% ┊ data[3]
          1524 ┊     3.27% ┊ core::fmt::Formatter::pad::h6825605b326ea2c5
          1413 ┊     3.03% ┊ std::panicking::rust_panic_with_hook::h1d3660f2e339513d
          1200 ┊     2.57% ┊ core::fmt::Formatter::pad_integral::h06996c5859a57ced
          1131 ┊     2.43% ┊ core::str::slice_error_fail::h6da90c14857ae01b
          1051 ┊     2.26% ┊ core::fmt::write::h03ff8c7a2f3a9605
           931 ┊     2.00% ┊ data[4]
           864 ┊     1.85% ┊ dlmalloc::dlmalloc::Dlmalloc::free::h27b781e3b06bdb05
           841 ┊     1.80% ┊ <char as core::fmt::Debug>::fmt::h07742d9f4a8c56f2
           813 ┊     1.74% ┊ __rust_realloc
           708 ┊     1.52% ┊ core::slice::memchr::memchr::h6243a1b2885fdb85
           678 ┊     1.45% ┊ <core::fmt::builders::PadAdapter<'a> as core::fmt::Write>::write_str::h96b72fb7457d3062
           631 ┊     1.35% ┊ universe_tick
           631 ┊     1.35% ┊ dlmalloc::dlmalloc::Dlmalloc::dispose_chunk::hae6c5c8634e575b8
           514 ┊     1.10% ┊ std::panicking::default_hook::{{closure}}::hfae0c204085471d5
           503 ┊     1.08% ┊ <&'a T as core::fmt::Debug>::fmt::hba207e4f7abaece6
```

[twiggy]: https://github.com/rustwasm/twiggy

### 手动检查 LLVM-IR

在 LLVM 生成 WebAssembly 之前，LLVM-IR 是编译器工具链中的最终中间表示。 因此，它与最终发出的 WebAssembly 非常相似。 更多的 LLVM-IR 通常意味着更多的 `.wasm` 大小，如果一个函数占用 LLVM-IR 的 25%，那么它通常会占用 `.wasm` 的 25%。 虽然这些数字仅适用于一般情况，但 LLVM-IR 具有在 `.wasm` 中不存在的关键信息（因为 WebAssembly 缺乏像 DWARF 这样的调试格式）：哪些子例程被内联到给定的函数中。

您可以使用以下 `cargo` 命令生成 LLVM-IR：

```
cargo rustc --release -- --emit llvm-ir
```

然后，您可以使用 `find` 在 `cargo` 的 `target` 目录中定位包含 LLVM-IR 的 `.ll` 文件：

```
find target/release -type f -name '*.ll'
```

#### 参考

* [LLVM 语言参考手册](https://llvm.org/docs/LangRef.html)

## 更具侵入性的工具和技术

调整构建配置以获得更小的 `.wasm` 二进制文件是非常容易的。然而，当你需要走更多的路时，你要准备使用更多的侵入性技术，比如重写源代码以避免臃肿。下面是一系列 "动手 "的技术，你可以应用这些技术来获得更小的代码大小。

### 避免字符串格式化

`format!`, `to_string`, 等等...会带来大量的代码膨胀。如果可能的话，只在调试模式下进行字符串格式化，而在发布模式下使用静态字符串。

### 避免 Panicking

这绝对是说起来容易做起来难，但是像 `twiggy` 和手动检查 LLVM-IR 这样的工具可以帮助你找出哪些函数是恐慌的。

Panics 并不总是以`panic!()` 宏调用的形式出现。 它们隐含地来自许多结构，例如：

* 索引切片在越界索引时发生 panics：`my_slice[i]`

* 如果除数为零，除法会 panic：`dividend / divisor`

* 解开`Option` 或`Result`：`opt.unwrap()` 或`res.unwrap()`

前两个可以翻译成第三个。 索引可以替换为容易出错的 `my_slice.get(i)` 操作。 除法可以用`checked_div` 调用代替。 现在我们只有一个案例要处理。

在不惊慌的情况下展开 `Option` 或 `Result` 有两种方式：安全和不安全。

安全的方法是在遇到 `None` 或 `Error` 时使用 `abort` 而不是 panicking： 

```rust
#[inline]
pub fn unwrap_abort<T>(o: Option<T>) -> T {
    use std::process;
    match o {
        Some(t) => t,
        None => process::abort(),
    }
}
```

最终，无论如何，恐慌都会转化为 `wasm32-unknown-unknown` 中的中止，所以这会给你相同的行为，但没有代码膨胀。

或者，[`unreachable` crate][unreachable] 为 `Option` 和 `Result` 提供了一个不安全的 [`unchecked_unwrap` 扩展方法][unchecked_unwrap]，它告诉 Rust 编译器*假设*`Option` 是`Some ` 或 `Result` 是 `Ok`。 如果该假设不成立，会发生什么情况是未定义的行为。 当您 110% *知道*假设成立时，您真的只想使用这种不安全的方法，而编译器不够聪明，无法看到它。 即使你走这条路，你应该有一个仍然进行检查的调试构建配置，并且只在发布构建中使用未经检查的操作。

[unreachable]: https://crates.io/crates/unreachable
[unchecked_unwrap]: https://docs.rs/unreachable/1.0.0/unreachable/trait.UncheckedOptionExt.html#tymethod.unchecked_unwrap

### 避免分配或切换到 `wee_alloc` 

Rust 的 WebAssembly 的默认分配器是 `dlmalloc` 到 Rust 的端口。 它的重量约为 10 KB。 如果您可以完全避免动态分配，那么您应该能够摆脱这十 KB。

完全避免动态分配可能非常困难。 但是从热代码路径中删除分配通常要容易得多（并且通常也有助于使这些热代码路径更快）。 在这些情况下，[用 `wee_alloc` 替换默认的全局分配器][wee_alloc] 应该为您节省这十 KB 中的大部分（但不是全部）。 `wee_alloc` 是一个分配器，专为您需要*某种*类型的分配器但不需要特别快的分配器的情况而设计，并且很乐意用分配速度换取较小的代码大小。

[wee_alloc]: https://github.com/rustwasm/wee_alloc

### 使用特征对象而不是泛型类型参数

当您创建使用类型参数的泛型函数时，如下所示：

```rust
fn whatever<T: MyTrait>(t: T) { ... }
```

然后 `rustc` 和 LLVM 将为使用该函数的每个 `T` 类型创建该函数的新副本。 这为基于每个副本所使用的特定“T”提供了许多编译器优化机会，但这些副本在代码大小方面加起来很快。

如果你使用 trait 对象而不是类型参数，像这样：

```rust
fn whatever(t: Box<MyTrait>) { ... }
// or
fn whatever(t: &MyTrait) { ... }
// etc...
```

然后使用通过虚拟调用的动态调度，并且在 `.wasm` 中只发出该函数的一个版本。 缺点是失去了编译器优化机会以及间接、动态调度的函数调用的额外成本。

### 使用 wasm-snip 工具

[`wasm-snip`用`unreachable`指令替换WebAssembly函数的主体。][snip]这是一个相当重的、钝的锤子，用于那些看起来像钉子的函数，如果你足够努力地眯着眼睛。

也许你知道某些函数在运行时不会被调用，但编译器在编译时无法证明这一点？把它剪掉吧! 之后，再次运行 `wasm-opt` 并加上 `--dce` 标志，所有被剪掉的函数所调用的函数（这些函数在运行时也不可能被调用）也会被删除。

这个工具对于删除恐慌的基础结构特别有用，因为无论如何，恐慌最终都会转化为陷阱。

[snip]: https://github.com/fitzgen/wasm-snip
