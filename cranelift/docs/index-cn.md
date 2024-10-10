# Cranelift 文档

## 杂项文档页面：

- [Cranelift IR](ir.md)
  Cranelift IR 是编译器大部分操作的数据结构。

- [Testing Cranelift](testing.md)
  本页记录了 Cranelift 的测试框架。

- [Cranelift 与 LLVM 比较](compare-llvm.md)
  LLVM 和 Cranelift 有相似之处也有不同之处。

## Cranelift 包文档：

- [cranelift](https://docs.rs/cranelift) 
   这是一个总括性的包，重新导出了代码生成和前端包，以便更容易使用。

- [cranelift-codegen](https://docs.rs/cranelift-codegen) 
   这是核心代码生成包。它以 Cranelift IR 为输入，并输出编码的机器指令以及符号重定位。

- [cranelift-codegen-meta](https://docs.rs/cranelift-codegen-meta) 
   这个包包含了代码生成器使用的元语言实用工具和描述。

- [cranelift-frontend](https://docs.rs/cranelift-frontend) 
   这个包提供了将代码转换为 Cranelift IR 的实用工具。

- [cranelift-native](https://docs.rs/cranelift-native) 
   这个包执行主机的自动检测，允许 Cranelift 生成针对其运行的机器优化的代码。

- [cranelift-reader](https://docs.rs/cranelift-reader) 
   这个包将 Cranelift IR 的文本格式转换为内存中的数据结构。

- [cranelift-module](https://docs.rs/cranelift-module) 
   这个包管理编译多个函数和数据对象。

- [cranelift-object](https://docs.rs/cranelift-object) 
   这个包为 `cranelift-module` 提供基于对象的后端，使用 [object](https://github.com/gimli-rs/object) 库输出原生对象文件。

- [cranelift-jit](https://docs.rs/cranelift-jit) 
   这个包为 `cranelift-module` 提供了一个 JIT 后端，将代码和数据发射到内存中。

### 解释和总结

Cranelift 是一个底层代码生成库，它提供了多种工具和实用程序，用于编译器的不同阶段。文档提供了关于 Cranelift IR、测试框架以及与 LLVM 的比较的详细信息。

Cranelift 的包文档详细介绍了每个组件的功能和用途：

- **cranelift**：作为一个总括性的包，它使得使用代码生成和前端包更加方便。
- **cranelift-codegen**：核心代码生成包，负责将 Cranelift IR 转换为机器指令。
- **cranelift-codegen-meta**：包含代码生成器使用的元语言工具和描述。
- **cranelift-frontend**：提供将代码转换为 Cranelift IR 的工具。
- **cranelift-native**：自动检测主机，以便生成针对当前机器优化的代码。
- **cranelift-reader**：将 Cranelift IR 文本格式转换为内存中的数据结构。
- **cranelift-module**：管理编译多个函数和数据对象。
- **cranelift-object**：提供基于对象的后端，输出原生对象文件。
- **cranelift-jit**：提供 JIT 后端，将代码和数据发射到内存中。

这些组件共同构成了 Cranelift 的生态系统，使其成为一个功能强大且灵活的代码生成工具。
