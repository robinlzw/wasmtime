# 介绍

[Wasmtime][github] 是由 [Bytecode Alliance][BA] 提供的一个独立运行时环境，用于 WebAssembly、WASI 和组件模型。

[WebAssembly]（缩写为 Wasm）是一种二进制指令格式，被设计为编程语言的可移植编译目标。Wasm 二进制文件通常具有 `.wasm` 文件扩展名。在本文档中，我们还将使用二进制文件的文本表示形式，它们具有 `.wat` 文件扩展名。

[WASI]（WebAssembly 系统接口）定义了接口，提供了一种安全且可移植的方式来访问类似操作系统的功能，如文件系统、网络、时钟和随机数。

[组件模型]是一种 Wasm 架构，提供了一种二进制格式，用于可移植的、跨语言的组合。更具体地说，它支持通过接口使组件能够相互通信。WASI 是根据组件模型接口定义的。

Wasmtime 在 [Web之外]运行 WebAssembly 代码，可以作为命令行实用程序或嵌入在更大应用程序中的库使用。它致力于成为：

- **快速**：Wasmtime 建立在优化的 [Cranelift] 代码生成器之上。
- **安全**：Wasmtime 的开发强烈关注正确性和安全性。
- **可配置**：Wasmtime 使用合理的默认设置，但也可以根据需要配置以提供更细粒度的控制，例如 CPU 和内存消耗。
- **符合标准**：Wasmtime 通过了官方 WebAssembly 测试套件，并且 Wasmtime 开发者与 WebAssembly 标准过程密切相关。

本文档旨在服务于多个目的，您将在其中找到：

* [如何从多种语言使用 Wasmtime](lang.md)
* [如何安装和使用 `wasmtime` CLI](cli.md)
* 关于 Wasmtime 中的[稳定性](stability.md)和[安全性](security.md)的信息。
* 关于 [为 Wasmtime 做出贡献](contributing.md)的文档。

... 更多！本指南的源代码[托管在 GitHub](https://github.com/bytecodealliance/wasmtime/tree/main/docs) 上，欢迎贡献！

[github]: https://github.com/bytecodealliance/wasmtime 
[BA]: https://bytecodealliance.org/ 
[Cranelift]: https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/README.md 
[WebAssembly]: https://webassembly.org/ 
[WASI]: https://wasi.dev 
[outside of the Web]: https://webassembly.org/docs/non-web/ 
[issue]: https://github.com/bytecodealliance/wasmtime/issues/new 
[The Component Model]: https://github.com/WebAssembly/component-model 

---

总结：
Wasmtime 是 Bytecode Alliance 提供的一个用于运行 WebAssembly、WASI 和组件模型的独立运行时环境。它旨在为编程语言提供一个可移植的编译目标，并在 Web 之外提供一种安全、快速且可配置的执行环境。Wasmtime 遵循 WebAssembly 标准，并通过官方测试套件，同时提供了丰富的文档和社区支持，鼓励开发者使用和贡献代码。Wasmtime 的设计考虑了跨语言的组件组合，并支持多种运行时环境，使其成为一个适用于各种应用场景的多功能工具。
