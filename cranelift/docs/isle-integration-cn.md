## ISLE与Cranelift的集成方式

本文提供了关于ISLE如何融入Cranelift的概览和常见问题解答。

### ISLE是什么？

ISLE（Instruction Selection and Lowering Expressions）是一种领域特定语言，用于编写指令选择和重写规则。ISLE源代码被[编译成Rust代码](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift/isle#implementation)。

关于ISLE语言本身的文档可以在此找到[ISLE语言参考手册](../isle/docs/language-reference.md)。

### ISLE如何与构建系统集成？

构建集成位于`cranelift/codegen/build.rs`中。

ISLE编译器被构建为一个构建依赖项，然后构建脚本使用它将ISLE源代码编译成生成的Rust代码。换句话说，ISLE编译器表现为一个额外的编译步骤，ISLE源代码就像任何Rust源代码一样被重建。编辑ISLE时不需要做特别的事情。

有时，我们希望看到实际生成的代码。默认情况下，生成的代码被放置在Cargo管理的`target/`路径中。如果你想查看源代码，可以如下使用可选特性`isle-in-source-tree`调用Cargo：

```shell
$ cargo check -p cranelift-codegen --features isle-in-source-tree
```

这将把ISLE源代码放置在`cranelift/codegen/isle_generated_code/`，你可以在那里检查它，在其中设置断点进行调试等。注意，如果你后来不使用这个特性构建，构建系统将要求你删除该目录。这是为了确保没有过时的副本存在，这可能会导致极大的混淆。

如果ISLE编译过程中有任何错误（例如，类型不匹配），你将看到一个带有文件、行号和一行错误的基本错误消息。要查看带有上下文的更详细的输出，可以使用`--features isle-errors`。这将给出带有源代码上下文的漂亮打印错误。

此外，`cranelift-codegen-meta` crate将自动生成ISLE `extern`声明和助手函数，用于处理CLIF。执行此操作的代码在`cranelift/codegen/meta/src/gen_inst.rs`中定义，它在`target/`输出目录中创建了几个ISLE文件，随后由ISLE编译器作为其序言的一部分读取。

### 相关文件在哪里？

* `cranelift/isle`：ISLE编译器的源代码。
* `cranelift/codegen/src/prelude.isle`：ISLE的公共定义和声明。这在每次ISLE编译中都被包含。
* `target/.../out/clif_lower.isle`：自动生成的与CLIF一起工作的声明和助手函数，用于指令降低。由`cranelift/codegen/build.rs`生成，构建到每个后端中。
* `target/.../out/clif_opt.isle`：自动生成的与CLIF一起工作的声明和助手函数，用于中端优化。由`cranelift/codegen/build.rs`生成，构建到中端优化器中。
* `cranelift/codegen/src/machinst/isle.rs`：将ISLE生成的代码集成到目标架构后端的公共Rust代码。包含在ISLE中声明的ISA不可知`extern`助手函数的实现。
* `cranelift/codegen/src/isa/<arch>/inst.isle`：ISA特定的ISLE助手。包含ISA中每个指令的构造函数，或获取特定寄存器的助手。有助于弥合原始的、非SSA ISA与降低规则所具有的纯SSA视图之间的差距。
* `cranelift/codegen/src/isa/<arch>/lower.isle`：ISA的指令选择降低规则。这些应该是纯的、SSA重写规则，有助于最终验证。
* `cranelift/codegen/src/isa/<arch>/lower/isle.rs`：Rust胶水代码，用于将这个ISA的ISLE生成的Rust代码集成到这个ISA的后端其余部分。包含在ISLE中声明的ISA特定`extern`助手函数的实现。

### 将ISLE生成的代码集成到Cranelift中

每个ISA特定的、由ISLE生成的文件都是泛型的`Context`特性，每个在ISLE中定义的`extern`助手都有一个特性方法。在`cranelift/codegen/src/isa/<arch>/lower/isle.rs`中定义了这些特性的一个具体实现。通常，将ISLE生成的代码集成到系统其余部分的方式是使用这些特性实现。

可能还有一个`lower`函数在`isle.rs`中定义，它封装了创建ISLE `Context`并调用生成的代码。

### 降低规则始终是纯的，使用SSA

定义在`cranelift/codegen/src/isa/<arch>/lower.isle`中的降低规则本身必须始终是从CLIF指令到目标ISA的`MachInst`的纯映射。

降低规则本身不应该处理或谈论的事情包括：

* 被修改的寄存器（既读取又写入，违反SSA）
* 寄存器的隐式使用
* 维护每个CLIF值或虚拟寄存器的使用计数

相反，这些事情应该由`cranelift/codegen/src/isa/<arch>/inst.isle`和一般的Rust代码（无论是在`cranelift/codegen/src/isa/<arch>/lower/isle.rs`中还是其他地方）的某种组合来处理。

当一个指令修改一个寄存器时，既要读取它又要写入它，我们通过“移动有丝分裂”来构建该指令的SSA视图，将移动从寄存器中分离出来。

例如，在x86上，`add`指令读取并写入它的第一个操作数：

    add a, b    ==    a = a + b

因此，我们呈现了一个SSA外观，其中`add`操作三个寄存器，而不是两个，并定义了其中之一，同时读取其他两个并保留它们不变：

    add a, b, c    ==    a = b + c

然后，作为一个实现细节的外观，我们根据需要发出移动：

    add a, b, c    ==>    mov a, b; add b, c

我们将发出这些移动的过程称为“移动有丝分裂”。对于像x86这样广泛使用被修改的寄存器和两操作数形式指令的ISA，我们通过ISA的`MachInst`方法实现移动有丝分裂。对于其他更接近RISC的ISA，如aarch64，其中被修改的寄存器非常罕见，我们在`inst.isle`层实现少量移动有丝分裂特殊情况。无论哪种方式，重要的是降低规则保持纯净。

最后，请注意，这些移动通常由寄存器分配器的移动合并清理，一旦我们切换到直接以SSA形式输入指令的`regalloc2`，移动有丝分裂将完全消失。

隐式操作特定寄存器的指令，或要求某些操作数在特定寄存器中的指令，也以类似方式处理：降低规则使用一个纯范式，忽略这些约束，并有显式采取隐式操作数的指令，我们确保在降低规则下一层（在`inst.isle`或Rust胶水代码中）满足这些约束。

### 降低规则何时被允许有副作用？

提取器（出现在`rule`左侧的匹配器）应该**永远**没有副作用。在评估规则的提取器时，我们还没有承诺评估该规则的右侧。如果提取器执行了副作用，我们可能会得到非常令人困惑的远程行动错误，其中我们从未完全匹配的规则可能会让我们措手不及。

每当你被诱惑在提取器中执行副作用时，你应该将需要执行该副作用的内容打包起来，然后有一个单独的构造函数接受该包并执行它描述的副作用。构造函数只能在规则的右侧调用，这只有在我们对这个规则做出承诺后才会发生，从而避免了前面描述的远程行动错误。

例如，加载在CLIF中有副作用：它们可能会陷入困境。因此，即使加载的值从未被使用，我们也会发出实现该加载的代码。但是，如果我们为x86编译，我们可以根据加载值的使用方式将加载下沉到另一个操作的操作数中。如果我们将该加载下沉到，比如说，`add`中，那么我们需要告诉降低上下文*不要*再降低CLIF `load`指令了，因为它已经有效地作为降低使用加载值的`add`的一部分降低了。标记指令为“已经降低了”是一个副作用，我们可能会被诱惑在匹配可下沉加载的提取器中执行该副作用。但我们不能这样做，因为尽管加载本身可能是可下沉的，但可能有原因我们最终不执行这个加载下沉规则，如果发生这种情况，我们仍然需要降低CLIF加载。

因此，我们使`sinkable_load`提取器创建一个`SinkableLoad`类型，它打包了我们了解加载和如何告诉降低上下文它已经下沉以及降低上下文不再需要降低它所需的所有信息，但*它实际上还没有告诉降低上下文*。

```lisp
;; inst.isle

;; 如果值由兼容的加载定义，则提取一个`SinkableLoad`。
(type SinkableLoad extern (enum))

;; 从值中提取一个`SinkableLoad`。
(decl sinkable_load (SinkableLoad)
(extern extractor sinkable_load sinkable_load)
```

ISLE（Instruction Selection and Lowering Expressions）是一种领域特定语言（DSL），用于编写指令选择和重写规则。ISLE源代码被编译成Rust代码，并集成到Cranelift编译器项目中。
ISLE的设计目标是提高编译器开发效率，通过简洁的方式表达指令映射关系，同时保持类型安全，并允许使用声明式模式进行多种目的的优化。

### ISLE与构建系统的集成

ISLE编译器作为构建依赖项被构建，构建脚本使用它将ISLE源代码编译成生成的Rust代码。这意味着ISLE编译器作为一个额外的编译步骤，ISLE源代码的重建就像任何Rust源代码一样。如果需要查看实际生成的代码，可以通过启用`isle-in-source-tree`特性来实现。

### 相关文件位置

- `cranelift/isle`：ISLE编译器的源代码。
- `cranelift/codegen/src/prelude.isle`：ISLE的公共定义和声明。
- `target/.../out/clif_lower.isle`和`clif_opt.isle`：自动生成的与CLIF一起工作的声明和助手函数，分别用于指令降低和中端优化。
- `cranelift/codegen/src/machinst/isle.rs`：将ISLE生成的代码集成到目标架构后端的公共Rust代码。
- `cranelift/codegen/src/isa/<arch>/inst.isle`和`lower.isle`：ISA特定的ISLE助手和指令选择降低规则。

### 将ISLE生成的代码集成到Cranelift

ISLE生成的文件是泛型的`Context`特性，每个在ISLE中定义的`extern`助手都有一个特性方法。具体的实现定义在`cranelift/codegen/src/isa/<arch>/lower/isle.rs`中。通常，ISLE生成的代码通过这些特性实现被集成到系统中。

### 降低规则的纯度和SSA使用

降低规则定义在`cranelift/codegen/src/isa/<arch>/lower.isle`中，必须始终是从CLIF指令到目标ISA的`MachInst`的纯映射。这些规则不应该处理违反SSA的寄存器，或者涉及隐式寄存器使用和维护CLIF值或虚拟寄存器的使用计数。

### 降低规则的副作用

提取器（出现在`rule`s的左手边的匹配器）不应该有副作用。如果提取器执行了副作用，可能会导致难以理解的远程行动错误。因此，任何可能的副作用都应该被打包成一个可以被单独构造的类型，然后在规则的右手边被调用。

### ISLE代码应利用类型

ISLE是一种类型化语言，应该利用这一点来防止可能的错误。例如，使用`with_flags`系列助手函数来配对产生标志的指令和消耗标志的指令，确保不会在标志使用指令之间插入任何错误的指令。

### 隐式类型转换

ISLE支持隐式类型转换，并在可能的情况下使用它们来简化降低规则。例如，`Value`和`ValueRegs`之间的转换，ISLE编译器会自动插入必要的转换。

### 隐式类型转换和副作用

虽然隐式转换非常方便，但需要小心管理它们引入的不可见副作用。特别是，隐式转换可能发生多次，因此定义的转换必须是*幂等的*，即如果多次调用必须返回相同的值。

### 总结

ISLE作为一种领域特定语言，为Cranelift编译器提供了一种高效、类型安全的方式来编写指令选择和重写规则。通过将ISLE源代码编译成Rust代码，Cranelift能够利用这些规则来进行指令降低和优化。
ISLE的设计允许开发者以声明式的方式编写规则，同时保持了代码的清晰性和可维护性。此外，ISLE的类型系统和隐式转换功能进一步简化了规则的编写，使得开发者可以更专注于编译器逻辑本身，而不是底层实现细节。

