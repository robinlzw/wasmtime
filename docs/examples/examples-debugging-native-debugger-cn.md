这篇文章介绍了如何同时使用`gdb`或`lldb`调试Wasm虚拟机和宿主（即Wasmtime CLI或你的嵌入Wasmtime程序）。

### 调试步骤：

1. **编译WebAssembly以启用调试信息**，通常使用`-g`选项；例如：
    ```sh
    clang foo.c -g -o foo.wasm
    ```

2. **运行Wasmtime并启用调试信息**；这是CLI中的`-D debug-info`和嵌入中的`Config::debug_info(true)`（例如，参见[Rust嵌入中的调试](./examples-rust-debugging.md)）。如果需要，还建议使用`-O opt-level=0`以更好地检查局部变量。

3. **使用支持的调试器**：
    ```sh
    lldb -- wasmtime run -D debug-info foo.wasm
    ```
    ```sh
    gdb --args wasmtime run -D debug-info -O opt-level=0 foo.wasm
    ```

### 遇到问题时的帮助：

- 在MacOS使用LLDB时，你可能需要运行：`settings set plugin.jit-loader.gdb.enable on` ([#1953](https://github.com/bytecodealliance/wasmtime/issues/1953)) 

- 使用LLDB时，调用`__vmctx.set()`来设置当前上下文，然后再调用任何解引用操作符 ([#1482](https://github.com/bytecodealliance/wasmtime/issues/1482))：
  ```sh
  (lldb) p __vmctx->set()
  (lldb) p *foo
  ```

- 实例内存起始地址可以在`__vmctx->memory`中找到。

- 在Windows上，你可能会遇到在调试器下默认启用的额外原生堆检查导致的WASM编译吞吐量下降。你可以通过设置环境变量`_NO_DEBUG_HEAP`为`1`来禁用它们。

### 讲解和总结：

调试WebAssembly应用程序时，能够同时调试Wasm虚拟机和宿主代码是非常重要的。这允许开发者深入了解虚拟机的运行状态以及宿主程序的行为。通过使用`gdb`或`lldb`，开发者可以设置断点、检查变量和内存状态，以及单步执行代码，从而更有效地诊断问题。

在调试过程中，确保编译Wasm模块时启用了调试信息，并在运行Wasmtime时也启用了调试信息。选择使用`gdb`或`lldb`作为调试器，并根据你的操作系统和环境进行相应的配置。例如，在MacOS上使用LLDB时，可能需要启用特定的插件，而在Windows上则可能需要调整环境变量以优化性能。

如果在调试过程中遇到问题，可以参考Wasmtime的GitHub问题跟踪器中的相关讨论，这些讨论可能提供了解决方案或有用的信息。此外，确保你的调试器设置正确，例如在LLDB中设置当前上下文，这对于正确地调试Wasm代码至关重要。

总的来说，通过遵循这些步骤和建议，你可以更有效地调试Wasmtime应用程序，无论是在开发新功能还是在解决现有问题时。
