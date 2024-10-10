Cranelift 是一个低级可重定向的代码生成器，它将与目标无关的中间表示（IR）翻译为可执行的机器码。该项目由 Bytecode Alliance 维护，主要用于为 WebAssembly 生成代码，同时具备多平台后端支持（如 x86-64、aarch64 等），并在 Wasmtime 中得到广泛使用。Cranelift 的代码生成质量接近 V8 等浏览器 JIT 引擎，同时编译速度比基于 LLVM 的引擎快一个数量级。Cranelift 的设计灵活，也可用于非 WebAssembly 场景。

总结：Cranelift 是一个高效、可重定向的代码生成器，主要用于 WebAssembly，但其设计也适用于其他应用。它比 LLVM 更快，但在性能上略有损失，是为需要快速编译的场景而设计的。

---


# Cranelift 代码生成器
========================

**一个 [Bytecode Alliance][BA] 项目**

[官网](https://cranelift.dev/)

Cranelift 是一个低级别的可重定向代码生成器。它将目标独立的中间表示转换成可执行的机器代码。

[BA]: https://bytecodealliance.org/ 
[![构建状态](https://github.com/bytecodealliance/wasmtime/workflows/CI/badge.svg)](https://github.com/bytecodealliance/wasmtime/actions) 
[![聊天](https://img.shields.io/badge/chat-zulip-brightgreen.svg)](https://bytecodealliance.zulipchat.com/#narrow/stream/217117-cranelift/topic/general) 
![最低 rustc 1.37](https://img.shields.io/badge/rustc-1.37+-green.svg) 
[![文档状态](https://docs.rs/cranelift/badge.svg)](https://docs.rs/cranelift) 

更多信息，请查看 [文档](docs/index.md)。

关于如何使用 JIT 的示例，请参阅 [JIT Demo]，它实现了一个玩具语言。

[JIT Demo]: https://github.com/bytecodealliance/cranelift-jit-demo 

关于如何使用 Cranelift 运行 WebAssembly 代码的示例，请参阅 [Wasmtime]，它实现了一个独立的、可嵌入的 VM，使用了 Cranelift。

[Wasmtime]: https://github.com/bytecodealliance/wasmtime 

状态
------

Cranelift 目前支持足够的功能来运行各种程序，包括执行 WebAssembly（MVP 和各种扩展，如 SIMD）所需的所有功能，尽管它需要在一个外部 WebAssembly 嵌入中使用，如 Wasmtime，才能成为完整的 WebAssembly 实现的一部分。它也可以作为非 WebAssembly 使用案例的后端：例如，有人正在努力使用 Cranelift 构建一个 [Rust 编译器后端]。

Cranelift 已经准备好投入生产，并在多个地方投入生产，所有这些都在 Wasmtime 的背景下。它与 V8 和可执行 Wasm 规范进行了差异比较的仔细模糊测试，并且寄存器分配器也单独进行了符号验证的模糊测试。有一个积极的 effort 正式验证 Cranelift 的指令选择后端。我们非常重视安全性，并有 [Bytecode Alliance 的安全政策]。

Cranelift 有四个后端：x86-64、aarch64（即 ARM64）、s390x（即 IBM Z）和 riscv64。所有后端都完全支持 Wasm MVP 的功能，x86-64 和 aarch64 也完全支持 SIMD。在 x86-64 上，Cranelift 支持许多平台上使用的 System V AMD64 ABI 调用约定和 Windows x64 调用约定。在 aarch64 上，Cranelift 支持标准 Linux 调用约定，并且也有针对 macOS（即 M1 / Apple Silicon）的特定支持。

Cranelift 的代码质量在竞争力方面与浏览器 JIT 引擎的优化层相当。[最近的一篇论文] 包括了由 Wasmtime 驱动的 Cranelift 与 V8 和基于 LLVM 的 Wasm 引擎 WAVM 的第三方基准测试（图 22）。Cranelift 生成的代码速度大约比 V8（TurboFan）慢 2%，比 WAVM（LLVM）慢 14%。在同一篇论文中，其编译速度被测量为大约比 WAVM（LLVM）快一个数量级。我们继续努力改进这两个指标。

[Rust 编译器后端]: https://github.com/bjorn3/rustc_codegen_cranelift 
[安全政策]: https://bytecodealliance.org/security 
[最近的一篇论文]: https://arxiv.org/abs/2011.13127 

核心代码生成包的依赖性很小，并且被仔细编写以处理恶意或任意编译器输入：特别是，它们不使用调用栈递归。

Cranelift 对堆界限检查、表界限检查和间接分支界限检查进行了一些基本的 Spectre 攻击缓解措施；见 [#1032] 了解更多。

[#1032]: https://github.com/bytecodealliance/wasmtime/issues/1032 

Cranelift 的 API 尚未被认为是稳定的，尽管我们确实遵循了带有次版本补丁发布的语义化版本控制（semver）。

Cranelift 通常需要最新的稳定 Rust 来构建，作为政策，并且就是这样进行测试的，但我们可以在尽力而为的基础上纳入对旧版本 Rust 的编译修复。

贡献
------------

如果你对贡献 Cranelift 感兴趣：谢谢你！我们有一个[贡献指南]，它将帮助你参与 Cranelift 项目。

[贡献指南]: https://bytecodealliance.github.io/wasmtime/contributing.html 

计划用途
------------

Cranelift 被设计为 WebAssembly 的代码生成器，但它足够通用，也可以在其他地方使用。最初计划的用途影响了它的设计：

- [Wasmtime 非 Web wasm 引擎](https://github.com/bytecodealliance/wasmtime)。
- [Rust 编译器的调试构建后端](rustc.md)。
- Firefox 中 SpiderMonkey 引擎的 WebAssembly 编译器（目前不再计划；SpiderMonkey 团队可能会在未来重新评估）。
- Firefox 中 IonMonkey JavaScript JIT 编译器的后端（目前不再计划；SpiderMonkey 团队可能会在未来重新评估）。

构建 Cranelift
------------------

Cranelift 使用 [传统的 Cargo 构建过程](https://doc.rust-lang.org/cargo/guide/working-on-an-existing-project.html)。

Cranelift 由一系列包组成，并使用 [Cargo 工作区](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html)，因此对于一些 cargo 命令，如 `cargo test`，需要使用 `--all` 来告诉 cargo 访问所有包。

顶层的 `test-all.sh` 是一个脚本，它运行所有的 cargo 测试，并执行代码格式、lint 和文档检查。

<details>
<summary>日志配置</summary>

Cranelift 使用 `log` 包在不同级别记录消息。它没有指定最大的日志级别，因此嵌入者可以选择它应该是什么；然而，这可能会影响 Cranelift 的代码大小。你可以使用 `log` 功能来降低最大日志级别。例如，如果你想在发布模式下将日志级别限制为 `warn` 消息及以上：

```rust
[dependency.log]
...
features = ["release_max_level_warn"]
```
</details>

编辑器支持
--------------

用于处理 Cranelift IR (clif) 文件的编辑器支持：

 - Vim: https://github.com/bytecodealliance/cranelift.vim 


### 解释和总结

Cranelift 是 Bytecode Alliance 支持的一个项目，它是一个低级别的可重定向代码生成器，能够将独立于目标的中间表示转换成可执行的机器代码。该项目旨在为 WebAssembly 提供一个高效且安全的代码生成器，同时也适用于其他非 WebAssembly 的用例。

Cranelift 具有多个后端，包括 x86-64、aarch64、s390x 和 riscv64，支持 WebAssembly MVP 和 SIMD 功能。它在安全性方面进行了投资，包括对 Spectre 攻击的基本缓解措施，并且遵循语义化版本控制。

该项目的生产就绪性得到了确认，因为它在 Wasmtime 的背景下已经在多个地方投入生产。Cranelift 的性能与浏览器 JIT 引擎相当，尽管生成的代码速度略慢于 V8 和 LLVM，但其编译速度更快。

Cranelift 的 API 尚未稳定，但项目遵循语义化版本控制，并且鼓励开发者参与贡献。项目的设计初衷是为了满足 WebAssembly 以及 Rust 编译器调试构建后端等用途。

最后，Cranelift 使用传统的 Cargo 构建过程，并通过一个脚本来运行所有的测试和检查。对于希望参与项目的开发者，项目提供了详细的贡献指南和编辑器支持。
