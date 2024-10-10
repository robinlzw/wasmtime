# 测试 Cranelift

Cranelift 在多个抽象和集成级别上进行测试。在可能的情况下，使用 Rust 单元测试来验证单个函数和类型。当测试编译器传递之间的交互时，文件级测试是合适的。

## Rust 测试

Rust 和 Cargo 对测试有很好的支持。Cranelift 在适当的情况下使用单元测试、文档测试和集成测试。[Rust By Example 页面上的测试](https://doc.rust-lang.org/rust-by-example/testing.html) 是如何编写这些测试形式的一个很好的示例。

## 文件测试

编译器使用大型数据结构来表示程序，生成测试数据很快就会变得难以处理。文件级测试使得为编译器测试提供大量的输入函数变得更加容易。

文件测试是位于 `filetests/` 目录层次结构中的 `*.clif` 文件。每个文件都有一个描述要测试内容的头部，后面跟着一些以 Cranelift 文本中间表示形式编写的输入函数：

```plaintext
test_file     : test_header `function_list`
test_header   : test_commands (`isa_specs` | `settings`)
test_commands : test_command { test_command }
test_command  : "test" test_name { option } "\n"
```

可用的测试命令如下所述。

许多测试命令只在目标指令集架构的上下文中才有意义。这些测试需要在测试头部中指定一个或多个 ISA 规范：

```plaintext
isa_specs     : { [`settings`] isa_spec }
isa_spec      : "isa" isa_name { `option` } "\n"
```

在 `isa` 行上给出的选项会修改 `cranelift-codegen/meta-python/isa/*/settings.py` 中定义的 ISA 特定设置。

所有类型的测试都允许修改共享的 Cranelift 设置：

```plaintext
settings      : { setting }
setting       : "set" { option } "\n"
option        : flag | setting "=" value
```

所有目标 ISA 可用的共享设置在 `cranelift-codegen/meta-python/base/settings.py` 中定义。

`set` 行会累积应用设置：

```plaintext
test legalizer
set opt_level=best
set is_pic=1
target riscv64
set is_pic=0
target riscv32 supports_m=false

function %foo() {}
```

这个例子将两次运行合法化器测试。两次运行都会有 `opt_level=best`，但它们会有不同的 `is_pic` 设置。32位运行还将禁用特定于 RISC-V 的标志 `supports_m`。

文件测试是 `cargo test` 的一部分，并且也可以使用 `clif-util test` 命令手动运行。

默认情况下，测试运行器会创建一个线程池，线程数量与逻辑 CPU 的数量相同。你可以通过 `CRANELIFT_FILETESTS_THREADS` 环境变量显式控制生成的线程数量。例如，要将测试运行器限制为单线程，请使用：

```plaintext
$ CRANELIFT_FILETESTS_THREADS=1 clif-util test path/to/file.clif
```

### Filecheck

下面描述的许多测试命令使用 *filecheck* 来验证它们的输出。Filecheck 是与同名 LLVM 工具相同的 Rust 实现。有关其语法的详细信息，请参见 [`documentation`](https://docs.rs/filecheck/)。

`.clif` 文件中的注释与它们跟随的实体相关联。这通常意味着一个指令或整个函数。使用 filecheck 的那些测试会提取与每个函数（或其实体）相关联的注释，并扫描它们以查找 filecheck 指令。然后，每个函数的测试输出与该函数的 filecheck 指令进行匹配。

出现在第一个函数之前的注释适用于每个函数。这在定义具有 `regex:` 指令的公共正则表达式变量时很有用，例如。

请注意，LLVM 的文件测试不会根据它们关联的函数来分离 filecheck 指令。它验证所有 filecheck 指令的连接输出。LLVM 的 `FileCheck` 命令有一个 `CHECK-LABEL:` 指令，以帮助分离不同函数的输出。Cranelift 的测试不需要这个。

### `test cat`

这是最简单的文件测试之一，用于测试转换为文本 IR 以及从文本 IR 转换回来。`test cat` 命令只是解析每个函数，然后再次将其转换为文本。然后，每个函数的文本与相关的 filecheck 指令进行匹配。

示例：

```plaintext
function %r1() -> i32, f32 {
block1:
    v10 = iconst.i32 3
    v20 = f32const 0.0
    return v10, v20
}
; sameln: function %r1() -> i32, f32 {
; nextln: block0:
; nextln:     v10 = iconst.i32 3
; nextln:     v20 = f32const 0.0
; nextln:     return v10, v20
; nextln: }
```

### `test verifier`

将每个函数通过 IR 验证器运行，并检查它是否产生了预期的错误消息。

预期的错误消息是在产生验证器错误的指令上用 `error:` 指令表示的。验证器错误的消息和报告的错误位置都会被验证：

```plaintext
test verifier

function %test(i32) {
    block0(v0: i32):
        jump block1       ; error: terminator
        return
}
```

如果一个函数不包含 `error:` 注释，如果函数验证正确，则测试通过。

### `test print-cfg`

打印每个函数的控制流图作为 Graphviz 图形，并对结果运行 filecheck。另见 `clif-util print-cfg` 命令：

```plaintext
; For testing cfg generation. This code is nonsense.
test print-cfg
test verifier

function %nonsense(i32, i32) -> f32 {
; check: digraph %nonsense {
; regex: I=\binst\d+\b
; check: label="{block0 | <$(BRIF=$I)>brif v1, block1(v2), block2 }"]
    
block0(v0: i32, v1: i32):
    v2 = iconst.i32 0
    brif v1, block1(v2), block2  ; unordered: block0:$BRIF -> block1
                                 ; unordered: block0:$BRIF -> block2

block1(v5: i32):
    return v0

block2:
    v100 = f32const 0.0
    return v100
}
```

### `test domtree`

计算每个函数的支配树，并验证它与 `dominates:` 注释是否一致：

```plaintext
test domtree

function %test(i32) {
    block0(v0: i32):
        jump block1              ; dominates: block1
    block1:
        brif v0, block2, block3  ; dominates: block2, block3
    block2:
        jump block3
    block3:
        return
}
```

每个可到达的基本块都有一个 *直接支配者*，它是一个跳转或分支指令。如果直接支配指令上的 `dominates:` 注释既正确又完整，这个测试就会通过。

这个测试还将计算出的 CFG 后序通过 filecheck 发送。

### `test legalizer`

为指定的目标 ISA 合法化每个函数，并将结果函数通过 filecheck 运行。这个测试命令可以用来验证为合法指令选择的编码以及合法化器执行的指令转换。

### `test regalloc`

测试寄存器分配器。

首先，为指定的目标 ISA 合法化每个函数。这对于寄存器分配是必需的，因为指令编码为寄存器分配器提供了寄存器类约束。

其次，对函数运行寄存器分配器，插入溢出代码，并将寄存器和堆栈插槽分配给所有值。

然后，结果函数通过 filecheck 运行。

### `test simple-gvn`

测试简单的 GVN 传递。

在每个函数上运行简单的 GVN 传递，然后将结果通过 filecheck 运行。

### `test licm`

测试 LICM 传递。

在每个函数上运行 LICM 传递，然后将结果通过 filecheck 运行。

### `test dce`

测试 DCE 传递。

在每个函数上运行 DCE 传递，然后将结果通过 filecheck 运行。

### `test shrink`

测试指令收缩传递。

在每个函数上运行收缩传递，然后将结果通过 filecheck 运行。

### `test simple_preopt`

测试预优化传递。

在每个函数上运行预优化传递，然后将结果通过 filecheck 运行。

### `test compile`

测试整个代码生成管道。

每个函数都通过完整的 `Context::compile()` 函数传递，这通常用于编译代码。这种类型的测试通常依赖于断言或验证器错误，但也可以使用方法 check 指令，这些指令将与 Cranelift IR 的最终形式匹配，就在二进制机器代码发射之前。

### `test run`

编译并执行一个函数。

这个测试命令允许几个指令：
- 要打印运行函数的结果到 stdout，请添加一个 `print` 指令，并调用前面的函数与参数（见下面的 `%foo`）；如果通过 Cargo 运行这些测试，请记住启用 `--nocapture`
- 要检查函数的结果，请添加一个 `run` 指令，并调用前面的函数与比较（`==` 或 `!=`）（见下面的 `%bar`）
- 为了向后兼容，要检查具有 `() -> i*` 签名的函数的结果，只需要 `run` 指令，无需调用或比较（见下面的 `%baz`）；非零值被解释为成功的测试执行，而零值被解释为失败的测试执行。

目前需要一个 `target`，但只用于指示主机平台是否可以运行测试，目前只过滤架构。将使用主机平台的原生目标来实际编译测试。

示例：

```plaintext
test run
target x86_64

; 如何打印函数的结果
function %foo() -> i32 {
block0:
    v0 = iconst.i32 42
    return v0
}
; print: %foo()

; 如何检查函数的结果
function %bar(i32) -> i32 {
block0(v0:i32):
    v1 = iadd_imm v0, 1
    return v1
}
; run: %bar(1) == 2

; 检查函数结果的传统方法
function %baz() -> i8 {
block0:
    v0 = iconst.i8 1
    return v0
}
; run
```

### 解释和总结

Cranelift 是一个底层代码生成库，用于编译器基础设施，它提供了多种测试方法来确保代码的质量和正确性。这些测试方法包括单元测试、文档测试、集成测试和文件测试，涵盖了从单个函数到整个代码生成流程的各个方面。

文件测试特别有用，因为它们允许测试大型数据结构，这些数据结构在程序表示中很常见，而且很难通过程序生成。通过使用 `.clif` 文件，Cranelift 可以轻松地为编译器测试提供大量的输入函数。

测试命令如 `test cat`、`test verifier`、`test print-cfg`、`test domtree`、`test legalizer`、`test regalloc`、`test simple-gvn`、`test licm`、`test dce`、`test shrink`、`test simple_preopt` 和 `test compile` 都有助于在不同阶段验证 Cranelift 的行为。此外，`test run` 命令允许实际编译和执行函数，这对于验证生成的机器代码的正确性至关重要。

总的来说，Cranelift 的测试策略是全面且多层次的，确保了代码生成过程的每个部分都经过了严格的测试和验证。
