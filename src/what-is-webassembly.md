# 什么是WebAssembly?

WebAssembly(wasm)是一种简单的机器模型和可执行格式，有一个[广泛的规范][extensive specification]。它被设计成可移植、紧凑，并以或接近原生速度执行。


作为一种编程语言，WebAssembly是由两种格式组成的。表示相同的结构，只是方式不同而已：

1. `.wat` 文本格式（称为`wat`，表示 "**W**eb**A**ssembly **T**ext"）使用 [S-expressions]，与 Lisp 系列语言有一些相似之处  像 Scheme 和 Clojure。
2. `.wasm` 二进制格式是较低级的，目的是让 wasm 虚拟机直接使用。它在概念上类似于 ELF 和 Mach-O

作为参考，这里有一个阶乘函数，在 `wat`:

```
(module
  (func $fac (param f64) (result f64)
    local.get 0
    f64.const 1
    f64.lt
    if (result f64)
      f64.const 1
    else
      local.get 0
      local.get 0
      f64.const 1
      f64.sub
      call $fac
      f64.mul
    end)
  (export "fac" (func $fac)))
```

如果您对 `wasm` 文件的外观感到好奇，可以将 [wat2wasm demo] 与上述代码一起使用。 

## 线性内存

WebAssembly有一个非常简单的[内存模型][memory model]。一个wasm模块可以访问一个单一的 "线性内存"，这基本上是一个字节的平面阵列。这个[内存可以按页面大小（64K）的倍数增长。它不能被缩小。

## WebAssembly 只是用于 Web 吗？

尽管它目前在 JavaScript 和 Web 社区中普遍受到关注，但 wasm 对其主机环境不做任何假设。因此，推测 wasm 将成为一种 "可移植的可执行 "格式，在未来被用于各种情况下是有意义的。然而，截至*今天*，wasm 主要与JavaScript（JS）有关，而 JavaScrip t有很多种类（包括 Web 和[Node.js]上的）。

[memory model]: https://webassembly.github.io/spec/core/syntax/modules.html#syntax-mem
[memory can be grown]: https://webassembly.github.io/spec/core/syntax/instructions.html#syntax-instr-memory
[extensive specification]: https://webassembly.github.io/spec/
[value types]: https://webassembly.github.io/spec/core/syntax/types.html#value-types
[Node.js]: https://nodejs.org
[S-expressions]: https://en.wikipedia.org/wiki/S-expression
[wat2wasm demo]: https://webassembly.github.io/wabt/demo/wat2wasm/
