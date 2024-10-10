# Wasmtime 架构

本文档旨在概述 Wasmtime 的实现。这将解释主要的 `wasmtime-*`  crate，以及 `wasmtime` crate 依赖于它们的原因。对于更详细的信息，建议直接查看代码本身和其中的注释。

## `wasmtime` crate

Wasmtime 的主要入口点是 `wasmtime` crate 本身。Wasmtime 设计得使得 `wasmtime` crate 几乎是一个 100% 安全的 API（在 Rust 的意义上是安全的），
除了一些小型且文档记录良好的函数外，它们是 `unsafe`。`wasmtime` crate 提供了对 WebAssembly 原语和功能（如编译模块、实例化它们、调用函数等）的访问。

目前，`wasmtime` crate 是用户使用的第一个 crate。这里的“第一”意味着 `wasmtime` 依赖的所有内容都被视为内部依赖。我们发布到 crates.io 的 crate，
但对内部 crate 的“良好”API或担心内部 crate 版本之间的破坏性变更投入很少的努力。这意味着这里讨论的所有其他 crate 都被视为 Wasmtime 的内部依赖，
并且根本不出现在 Wasmtime 的公共 API 中。使用 Cargo 的术语来说，所有 `wasmtime` 依赖的 `wasmtime-*` crates 都是“私有”依赖。

此外，目前 Wasmtime 内部 crate 之间的安全/不安全边界并不是最明确定义的。有些方法本应该被标记为 `unsafe` 但并没有，而 `unsafe` 方法并没有详尽的文档说明它们为什么是 `unsafe`。
这是一个持续改进的领域，目标是让安全方法在 Rust 的意义上实际上是安全的，并且为 `unsafe` 方法提供文档，清楚地列出它们为什么是 `unsafe`。

## 重要概念

在讨论更详细的内部实现之前，了解一些重要类型及其在 Wasmtime 中的含义非常重要：

* `wasmtime::Engine` - 这是一个全局编译上下文，可以看作是“根上下文”。`Engine` 通常每个程序创建一次，并预期在多个线程之间共享（内部它是原子引用计数的）。
每个 `Engine` 存储配置值和其他跨线程数据，例如 `Module` 实例的类型内部。关于 `Engine` 的主要事项是，对其内部的任何更改通常涉及获取锁，而 `Store` 下面不需要锁。

* `wasmtime::Store` - 这是 WebAssembly 中的“存储”概念。虽然也有正式的定义，但可以将其视为相关 WebAssembly 对象的集合。这包括实例、全局变量、内存、表格等。
`Store` 不实现对内部项目的任何形式的垃圾收集（有一个 `gc` 函数，但那只是针对 `externref` 值）。

这意味着一旦你创建了一个 `Instance` 或一个 `Table`，内存实际上在 `Store` 本身被释放之前不会被释放。`Store` 是用于几乎所有 wasm 操作的“上下文”。
`Store` 还包含实例句柄，这些句柄递归地引用回 `Store`，导致 `Store` 内大量指针别名。现在重要的是要知道 `Store` 是一个隔离单元。
WebAssembly 对象完全包含在一个 `Store` 内，目前没有任何东西可以在存储之间交叉（除了你手动连接的标量）。换句话说，来自不同存储的 wasm 对象不能相互交互。
`Store` 不能同时从多个线程使用（几乎所有操作都需要 `&mut self`）。

* `wasmtime::runtime::vm::InstanceHandle` - 这是 WebAssembly 实例的低级表示。同时，这也用作所有主机定义对象的表示。
例如，如果你调用 `wasmtime::Memory::new`，它将在内部创建一个 `InstanceHandle`。
这是一个非常 `unsafe` 的类型，应该将其所有函数都标记为 `unsafe` 或者以其他方式记录更严格的保证，但这是一个内部类型，我们目前没有考虑公开使用。
`InstanceHandle` 不知道如何自我释放，并依赖于调用者管理其内存。目前，这要么是按需分配（使用 `malloc`），要么是池式分配（使用池式分配器）。
`deallocate` 方法在这两条路径中是不同的（以及 `allocate` 方法）。

  `InstanceHandle` 在内存中布局有一些 Rust 拥有的值首先捕获内存/表格等的动态状态。
  这些字段大部分对于主机定义的对象是未使用的（例如一个 `wasmtime::Table::new`），但对于实例化的 WebAssembly 模块，这些字段将包含更多信息。
  在内存中的 `InstanceHandle` 之后是一个 `VMContext`，接下来将讨论。`InstanceHandle` 值是主要的内部运行时表示，是 `crate::runtime::vm` 代码所处理的。
  `wasmtime::Store` 持有所有这些 `InstanceHandle` 值，并在适当的时间释放它们。
  从运行时的角度来看，这简化了事情，因此 wasm 模块相互通信的图被简化为 `InstanceHandle` 值全部相互通信。`wasmtime::Store` 持有所有这些 `InstanceHandle` 值，并在适当的时间释放它们。

* `crate::runtime::vm::VMContext` - 这是一个原始指针，在 `InstanceHandle` 的分配中传递。`VMContext` 没有在 Rust 中定义结构（它是一个 0 尺寸结构），
因为它的内容根据 `VMOffsets` 动态确定，或者它来自源 wasm 模块。每个 `InstanceHandle` 有一个与它对应的 `VMContext` 的“形状”。

例如，一个 `VMContext` 存储了所有 WebAssembly 全局变量的值，但如果一个 wasm 模块没有全局变量，那么这个数组的大小将是 0，它将不会被分配。
`VMContext` 的意图是成为 JIT 代码可能访问的所有 wasm 模块状态的高效内存表示。`VMContext` 的布局由模块动态确定，JIT 代码专门针对这个结构。

这意味着该结构被 JIT 代码高效访问，但被原生主机代码访问效率较低。`VMContext` 的非详尽列表的用途是：

  * 存储 WebAssembly 实例状态，例如全局值、指向表格的指针、指向内存的指针以及指向其他 JIT 函数的指针。
  * 分离 wasm 导入和本地状态。导入值有指向其实际值的指针存储，本地状态有内联定义的状态。
  * 保存触发 JIT 代码堆栈溢出的堆栈限制的指针。
  * 保存指向 `VMExternRefActivationsTable` 的指针，用于快速路径插入 `externref` 值到表格中。
  * 保存指向 `*mut dyn crate::runtime::vm::Store` 的指针，以便在 libcalls 中执行存储级操作。

  关于 `VMContext` 布局的注释可以在 `vmoffsets.rs` 文件中找到。

* `wasmtime::Module` - 这是编译后的 WebAssembly 模块的表示。目前 Wasmtime 总是假设 wasm 模块总是编译为本机 JIT 代码。
`Module` 保存了所述编译的结果，目前 Cranelift 可以用于编译。Wasmtime 的目标是支持其他表示模块的模式，但这些目前尚未实现，只有 Cranelift 实现并支持。

* `wasmtime_environ::Module` - 这是一个描述 wasm 模块类型和结构的描述符，而没有保存任何实际的 JIT 代码。
这种类型的实例在编译过程的早期创建，当函数本身实际编译时不会被修改。这保留了关于函数、全局变量等的内部类型表示和状态。
从某种意义上说，这可以被看作是验证或类型检查 wasm 模块的结果，尽管它没有每个操作码的类型或详细的函数级细节等信息。

## 编译模块

有了高层次的概述和一些背景信息，这将详细介绍编译 WebAssembly 模块所采取的步骤。主要的入口点是 `wasmtime::Module::from_binary` API。
还有许多其他入口点处理表面级别的细节，如从文本到二进制的翻译、从文件系统加载等。

编译大致分为几个阶段：

1. 首先编译遍历 WebAssembly 模块，验证除了函数体之外的所有内容。这个同步的 wasm 模块传递创建了一个 `wasmtime_environ::Module` 实例，并为函数编译做准备。
注意，随着模块链接提案，一个输入模块最终可能创建多个输出模块以进行处理。每个模块独立处理，所有进一步的步骤都基于每个模块并行化。

注意，WebAssembly 模块的解析和验证使用 `wasmparser` crate 进行。验证与解析交错进行，在使用解析值之前验证它们。

2. 接下来，模块内的所有函数并行验证和编译。目前不执行任何跨过程分析，每个函数都作为自己的代码孤岛编译。这是在每个函数基础上调用 Cranelift 的时候。

3. 此时编译结果都编织成一个 `wasmtime_jit::CompilationArtifacts` 结构。
这保存了模块信息（`wasmtime_environ::Module`）、编译的 JIT 代码（存储为 ELF 图像）以及有关函数的各种其他信息，
如平台不可知的展开信息、每个函数的陷阱表（指示哪些 JIT 指令可以陷阱以及陷阱的含义）、每个函数的地址映射（从 JIT 地址映射回 wasm 偏移量）以及从 wasm 模块中的 DWARF 信息解析的调试信息。
这些结果目前是惰性的，实际上不能执行，但它们适合在这一点上序列化到磁盘或开始下一个阶段...

4. 最后一步是实际上将所有代码放入准备执行的形式。这一步从上一步的 `CompilationArtifacts` 开始。在这里，分配了一个新的内存映射，并将 JIT 代码复制到这个内存映射中。
然后这个内存映射从读写切换为只读/执行，因此它实际上是可执行的 JIT 代码。在这里，各种钩子如加载调试信息、通知 JIT 分析器新代码等都发生。
此时产生了一个 `wasmtime_jit::CompiledModule`，它本身被包装在一个 `wasmtime::Module` 中。此时，模块已准备好实例化。

`wasmtime::Module` 是一个原子引用计数对象，一旦实例化到 `Store` 中，`Store` 将持有对模块内部的强引用。这意味着所有实例化的 `wasmtime::Module` 共享相同的编译代码。
此外，`wasmtime::Module` 是少数几个生活在 `wasmtime::Store` 之外的对象之一。这意味着 `wasmtime::Module` 的引用计数是其自己的内存管理形式。

注意，共享模块编译代码的属性对编译代码可以假设的内容有有趣的影响。例如，Wasmtime 实现了一种类型内部表示形式，但内部表示形式的类型在几个不同的级别上发生。
在一个模块内我们消除了函数类型的重复，但在 `Store` 中跨模块的类型需要用相同的值表示。
这意味着如果相同的模块被实例化到许多存储中，其相同的函数类型可能采用许多值，因此编译代码不能假设函数类型的特定值。（有关类型信息的更多信息稍后介绍）。
总的来说，编译代码在很大程度上依赖于 `VMContext` 来获取上下文输入，因为 JIT 代码旨在如此广泛地重用。

### _trampoline（蹦床）

编译的一个重要方面是创建蹦床。在这种情况下，蹦床指的是由 Wasmtime 执行以进入 WebAssembly 代码的代码。主机可能事先不知道它想要调用的 WebAssembly 函数的签名。
Wasmtime JIT 代码使用本机 ABIs 编译（例如，根据 Unix 上的 System V 将参数/结果存储在寄存器中），这意味着 Wasmtime 嵌入环境没有简单的方法进入 JIT 代码。

蹦床编译到模块中解决的问题，它提供了一个已知 ABI 的函数，该函数将调用具有特定其他类型签名/ABI 的函数。Wasmtime 收集模块的所有导出函数并创建它们的类型签名集。
请注意，在这个上下文中导出实际上意味着“可能导出”，包括插入到全局/函数表中，转换为 `funcref` 等。为这些类型签名中的每一个生成蹦床，并与模块的其余 JIT 代码一起存储。

然后这些蹦床在 `wasmtime::Func::call` API 中使用，在该特定情况下，因为我们不知道目标函数的 ABI，所以使用蹦床（具有已知 ABI）并通过网络堆栈传递所有参数/结果。

另一个值得注意的点是，蹦床目前没有被消除重复项。每个编译的模块都包含自己的蹦床集合，如果两个编译的模块具有相同的类型，它们将拥有相同的蹦床的不同副本。

### 类型内部表示形式和 `VMSharedSignatureIndex`

编译的一个重要方面是 `VMSharedSignatureIndex` 类型及其使用方式。Wasm 中的 `call_indirect` 操作码将实际函数的签名与指令的函数签名进行比较，如果签名不匹配则陷阱。
这在 Wasmtime 中通过整数比较实现，并且比较发生在 `VMSharedSignatureIndex` 值上。这个索引是对函数类型的内部表示形式的内部表示。

`VMSharedSignatureIndex` 的内部表示范围发生在 `wasmtime::Engine` 级别。模块被编译到一个 `Engine` 中。将 `Module` 插入到 `Engine` 中将为模块中发现的所有类型分配一个 `VMSharedSignatureIndex`。

模块的 `VMSharedSignatureIndex` 值对于该模块的一个实例是本地的（它们可能在每次将一个 `Module` 插入到不同的 `Engine` 中时发生变化）。这些在实例化过程中由运行时用于为所有函数分配类型 ID。

## 实例化模块

一旦模块被编译，通常会实例化以实际访问导出并调用 wasm 代码。实例化总是在 `wasmtime::Store` 内发生，创建的实例（加上所有导出）与 `Store` 相关联。

实例化的大致流程如下：

1. 首先，对所有导入进行类型检查。提供的导入列表与 `wasmtime_environ::Module` 中记录的导入列表进行交叉引用，并验证所有类型是否一致并匹配（根据核心 wasm 规范的类型匹配定义）。

2. 每个 `wasmtime_environ::Module` 都有一个初始化程序列表，需要在实例化完成之前完成。
对于 MVP wasm，这只涉及将导入加载到正确的索引数组中，但对于模块链接，这可能涉及实例化其他模块、处理 `alias` 字段等。
在任何情况下，这一步的结果是 `crate::runtime::vm::Imports` 数组，它具有 wasm 模块的所有导入项的值。
请注意，在这种情况下，导入通常是某种原始指针到实际状态以及导入自的实例的 `VMContext`。这一步的最终结果是一个 `InstanceAllocationRequest`，然后提交给配置的实例分配器，无论是按需还是池式。

3. 分配相应的 `InstanceHandle`。这种分配方式取决于策略（malloc 按需，池式分配）。
除了初始化 `InstanceHandle` 的字段外，这还初始化了该句柄的 `VMContext` 的所有字段（如上所述，它在内存中位于 `InstanceHandle` 分配之后）。
这不会在此时处理任何数据段、元素段或 `start` 函数。

4. 在这一点上，`InstanceHandle` 存储在 `Store` 中。这是“无路可退”的点，句柄必须保持活动状态，与 `Store` 本身的生命周期相同。
如果初始化步骤失败，那么实例可能已经通过元素段将其函数插入到导入表中。这意味着即使我们未能初始化此实例，其状态仍可能对其他实例/对象可见，因此我们需要保持它活动状态。

5. 最后一步是执行 wasm 定义的实例化。这涉及处理元素段、数据段、`start` 函数等。这些操作大多是将 Wasmtime 的内部表示转换为规范所需的行为。

实例化模块的另一部分值得注意的是，在 `Store` 中维护了一个 `ModuleRegistry`，用于存储存储中实例化的所有模块。这个注册表的目的是保留运行实例所需的模块项的强引用。
这主要包括 JIT 代码，但也包括有关函数地址的元数据等。这些数据大多存储在 `GLOBAL_MODULES` 映射中，以便在陷阱期间稍后访问。

## 陷阱

实例创建后，wasm 开始运行，大多数事情都是相当标准的。蹦床用于进入 wasm（或者如果我们使用 `wasmtime::TypedFunc`，则可以使用已知 ABI），JIT 代码通常执行 wasm。然而，实现的一个重要方面是陷阱。

今天的 Wasmtime 使用 `longjmp` 和 `setjmp` 实现陷阱。`setjmp` 函数不能在 Rust 中定义（甚至不安全地定义 - https://github.com/rust-lang/rfcs/issues/2625），
因此 `crates/wasmtime/src/runtime/vm/helpers.c` 文件实际上调用了 setjmp/longjmp。请注意，通常在 Rust 中执行 `longjmp` 是不安全的，因为它跳过了基于堆栈的析构函数，
因此在我们回调到 Rust 执行 wasm 后，需要在 Wasmtime 中小心，一旦 wasm 被调用，堆栈上就不要有任何重要的析构函数。

陷阱可以从几个不同的来源发生：

* 显式陷阱 - 当主机调用返回陷阱时，例如。这些在 `raise_user_trap` 或 `raise_lib_trap` 中触底，两者都立即调用 `longjmp` 返回到 wasm 起始点。
请注意，这些操作，像调用 wasm 一样，必须非常小心，以确保在调用 wasm 时堆栈上没有任何析构函数。

* 信号 - 这是陷阱的主要向量。基本上，我们使用段错误和非法指令来实现 wasm 代码中的陷阱。当线性内存访问超出界限时，会发生段错误，非法指令是 wasm `unreachable` 指令的实现方式。
在这两种情况下，Wasmtime 安装了一个特定于平台的信号处理程序来捕获信号，检查信号的状态，然后处理它。
请注意，Wasmtime 试图只捕获来自 JIT 代码本身的信号，以免意外掩盖其他错误。通过 `longjmp` 退出信号处理程序，返回到原始的 wasm 调用站点。

总的来说，Wasmtime 对 wasm 的堆栈帧（自然通过 Cranelift）以及执行代码（即在我们进入 wasm 之前 aka 在 `setjmp` 之前）和重新进入 wasm 之后（aka 在可能的 `longjmp` 之前的帧）有非常严格的控制。

Wasmtime 的信号处理程序使用在实例化期间填充的 `GLOBAL_MODULES` 映射，以确定触发信号的程序计数器是否确实是一个有效的 wasm 陷阱。这应该是正确的，除了主机程序有其他错误触发了信号的情况。

最后值得一提的是，Wasmtime 使用 Rust `backtrace` crate 在 wasm 异常发生时捕获堆栈跟踪。这迫使 Wasmtime 生成特定于本机平台的展开信息，以正确展开堆栈并为 wasm 代码生成堆栈跟踪。
这也带来了其他好处，例如在使用 Wasmtime 时改进通用采样分析器。

## 线性内存

Wasmtime 中的线性内存实际上是用 `mmap`（或平台等效物）实现的，但有一些微妙的细微差别值得指出。线性内存的实现相对可配置，这导致了运行时和生成的代码需要处理的一些情况。

首先，线性内存的一些属性可以配置：

* `wasmtime::Config::static_memory_maximum_size`
* `wasmtime::Config::static_memory_guard_size`
* `wasmtime::Config::dynamic_memory_guard_size`
* `wasmtime::Config::guard_before_linear_memory`

`Config` 上的方法有很多文档，可以详细讨论一些细节，但总的来说，Wasmtime 有两种内存模式：静态和动态。
静态内存表示一个地址空间保留，它永远不会移动，页面被提交以表示内存增长。动态内存表示分配，其中提交的部分与 wasm 内存的大小完全匹配，增长通过分配更大的内存块发生。

守护大小配置指示线性内存之后的守护区域的大小。这个守护大小影响 JIT 代码是否发出界限检查。如果越界地址可以证明遇到了守护页面，则省略界限检查。

`guard_before_linear_memory` 配置还会在线性内存之前放置守护页面，以及之后（两端的大小相同）。这只是用来保护可能的 Cranelift 错误，否则没有任何用途。

对于 64 位平台的 Wasmtime，默认设置如下：

* 4GB 静态最大大小，意味着所有 32 位内存都是静态的，64 位内存是动态的。
* 2GB 静态守护大小，意味着所有负载/存储的偏移量小于 2GB 不需要界限检查 32 位内存。
* 启用了线性内存之前的守护页面。

总的来说，这意味着 32 位线性内存默认在 Wasmtime 中导致 8GB 虚拟地址空间保留。
使用池式分配器，其中我们知道线性内存是连续的，这导致每个内存 6GB 保留，因为一个内存之后的守护区域是下一个内存之前的守护区域。

请注意，64 位内存（WebAssembly 的 memory64 提案）可以配置为静态，但目前还不能省略界限检查。这个配置可以通过 `static_memory_forced` 配置选项实现。
另外请注意，Wasmtime 中对 64 位内存的支持是功能性的，但目前还没有调整，所以可能还有一些性能工作和更好的默认设置需要管理。

## 表格和 `externref`

WebAssembly 表包含引用类型，当前要么是 `funcref`，要么是 `externref`。
在 Wasmtime 中，`funcref` 表示为 `*mut VMCallerCheckedFuncRef`，而 `externref` 表示为 `VMExternRef`（内部是 `*mut VMExternData`）。
表因此被表示为指针向量。默认情况下，表的存储内存管理通过 Rust 的 `Vec` 进行，它使用 `malloc` 和朋友进行内存管理。使用池式分配器时，它使用预分配的内存进行存储。

如前所述，`Store` 对 wasm 对象本身没有内部垃圾收集，因此 wasm 中的 `funcref` 表非常简单，即不管理存储在其中的任何指针的生命周期，它们被简单地假定为只要表在使用，它们就有效。

对于 `externref` 表，情况就更加复杂。`VMExternRef` 是 `Arc<dyn Any>` 的一个版本，但在 Wasmtime 中专门用于 JIT 代码知道直接操作它的引用计数字段的偏移量。
此外，`externref` 表需要自己管理引用计数字段，因为存储在表中的指针需要为它分配一个强引用计数。

## GC 和 `externref`

Wasmtime 使用原子引用计数指针实现 WebAssembly 的 `externref` 类型。请注意，原子部分不是 wasm 本身需要的，而是来自必须安全地将 `ExternRef` 值发送到其他线程的 Rust 嵌入环境。
Wasmtime 也没有带有循环收集器，因此主机分配的 `ExternRef` 对象的循环将泄漏。

尽管如此，`Store::gc` 方法仍然存在。这是在 wasm 代码执行时管理引用计数的实现细节。
不是在 `externref` 值在堆栈上移动时单独管理其引用计数，Wasmtime 实现了“延迟引用计数”，
其中有一个过于保守的 `ExternRef` 值列表，这些值可能正在使用中，并且定期执行 GC 以使这个过于保守的列表成为一个精确的列表。
这利用了 Cranelift 的堆栈映射支持以及 `backtrace` 的回溯支持，以确定堆栈上的活动根。`Store::gc` 方法强制可能过于保守的列表成为一个精确的 `externref` 值列表，这些值在堆栈上积极使用中。

## crates 索引

主要的 Wasmtime 内部 crates 如下：

* `wasmtime` - `wasmtime` 的安全公共 API。
  * `wasmtime::runtime::vm` - Wasmtime 的低级运行时实现。这是 `VMContext` 和 `InstanceHandle` 存在的地方。这个模块曾经是一个 crate，但现在已经折叠到 `wasmtime` 中。
* `wasmtime-environ` - 低级编译支持。这是 `Module` 及其环境的翻译发生的地方，尽管没有在这个 crate 中实际进行编译（尽管它为编译器定义了接口）。这个 crate 的结果被传递给其他 crates 进行实际编译。
* `wasmtime-cranelift` - 使用 Cranelift 实现函数级编译。

请注意，目前 Cranelift 是 wasmtime 的必需依赖项。大多数从 `wasmtime-environ` 导出的类型在其 API 中使用 cranelift 类型。有一天，目标是移除必需的 cranelift 依赖项，并使 `wasmtime-environ` 成为一个相对独立的 crate。

除了上述 crates 之外，还有一些其他的杂项 crates `wasmtime` 依赖：

* `wasmtime-cache` - 可选依赖项，用于管理文件系统上的默认缓存。这在 CLI 中默认启用，但在 `wasmtime` crate 中默认不启用。
* `wasmtime-fiber` - 实现 Wasmtime 中 `async` 支持使用的堆栈切换。
* `wasmtime-debug` - 实现将 wasm dwarf 调试信息映射到本机 dwarf 调试信息。
* `wasmtime-profiling` - 实现将生成的 JIT 代码连接到标准分析运行时。
* `wasmtime-obj` - 实现从编译的函数创建 ELF 图像。

---

总结：
Wasmtime 是一个用于 WebAssembly、WASI 和组件模型的独立运行时环境，由 Bytecode Alliance 提供。它旨在提供快速、安全、可配置且符合标准的运行时环境。
Wasmtime 的架构基于 Rust 编程语言构建，提供了一个安全的 API，并且大部分 API 都是安全的。Wasmtime 的内部实现涉及多个 crates，每个都有其特定的功能和责任。
主要的 crates 包括 `wasmtime`、`wasmtime-environ` 和 `wasmtime-cranelift`，以及其他支持性 crates。Wasmtime 的设计考虑了 WebAssembly 的特性，如类型内部表示形式、线性内存管理和表格处理。
此外，Wasmtime 还实现了对 `externref` 类型的垃圾收集和延迟引用计数，以管理 wasm 代码执行期间的资源。Wasmtime 的架构和实现细节展示了其对性能、安全性和可移植性的重视，使其成为一个适用于多种应用场景的强大工具。


