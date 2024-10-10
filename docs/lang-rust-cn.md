# 使用 Rust 执行 WebAssembly

本文档展示了如何使用 [Rust API][apidoc] 嵌入 Wasmtime 来执行一个简单的 wasm 程序。请确保也查看[完整的 API 文档][apidoc]，了解 [`wasmtime` crate][wasmtime] 提供的所有功能。

[apidoc]: https://bytecodealliance.github.io/wasmtime/api/wasmtime/
[wasmtime]: https://crates.io/crates/wasmtime

## 创建要执行的 WebAssembly

我们将假设你已经有现成的 wasm 文件。为了简化，我们还假设你有一个 `hello.wat` 文件，内容如下：

```wat
(module
  (func (export "answer") (result i32)
     i32.const 42
  )
)
```

这里我们只导出一个返回整数的函数，我们将从 Rust 中读取这个值。

## Hello, World!

首先，我们创建一个 Rust 项目：

```sh
$ cargo new --bin wasmtime_hello
$ cd wasmtime_hello
```

接下来，将 `hello.wat` 添加到项目的根目录。

我们将使用 `wasmtime` crate 来运行 wasm 文件。请执行命令 `cargo add wasmtime` 以使用 crate 的最新版本。`Cargo.toml` 文件中的 `dependencies` 块将如下所示：

```toml
[dependencies]
wasmtime = "19.0.0"
```

接下来，我们编写执行此 wasm 文件所需的代码。最简单的版本如下：

```rust,no_run
# extern crate wasmtime;
use std::error::Error;
use wasmtime::*;

fn main() -> Result<(), Box<dyn Error>> {
    // `Engine` 存储并配置全局编译设置，如优化级别、启用的 wasm 功能等。
    let engine = Engine::default();

    // 我们首先创建一个 `Module`，它表示输入 wasm 模块的编译形式。在本例中，将在解析文本格式后 JIT 编译。
    let module = Module::from_file(&engine, "hello.wat")?;

    // `Store` 是将拥有实例、函数、全局变量等的容器。所有的 wasm 项目都存储在 `Store` 中，我们将始终使用它与 wasm 世界交互。可以在商店中存储自定义数据，但我们现在只使用 `()`。
    let mut store = Store::new(&engine, ());

    // 有了编译好的 `Module`，我们可以实例化它，创建一个 `Instance`，我们可以实际操作函数。
    let instance = Instance::new(&mut store, &module, &[])?;

    // `Instance` 允许我们访问各种导出的函数和项目，我们在这里访问 `answer` 导出的函数并运行它。
    let answer = instance.get_func(&mut store, "answer")
        .expect("`answer` was not an exported function");

    // 有几种方法可以调用 `answer` `Func` 值。最简单的是使用 `typed` 静态断言其签名（在本例中断言它不接受参数并返回一个 i32），然后调用它。
    let answer = answer.typed::<(), i32>(&store)?;

    // 最后，我们可以调用我们的函数！注意使用 `?` 进行错误传播是为了处理 wasm 函数陷入的情况。
    let result = answer.call(&mut store, ())?;
    println!("Answer: {:?}", result);
    Ok(())
}
```

我们可以使用 `cargo run` 构建并执行示例。请注意，依赖于 `wasmtime` 意味着你依赖于 JIT 编译器，因此可能需要一些时间来构建其所有依赖项：

```sh
$ cargo run
  Compiling ...
  ...
   Finished dev [unoptimized + debuginfo] target(s) in 42.32s
    Running `wasmtime_hello/target/debug/wasmtime_hello`
Answer: 42
```

就这样！我们现在在 `wasmtime` 中执行了我们的第一个 WebAssembly 并得到了结果。

## 导入主机功能

我们刚刚看到的是如何调用 wasm 函数并查看结果的一个小例子。然而，大多数有趣的 wasm 模块将导入一些函数以做一些更有趣的事情。为此，你需要从 Rust 为 wasm 调用提供导入函数！

让我们看一个导入日志记录函数以及一些简单算术的 wasm 模块。

```wat
(module
  (import "" "log" (func $log (param i32)))
  (import "" "double" (func $double (param i32) (result i32)))
  (func (export "run")
    i32.const 0
    call $log
    i32.const 1
    call $log
    i32.const 2
    call $double
    call $log
  )
)
```

这个 wasm 模块将多次调用我们的 `"log"` 导入，然后调用 `"double"` 导入。我们可以使用以下代码编译和实例化此模块：

```rust,no_run
# extern crate wasmtime;
use std::error::Error;
use wasmtime::*;

struct Log {
    integers_logged: Vec<u32>,
}

fn main() -> Result<(), Box<dyn Error>> {
    let engine = Engine::default();
    let module = Module::from_file(&engine, "hello.wat")?;

    // 对于主机提供的函数，建议使用 `Linker`，它根据名称解析函数。
    let mut linker = Linker::new(&engine);

    // 首先，我们创建一个简单的 "double" 函数，它只会将其输入乘以二并返回。
    linker.func_wrap("", "double", |param: i32| param * 2)?;

    // 接下来，我们定义一个 `log` 函数。注意，我们使用 Wasmtime 提供的 `Caller` 参数来访问 `Store` 上的状态，这允许我们记录日志信息。
    linker.func_wrap("", "log", |mut caller: Caller<'_, Log>, param: u32| {
        println!("log: {}", param);
        caller.data_mut().integers_logged.push(param);
    })?;

    // 如上所述，实例化总是在 `Store` 内进行。这意味着要使用我们的 `Linker` 实例化，我们需要创建一个商店。注意，我们还在这里用自定义数据初始化商店。
    //
    // 之后我们使用 `linker` 创建实例。
    let data = Log { integers_logged: Vec::new() };
    let mut store = Store::new(&engine, data);
    let instance = linker.instantiate(&mut store, &module)?;

    // 如前所述，我们可以获取 run 函数并执行它。
    let run = instance.get_typed_func::<(), ()>(&mut store, "run")?;
    run.call(&mut store, ())?;

    // 我们还可以检查记录了哪些整数：
    println!("logged integers: {:?}", store.data().integers_logged);

    Ok(())
}
```

请注意，有多种方法可以定义一个 `Func`，请务必[查阅其文档][`Func`]了解其他创建主机定义函数的方法。

[`Func`]: https://bytecodealliance.github.io/wasmtime/api/wasmtime/struct.Func.html 

---

总结：
本文展示了如何使用 Rust 中的 Wasmtime API 来执行 WebAssembly 程序。首先，我们创建了一个 Rust 项目，并添加了 `wasmtime` 作为依赖项。然后，我们编写了 Rust 代码来加载和实例化一个简单的 wasm 模块，并调用其导出的函数。此外，我们还探讨了如何为 wasm 模块提供导入的函数，这使得 wasm 模块能够调用 Rust 中定义的函数。通过这些步骤，我们能够在 Rust 中嵌入和执行 WebAssembly 代码，展示了 Wasmtime 在 WebAssembly 生态系统中的实用性和灵活性。
