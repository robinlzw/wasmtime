欢迎来到 WASI！

WASI 是 WebAssembly 系统接口（WebAssembly System Interface）的缩写。它是由 [Wasmtime] 项目设计的 API，提供访问类似操作系统的功能，包括文件和文件系统、Berkeley 套接字、时钟和随机数，我们计划将这些功能标准化。

WASI 设计上独立于浏览器，因此它不依赖于 Web API 或 JavaScript，也不受与 JavaScript 兼容的需求限制。它还集成了基于能力的安全性，因此将 WebAssembly 的沙箱特性扩展到了 I/O。

有关更详细的背景信息，请查看 [WASI 概览](WASI-overview.md)，以及 [WASI 教程](WASI-tutorial.md)，后者展示了如何将各个部分组合在一起。

请注意，这里的一切都是原型，虽然很多东西都可以工作，但仍有许多缺失的功能和一些粗糙的边缘。例如，网络支持是不完整的。

## 我如何编写使用 WASI 的程序？

目前表现良好的两个工具链是 Rust 工具链和特别打包的 C/C++ 工具链。当然，我们希望其他工具链也能够实现 WASI！

### Rust

要安装支持 WASI 的 Rust 工具链，请参见 [在线指南部分](https://bytecodealliance.github.io/wasmtime/examples-rust-embed.html)。

### C/C++

要安装支持 WASI 的 C/C++ 工具链，请参见 [在线指南部分](https://bytecodealliance.github.io/wasmtime/lang-c.html)。

## 我如何运行使用 WASI 的程序？

目前的选择是 [Wasmtime] 和 [浏览器 polyfill]，尽管我们希望 WASI 能够在许多 wasm 虚拟机中实现。

### Wasmtime

[Wasmtime] 是一个非 Web WebAssembly 引擎，是 [Bytecode Alliance 项目](https://bytecodealliance.org) 的一部分。要构建它，下载代码并使用 `cargo build --release` 构建。它可以通过简单地运行 `wasmtime foo.wasm` 或 `cargo run --bin wasmtime foo.wasm` 来运行使用 WASI 的 wasm 程序。

### 浏览器 polyfill

源代码在 [这里](https://github.com/bjorn3/browser_wasi_shim)。

## 我在哪里可以了解更多？

除了 [WASI 概览](WASI-overview.md)，还可以查看各种 [WASI 文档](WASI-documents.md)。

---

总结：
WASI 是一个为 WebAssembly 设计的系统接口，它提供了操作系统级别的功能，并且具有独立的安全性模型。目前，Rust 和 C/C++ 工具链支持 WASI，可以使用 Wasmtime 运行时或浏览器 polyfill 在不同环境中运行 WASI 程序。尽管 WASI 仍在开发中，但它已经能够提供一些基本的系统功能，并且预计未来会有更多的功能加入。更多关于 WASI 的信息可以通过查看相关文档和指南获得。如果你需要访问特定的在线资源，但遇到了问题，可能是由于链接错误或网络问题，请检查链接的合法性并适当重试。
