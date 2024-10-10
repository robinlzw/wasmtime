# 平台支持

本页面旨在提供 Wasmtime 平台支持的高级概览以及 Wasmtime 的一些愿景。更详细的信息，请参阅有关[稳定性层级](./stability-tiers.md)的文档，该文档详细说明了 Wasmtime 在不同矩阵组合基础上支持的内容。

Wasmtime 致力于支持任何人想要运行 WebAssembly 的硬件。Wasmtime 的维护者自己支持一些“主要”平台，但可能需要移植工作来支持维护者不熟悉的平台。开箱即用的 Wasmtime 支持：

* Linux x86_64、aarch64、s390x 和 riscv64
* macOS x86_64、aarch64
* Windows x86_64

其他平台，如 Android、iOS 和 BSD 系列操作系统，尚未内置。欢迎针对移植的 PR，维护者也乐于为这些平台添加更多的 CI 条目。

## 编译器支持

Cranelift 支持 x86_64、aarch64、s390x 和 riscv64。目前不支持任何 32 位平台。构建 Cranelift 的新后端是一个相对较大的工作，维护者愿意提供帮助，但建议首先与 Cranelift 维护者联系以讨论此事。

Winch 支持 x86_64。aarch64 后端正在开发中。Winch 建立在 Cranelift 发出指令的支持上，因此 Winch 目前可能的后端列表限于 Cranelift 支持的。

使用 Cranelift 或 Winch 需要一个支持即时创建可执行内存页的主机操作系统。目前不支持静态链接到单个预编译模块。

Cranelift 和 Winch 都可以在 AOT 或 JIT 模式下使用。在 AOT 模式下，一个进程预先编译一个模块/组件，然后将其加载到另一个进程中。在 JIT 模式下，所有这些操作都在同一个进程内完成。

目前，Cranelift 和 Winch 都不支持从 Winch 编译自动切换到 Cranelift 编译的层级。模块完全使用 Winch 或 Cranelift 编译。

## 解释器支持

目前，`wasmtime` 没有以解释模式运行 WebAssembly 代码的模式。然而，确实希望添加对解释器的支持，这将具有最小的系统依赖性。计划系统将需要支持某种形式的动态内存分配，除此之外，其他需求不多。

## 对 `#![no_std]` 的支持

`wasmtime` crate 支持在 Rust 中不使用标准库的平台构建，但仅限于其编译时 Cargo 特性的一个子集。当前支持的 Cargo 特性包括：

* `runtime`
* `gc`
* `component-model`

这明显不包括 `default` 特性，这意味着当依赖 Wasmtime 时，你需要指定 `default-features = false`。同样明显的是，目前不包括 Cranelift 或 Winch，这意味着 no_std 平台必须在 AOT 模式下使用，即模块在其他地方预先编译。

Wasmtime 对 no_std 的支持要求嵌入者实现相当于 C 头文件的功能，以指示如何执行基本的操作系统操作，如分配虚拟内存。这个 API 可以作为 Wasmtime 发布工件中的 `wasmtime-platform.h` 查看，或在源代码树中的 `examples/min-platform/embedding/wasmtime-platform.h` 查看。注意，这个 API 目前不保证稳定，需要在更新 Wasmtime 时进行更新。

Wasmtime 运行时将使用此文件中定义的符号，这意味着如果它们没有定义，则会产生链接时错误。嵌入者需要根据其文档实现这些函数，以使 Wasmtime 能够在自定义平台上运行。

---

总结：
Wasmtime 旨在支持广泛的平台，以便任何人都可以在其上运行 WebAssembly。目前，Wasmtime 已经支持多个主流平台，包括不同架构的 Linux、macOS 和 Windows 系统。
对于其他平台，如 Android、iOS 和 BSD 系列操作系统，虽然尚未内置支持，但 Wasmtime 社区欢迎开发者提交 PR 来扩展支持。
Cranelift 和 Winch 作为 Wasmtime 的编译器后端，支持多种架构，但目前还不支持 32 位平台。Wasmtime 还提供了对 `#![no_std]` 的支持，
使其能够在不使用标准库的 Rust 平台上运行，但需要注意，这要求嵌入者实现一些基本的操作系统操作。

此外，Wasmtime 计划未来添加对解释器的支持，以进一步降低系统依赖性。
