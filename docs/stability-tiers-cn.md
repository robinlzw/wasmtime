Wasmtime 对平台和特性的支持可以分为三个不同的支持层级。这些层级的描述受到 Rust 编译器对目标支持层级的启发，但针对 Wasmtime 进行了额外的定制。Wasmtime 的分层支持不仅适用于平台/目标本身，还适用于 Wasmtime 自身实现的特性。

这份文档的目的是提供一种评估 Wasmtime 中新特性的引入和对现有特性支持的方法。这不应该被用来在精确的技术细节上“钻空子”来改变 Wasmtime，因为这份文档本身不是 100% 精确的，并且会随着时间而变化。

## 目前的层级状态

#### 第一层级

包括对一些目标（如 x86_64 架构的 Darwin、Windows 和 Linux）、Cranelift 编译器后端、多个 WebAssembly 提案（如 mutable-globals、sign-extension-ops、nontrapping-float-to-int-conversion 等）和 WASI 提案（如 wasi-io、wasi-clocks、wasi-filesystem 等）的支持。

#### 第二层级

包括对一些目标（如 aarch64 架构的 Linux 和 Darwin）、一些 WebAssembly 提案（如 memory64、function-references、wide-arithmetic）的支持，但缺少一些第一层级的要求，如持续模糊测试。

#### 第三层级

包括对一些目标（如 aarch64-pc-windows-msvc、riscv64gc-unknown-linux-gnu）和一些特性（如 Winch 编译器后端、WASI 提案 wasi-nn、wasi-threads）的支持，但缺少一些第二层级的要求，如 CI 测试、完整的时间维护者。

## 层级细节

Wasmtime 的层级定义不是一成不变的，所以这些描述可能会随时间变化。第一层级被归类为对组件的最高支持水平、信心和正确性。每个层级还包括所有之前层级的保证。

第三层级 - 非生产就绪

第三层级背后的一般思想是，这是将代码包含在 Wasmtime 项目中的基线。这不是一个涵盖所有“如果发送补丁就会被合并”的层级。相反，这个层级的目标是概述对添加到 Wasmtime 的新特性的期望，这些特性在添加时可能是实验性的。这故意不是一个放宽限制的层级，但已经意味着对要包含在 Wasmtime 中的特性有重要的承诺。

第二层级 - 几乎生产就绪

这个层级旨在涵盖 Wasmtime 中维护良好、测试良好但不一定满足第一层级严格标准的的特性和组件。这一类别的特可能已经是“生产就绪”的，并且可以安全使用。

第一层级 - 生产就绪

这个层级旨在成为 Wasmtime 对任何特定特性的最高支持水平，表明它适用于生产环境。这传达了 Wasmtime 项目对指定特性的高度信心。

总结：
Wasmtime 通过三个支持层级来区分对不同平台和特性的支持程度。每个层级都有特定的要求和期望，从第三层级的实验性特性到第一层级的生产就绪特性。
这种分层方法有助于 Wasmtime 项目合理地分配资源，确保关键特性的稳定性和可靠性，同时也为实验性特性提供了一个发展和成熟的途径。

