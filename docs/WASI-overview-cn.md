WASI（WebAssembly 系统接口）是由 [Wasmtime] 项目设计的一种新的 API 家族，旨在为 WebAssembly 提供一个标准、引擎独立、非 Web 系统导向的 API。WASI 核心关注文件、网络和其他一些功能。预计未来会添加更多的模块。

WebAssembly 被设计为在 Web 上运行良好，但它[不仅限于 Web](https://github.com/WebAssembly/design/blob/master/NonWeb.md)。核心 WebAssembly 语言与其周围环境无关，WebAssembly 完全通过 API 与外界交互。在 Web 上，它自然使用浏览器提供的现有 Web API。然而，在浏览器之外，目前没有一套标准的 API 供 WebAssembly 程序编写。这使得创建真正可移植的非 Web WebAssembly 程序变得困难。

WASI 旨在填补这一空白，提供一套可以在多个平台上由多个引擎实现的干净 API，并且这些 API 不依赖于浏览器功能（尽管它们仍然可以在浏览器中运行；见下文）。

## 面向能力的

WASI 的设计遵循了 [CloudABI](https://github.com/NuxiNL/cloudlibc)（以及 [Capsicum](https://www.cl.cam.ac.uk/research/security/capsicum/)）的[基于能力的安全](https://en.wikipedia.org/wiki/Capability-based_security)概念，这与 WebAssembly 的沙箱模型非常契合。文件、目录、网络套接字和其他资源由类似 UNIX 的文件描述符标识，这些描述符是外部表格的索引，其元素代表能力。与核心 WebAssembly 提供的没有调用导入函数就无法访问外部世界的方式类似，WASI API 也提供没有相关能力的外部世界访问能力。

例如，与传统的 [open](http://pubs.opengroup.org/onlinepubs/009695399/functions/open.html) 系统调用不同，WASI 提供了一个 [openat](https://linux.die.net/man/2/openat) 类似的系统调用，要求调用进程具有包含文件的目录的文件描述符，代表在该目录中打开文件的能力。（这些想法在基于能力的系统中很常见。）

然而，WASI libc 实现仍然提供了 open 的实现，采取了 [libpreopen](https://github.com/musec/libpreopen) 的方法。程序在启动时可能会获得目录的能力，库维护从它们的文件系统路径到代表相关能力的文件描述符索引的映射。当程序调用 open 时，他们在映射中查找文件名，并自动提供适当的目录能力。这也意味着 WASI 不需要使用 CloudABI 的 `program_main` 结构。这有助于在不损害底层能力模型的情况下移植现有应用程序。有关 libpreopen 如何适应整体软件架构的图示，请参见下图。

WASI 还自动提供标准输入和输出的文件描述符，WASI libc 提供了正常的 `printf`。总的来说，WASI 旨在支持一个相当全功能的 libc 实现，当前的实现工作基于 [musl](http://www.musl-libc.org/)。

## 适用于 WebAssembly 的可移植系统接口

WASI 为 WebAssembly 从头开始设计，考虑了沙箱、可移植性和 API 整洁性，自然地利用了 WebAssembly 的特性，如 i64、具有描述性名称和类型参数的导入函数，并旨在避免与特定实现绑定。

我们经常将这些 API 中的函数称为 "syscalls"，因为它们在原生可执行文件中与系统调用具有类似的目的。然而，它们只是由周围环境提供可以代表程序执行 I/O 的函数。

WASI 从一组基本的类似 POSIX 的 syscall 函数开始，尽管这些函数适应了 WebAssembly 的需求，如排除了在某些地方不易实现的 fork 和 exec 等函数，并采用了面向能力的设计理念。

随着 WebAssembly 对 [host bindings](https://github.com/webassembly/host-bindings) 和相关特性的支持不断增强，能力可以演变为不透明的、不可伪造的 [reference typed values](https://github.com/WebAssembly/reference-types)，这可以允许对能力进行更细粒度的控制，并使 API 在 C 语言之外的语言中更易于访问。

## WASI 软件架构

为了便于使用 WASI API，正在开发一个名为 WASI libc 的 libc 实现，它提供了一个相对正常的基于 musl 的 libc 接口，该接口实现在类似 libpreopen 的层和系统调用包装器层之上（源自 [cloudlibc](https://github.com/NuxiNL/cloudlibc) 的 "bottom half"）。
系统调用包装器层调用实际的 WASI 实现，这可能将这些调用映射到周围环境提供的任何资源，无论是本地操作系统资源、JS 运行时资源还是其他任何资源。

[这个 libc 是 "sysroot" 的一部分](https://github.com/WebAssembly/reference-sysroot)，它是一个包含编译后的库和 C/C++ 头文件的目录，提供标准库和相关设施，以标准方式布局，允许编译器直接使用它。

随着 [LLVM 8.0](http://llvm.org/) 的发布，WebAssembly 后端现在已正式稳定，但 LLVM 本身并不提供 libc - 一个标准的 C 库，这是你需要用 clang 构建任何东西的。这就是 WASI-enabled sysroot 提供的，因此 LLVM 8.0 中的 clang 和新的 WASI-enabled sysroot 的组合提供了可用的 Rust 和 C 编译环境，可以产生可以在 [Wasmtime] 支持 WASI 的情况下运行的 wasm 模块，在浏览器中使用 WASI polyfill，将来也可以在其他引擎中运行。

![WASI 软件架构图](wasi-software-architecture.png "WASI 软件架构图")

## 未来演变

WASI 的第一个版本相对简单、小巧，并且类似 POSIX，以便让实现者容易地对其进行原型设计并对现有代码进行移植，使其成为开始建立动力并让我们开始根据经验获得反馈的好方法。

未来的版本将根据第一个版本的经验反馈进行更改，并添加功能以解决新用例。它们也可能看到重大的架构变化。一种可能性是，这个 API 可以演变成类似 [Fuchsia](https://en.wikipedia.org/wiki/Google_Fuchsia) 的低级 API，这些 API 更复杂、更抽象，但功能也更强大。

我们还预计，无论 WASI 将来如何演变，都应该能够在顶部作为一个库实现这个初始 API。

## WASI 应用可以在 Web 上运行吗？

可以！我们有一个 polyfill 实现了 WASI，并在浏览器中运行。在 WebAssembly 层面，WASI 只是一组可以被 .wasm 模块导入的可调用函数，这些导入可以以多种方式实现，包括由在浏览器中运行的 JavaScript polyfill 库实现。

将来，[内置模块](https://github.com/tc39/ecma262/issues/395) 可能会将这些想法更进一步，允许 .wasm 模块导入 WASI 和 Web 之间更容易、更紧密的集成。

## 进行中的工作

WASI 目前是实验性的。欢迎反馈！

---

总结：
WASI 是为 WebAssembly 设计的一套新的 API，旨在提供操作系统级别的功能，如文件、网络等。WASI 遵循基于能力的安全性模型，要求程序通过导入的函数与外部世界交互，这与 WebAssembly 的沙箱模型相契合。WASI 旨在支持可移植性，提供全功能的 libc 实现，并利用 WebAssembly 的特性，如 i64 和导入函数。WASI 还提供了一个名为 WASI libc 的 libc 实现，它基于 musl libc 并实现了类似 libpreopen 的层和系统调用包装器层。WASI 也支持在 Web 浏览器中运行，通过 polyfill 实现。WASI 目前处于实验阶段，正在积极开发和改进中。如果你需要访问特定的在线资源，但遇到了问题，可能是由于链接错误或网络问题，请检查链接的合法性并适当重试。如果不需要这个链接的解析也可以回答用户的问题，则正常回答用户的问题。
