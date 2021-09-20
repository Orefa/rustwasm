# JavaScript 互操作

## 导入和导出 JS 函数

### 从 Rust 方面 

在 JS 主机中使用 wasm 时，从 Rust 端导入和导出函数很简单：它的工作方式与 C 非常相似。

WebAssembly 模块声明了一系列导入，每个导入都有一个 *module name* 和一个 *import name*。 `extern { ... }` 块的模块名称可以使用 [`#[link(wasm_import_module)]`][wasm_import_module] 指定，目前它默认为“env”。

出口只有一个名称。 除了任何 `extern` 函数，WebAssembly 实例的默认线性内存被导出为“内存”。 

[wasm_import_module]: https://github.com/rust-lang/rust/issues/52090

```rust
// import a JS function called `foo` from the module `mod`
#[link(wasm_import_module = "mod")]
extern { fn foo(); }

// export a Rust function called `bar`
#[no_mangle]
pub extern fn bar() { /* ... */ }
```

由于 wasm 的值类型有限，这些函数必须仅对原始数字类型进行操作。

### 从JS端

在 JS 中，wasm 二进制文件变成了 ES6 模块。 它必须使用线性内存*实例化*，并具有一组匹配预期导入的 JS 函数。 实例化的详细信息可在 [MDN][instantiation] 上找到。 

[instantiation]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/instantiateStreaming

生成的 ES6 模块将包含从 Rust 导出的所有函数，现在可以作为 JS 函数使用。

[这里][hello world] 是整个设置的一个非常简单的示例。 

## 超越数字

在 JS 中使用 wasm 时，wasm 模块的内存和 JS 内存之间存在明显的分裂：

- 每个 wasm 模块都有一个线性内存（在本文档的顶部描述），在实例化期间进行初始化。 **JS 代码可以自由读写这块内存**。

- 相比之下，wasm 代码不能*直接*访问 JS 对象。

因此，复杂的互操作以两种主要方式发生：

- 将二进制数据复制到或复制到 wasm 内存。 例如，这是向 Rust 端提供拥有的 `String` 的一种方式。

- 建立一个显式的 JS 对象“堆”，然后给定“地址”。 这允许 wasm 代码间接引用 JS 对象（使用整数），并通过调用导入的 JS 函数对这些对象进行操作。

幸运的是，这个互操作的故事非常适合通过一个通用的“bindgen”风格的框架来处理：[wasm-bindgen]。 该框架可以编写自动映射到惯用 JS 函数的惯用 Rust 函数签名。

[wasm-bindgen]: https://github.com/rustwasm/wasm-bindgen

## 自定义部分

自定义部分允许将命名的任意数据嵌入到 wasm 模块中。 段数据在编译时设置并直接从 wasm 模块读取，不能在运行时修改。

在 Rust 中，自定义节是使用 `#[link_section]` 属性公开的静态数组（`[T; size]`）：

```rust
#[link_section = "hello"]
pub static SECTION: [u8; 24] = *b"This is a custom section";
```

这会在 wasm 文件中添加一个名为 `hello` 的自定义部分，rust 变量名称 `SECTION` 是任意的，更改它不会改变行为。 这里的内容是文本字节，但可以是任何任意数据。

自定义部分可以在 JS 端使用 [`WebAssembly.Module.customSections`] 函数读取，它接受一个 wasm 模块和部分名称作为参数，并返回一个 [`ArrayBuffer`] 数组。 可以使用相同的名称指定多个部分，在这种情况下，它们都将出现在此数组中。

```js
WebAssembly.compileStreaming(fetch("sections.wasm"))
.then(mod => {
  const sections = WebAssembly.Module.customSections(mod, "hello");

  const decoder = new TextDecoder();
  const text = decoder.decode(sections[0]);

  console.log(text); // -> "This is a custom section"
});
```

[`ArrayBuffer`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer
[`WebAssembly.Module.customSections`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Module/customSections
