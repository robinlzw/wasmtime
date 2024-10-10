这篇文章讨论了如何在Wasmtime中添加对新的WebAssembly提案的支持，并概述了使这些提案默认启用的标准。以下是文章的主要内容总结：

1. **添加对Wasm提案的支持**：
   - 需要更新多个crates以支持新的WebAssembly提案，包括`wasmparser`、`wat`、`wast`、`wasmprinter`、`wasm-encoder`和`wasm-smith`。
   - 在`wasmtime` crate中添加配置方法和命令行标志以启用新功能。
   - 在`build.rs`中启用规范测试，但暂时忽略它们。
   - 逐个解决并启用规范测试。
   - 在fuzz目标中启用提案，并添加示例到相关语料库。
   - 为特定提案编写自定义的fuzz目标、预言器和/或测试用例生成器。
   - 在`wasmtime` crate的API中公开提案的新功能。
   - 在C API中公开提案的新功能，并在其他语言的API中公开这些功能，例如Python、Go和.NET。
   - 在`wasmtime/docs/stability-wasm-proposals-support.md`中记录对提案的支持。

2. **默认启用提案支持的标准**：
   - 提案必须处于WebAssembly标准化过程的第四阶段或更高。
   - Wasmtime中的所有规范测试必须通过。
   - 没有开放的问题、设计顾虑或严重的已知错误。
   - 至少进行了一周的fuzz测试。
   - 我们确信fuzz测试完全覆盖了提案的功能。
   - 提案的功能在`wasmtime` crate的API和C API中被公开。
   - 提案的功能至少在一个其他语言的API中被公开。

3. **在WASI中添加组件功能**：
   - `cap-std`仓库包含了实现基于能力的Rust标准库的crates。一旦功能被添加到相关crates中，就可以通过将它们包含在[wasi crate](https://github.com/bytecodealliance/wasmtime/tree/main/crates/wasi)的preview2目录中来添加到wasmtime。
   - 目前，依赖preview2 ABI的WebAssembly模块不能直接被wasmtime命令执行。文章提供了测试这些更改的步骤，包括构建wasmtime、创建测试模块、构建[wasi-preview1-component-adapter](https://github.com/bytecodealliance/wasmtime/tree/main/crates/wasi-preview1-component-adapter)作为命令适配器、使用[wasm-tools](https://github.com/bytecodealliance/wasm-tools)将测试模块转换为组件，以及使用本地构建的wasmtime运行测试组件。

如果你需要访问上述链接以获取更多信息，但遇到了网络问题，这可能是由于链接本身的问题或网络连接问题。请检查链接的合法性，并在必要时重试。如果问题仍然存在，可能需要联系项目的维护者或查看项目的文档以获取更多帮助。
