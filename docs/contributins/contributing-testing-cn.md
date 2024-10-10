这篇文章介绍了如何在Wasmtime项目中运行测试并添加新的测试。以下是文章的主要内容总结：

1. **安装`wasm32`目标**：
   - 使用`rustup`安装`wasm32-wasip1`和`wasm32-unknown-unknown`目标。

2. **运行测试**：
   - 使用不同的命令来运行测试，例如`cargo test`、`cargo test -p cranelift-tools`和`cargo test -p wasmtime-wasi`。
   - 要运行所有测试，执行`./ci/run-tests.sh`。

3. **测试特定crate**：
   - 使用`cargo test -p wasmtime-whatever`来测试特定的Wasmtime crate。

4. **运行Wasm规范测试**：
   - 确保已检出并初始化Wasm规范测试套件的git子模块。
   - 在仓库根目录执行`cargo test --test wast`来运行规范测试。

5. **运行WASI集成测试**：
   - 使用`cargo test -p wasmtime-wasi`来运行WASI集成测试。

6. **添加新测试**：
   - 对于单元测试，在同一`.rs`文件中添加`test`模块。
   - 对于集成测试，每个crate支持一个单独的`tests`目录。
   - 对于Wast风格的测试，添加到`tests/misc_testsuite`目录。
   - 对于WASI集成测试，将测试程序添加到`test-programs` crate。

文章还提到了如何为WASI实现添加特定的测试用例，以及如何将新的测试添加到规范测试套件中。

如果你需要访问上述链接以获取更多信息，但遇到了网络问题，这可能是由于链接本身的问题或网络连接问题。请检查链接的合法性，并在必要时重试。如果问题仍然存在，可能需要联系项目的维护者或查看项目的文档以获取更多帮助。
