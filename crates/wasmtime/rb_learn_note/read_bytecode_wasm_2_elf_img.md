---
```rust
    /// Publishes the internal ELF image to be ready for execution.
    ///
    /// This method can only be called once and will panic if called twice. This
    /// will parse the ELF image from the original `MmapVec` and do everything
    /// necessary to get it ready for execution, including:
    ///
    /// * Change page protections from read/write to read/execute.
    /// * Register unwinding information with the OS
    ///
    /// After this function executes all JIT code should be ready to execute.
    pub fn publish(&mut self) -> Result<()> {
        assert!(!self.published);
        self.published = true;
```		
		
---
```rust
    pub(crate) fn load_code(
        &self,
        mmap: crate::runtime::vm::MmapVec,
        expected: ObjectKind,
    ) -> Result<Arc<crate::CodeMemory>> {
        serialization::check_compatible(self, &mmap, expected)?;
        let mut code = crate::CodeMemory::new(mmap)?;
        code.publish()?;
        Ok(Arc::new(code))
    }
```

---
```rust
    /// Loads a `CodeMemory` from the specified in-memory slice, copying it to a
    /// uniquely owned mmap.
    ///
    /// The `expected` marker here is whether the bytes are expected to be a
    /// precompiled module or a component.
    pub(crate) fn load_code_bytes(
        &self,
        bytes: &[u8],
        expected: ObjectKind,
    ) -> Result<Arc<crate::CodeMemory>> {
        self.load_code(crate::runtime::vm::MmapVec::from_slice(bytes)?, expected)
    }
```

---
```rust
impl<'a> CodeBuilder<'a> {
    fn compile_cached<T>(
        &self,
        build_artifacts: fn(&Engine, &[u8], Option<&[u8]>) -> Result<(MmapVecWrapper, Option<T>)>,
    ) -> Result<(Arc<CodeMemory>, Option<T>)> {
// ...
let code = engine.0.load_code_bytes(&serialized_bytes, kind).ok()?;
// ...
}
```

---
```rust
    /// Same as [`CodeBuilder::compile_module_serialized`] except that a
    /// [`Module`](crate::Module) is produced instead.
    ///
    /// Note that this method will cache compilations if the `cache` feature is
    /// enabled and turned on in [`Config`](crate::Config).
    pub fn compile_module(&self) -> Result<Module> {
        let (code, info_and_types) = self.compile_cached(super::build_artifacts)?;
        Module::from_parts(self.engine, code, info_and_types)
    }
```

---
```rust
    /// Creates a new WebAssembly `Module` from the contents of the given
    /// `file` on disk.
    ///
    /// This is a convenience function that will read the `file` provided and
    /// pass the bytes to the [`Module::new`] function. For more information
    /// see [`Module::new`]
    ///
    /// # Examples
    ///
    /// ```no_run
    /// # use wasmtime::*;
    /// # fn main() -> anyhow::Result<()> {
    /// let engine = Engine::default();
    /// let module = Module::from_file(&engine, "./path/to/foo.wasm")?;
    /// # Ok(())
    /// # }
    /// ```
    ///
    /// The `.wat` text format is also supported:
    ///
    /// ```no_run
    /// # use wasmtime::*;
    /// # fn main() -> anyhow::Result<()> {
    /// # let engine = Engine::default();
    /// let module = Module::from_file(&engine, "./path/to/foo.wat")?;
    /// # Ok(())
    /// # }
    /// ```
    #[cfg(all(feature = "std", any(feature = "cranelift", feature = "winch")))]
    pub fn from_file(engine: &Engine, file: impl AsRef<Path>) -> Result<Module> {
        crate::CodeBuilder::new(engine)
            .wasm_binary_or_text_file(file.as_ref())?
            .compile_module()
    }
```
	
---
```rust
/// 发布内部 ELF 镜像以准备执行。
///
/// 该方法只能调用一次，如果调用两次则会导致 panic。此方法将从原始的 `MmapVec` 中解析
/// ELF 镜像，并完成准备执行所需的所有工作，包括：
///
/// * 将页面保护从读/写更改为读/执行。
/// * 向操作系统注册展开信息。
///
/// 此函数执行后，所有 JIT 代码应准备就绪以供执行。
pub fn publish(&mut self) -> Result<()> {
    assert!(!self.published);
    self.published = true;

    if self.text().is_empty() {
        return Ok(());
    }

    // 这里的不安全性来自几个方面：
    //
    // * 我们实际上在更新一些页面保护以使其可执行内存。
    //
    // * 我们在注册展开信息，这依赖于信息的正确性。 
    //   这适用于实际的展开表以及我们传入的指针的有效性。
    unsafe {
        // 首先，如果必要，应用重定位。这可以发生在像 libcalls 这样的地方，
        // 这些调用在降低过程中发生，且没有通过通过 `VMContext` 间接的
        // 基于 Wasm 的 libcalls 层。请注意，大多数模块不会有重定位，
        // 所以这通常不会做任何事情。
        self.apply_relocations()?; 

        // 接下来，通过将此映像的所有内存设置为只读来冻结其内容。
        // 从这一点开始，不应再修改任何内容，因此提交所有内容。
        // 对于编译到内存中的映像，这将意味着向其他核心发送 IPI 
        // 以驱逐可写映射。对于从磁盘加载的映像，只要没有任何重定位，
        // 则不应导致 IPI，因为在任何时刻都不应有其他写入映像的操作。
        self.mmap.make_readonly(0..self.mmap.len())?; 

        let text = self.text();

        // 如果处理器需要，清除新分配的代码缓存。
        //
        // 在将内存标记为 R+X 之前执行此操作，从技术上讲，我们应该能够在之后进行，但
        // 有些 CPU 在对只读内存执行此操作时存在缺陷。
        icache_coherence::clear_cache(text.as_ptr().cast(), text.len())
            .expect("缓存清除失败");

        // 将可执行部分从只读切换到可读/可执行。
        self.mmap
            .make_executable(self.text.clone(), self.enable_branch_protection)
            .context("无法使内存可执行")?; 

        // 清除管道中的任何正在进行的指令。
        icache_coherence::pipeline_flush_mt().expect("管道清除失败");

        // 在设置好所有内存后，使用平台特定的 `UnwindRegistration` 实现来通知
        // 一般运行时，我们的所有刚发布的 JIT 函数都有可用的展开信息。
        self.register_unwind_info()?;
    }

    Ok(())
}
```