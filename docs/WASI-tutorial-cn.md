这个文件是一个 WASI（WebAssembly System Interface）教程，它分为两部分：

### 第一部分：使用 WASI 运行常见语言
这部分介绍了如何将 C 和 Rust 程序编译为 WASI，并使用 `wasmtime` 运行时执行编译后的 WebAssembly 模块。

#### 从 C 编译到 WASI
提供了一个简单的 C 程序示例，该程序执行文件复制操作。这个示例展示了如何编译和运行程序，以及执行简单的沙箱配置。C 代码使用了标准的 POSIX API，并且对 WASI、WebAssembly 或沙箱没有任何了解。

示例代码被保存在 `demo.c` 文件中，使用 `wasi-sdk` 提供的 `clang` 编译器进行编译，编译命令如下：
```shell
$ clang demo.c -o demo.wasm
```
如果 `wasi-sdk` 没有安装在默认路径，可以通过指定 `--sysroot` 参数来编译：
```shell
$ clang demo.c --sysroot <path to sysroot> -o demo.wasm
```

#### 从 Rust 编译到 WASI
同样的效果也可以通过 Rust 实现。首先创建一个新的二进制 crate，然后编写 Rust 代码并将其保存在 `src/main.rs` 文件中。使用以下命令安装支持 WASI 的 Rust 工具链并构建项目：
```shell
$ rustup target add wasm32-wasip1
$ cargo build --target wasm32-wasip1
```
构建后的 WebAssembly 模块将位于 `target/wasm32-wasip1/debug` 目录中。

### 第二部分：在 Wasmtime 中执行
编译后的 WebAssembly 模块 `demo.wasm` 可以直接使用 `wasmtime` 运行，如下所示：
```shell
$ wasmtime demo.wasm
```
如果程序需要命令行参数，可以提供它们：
```shell
$ echo hello world > test.txt
$ wasmtime demo.wasm test.txt /tmp/somewhere.txt
```
`wasmtime` 使用 `--dir=` 选项来预打开目录，并将其作为能力提供给程序，以便在该目录中打开文件。

### WebAssembly 文本示例
这部分讨论了如何将 WebAssembly 文本格式编译为 wasm，并使用 `wasmtime` 运行时执行编译后的 WebAssembly 模块。示例代码使用 WASI 的 `fd_write` 实现将 `hello world` 写入标准输出。

首先，创建一个 `demo.wat` 文件，然后使用 `wasmtime` 直接执行 `.wat` 文件，或者使用 [wabt] 命令行工具将 `.wat` 转换为 `.wasm` 二进制格式，然后使用 `wasmtime` 执行。

对于浏览器中的示例运行，可以将编译后的 `.wasm` 文件上传到 [WASI browser polyfill]。

---

**总结：**
这个教程详细介绍了如何将 C 和 Rust 程序编译为 WASI 兼容的 WebAssembly 模块，并使用 `wasmtime` 运行时执行这些模块。
它还介绍了如何使用 WASI 的系统调用接口来操作文件，并展示了如何通过 `wasmtime` 的能力模型来控制对文件系统的访问。
此外，还提供了一个 WebAssembly 文本格式的示例，说明了如何将其编译为可执行的 WebAssembly 模块，并在 `wasmtime` 中运行。
这个教程为开发者提供了一个了解和使用 WASI 进行 WebAssembly 程序开发的实践指南。
