Wasmtime 和 Cranelift 项目非常依赖持续集成（CI）来确保代码的持续运作和高质量。这个项目的 CI 设置相当复杂和广泛，因此值得在这里介绍它的组织方式和对贡献者的期望。

### CI 的组织和期望

1. **所有 CI 都在 GitHub Actions 上进行**，配置文件位于仓库的 `.github` 目录中。

2. **Pull Requests 和 CI**：
   - 每个 Pull Request（PR）都会运行一部分完整的 CI 测试套件。CI 的目的是快速捕捉大多数错误和问题。
   - 默认情况下，测试套件在 x86_64 Linux 上运行，但可能会根据 PR 修改的文件而变化。
   - PR 作者需要修复 CI 失败的问题，除非 CI 失败与 PR 无关。在这种情况下，应通知其他维护者。
   - 一些审查者可能会等到 PR 的 CI 变为绿色后才进行审查，因为否则可能表明需要进行更改。
   - 使用 GitHub 的 Merge Queue 功能合并 PR，进入 Merge Queue 需要 PR 的 CI 为绿色。
   - 进入 Merge Queue 后，PR 将执行完整的测试套件，可能会运行之前未在 PR 上运行的测试。

3. **CI 上运行的测试**：
   - 代码格式化：运行 `cargo fmt -- --check` 确保所有代码都使用 rustfmt 格式化。
   - 文档测试：在 CI 上测试文档中的代码片段，确保它们有效。
   - Crate 测试：执行类似 `cargo test --all` 和 `cargo test --all --release` 的命令，测试所有工作区 crates 的整个测试套件。
   - 模糊测试回归测试：从模糊测试语料库中随机取样并运行模糊测试。

4. **CI 产生的工件**：
   - 生成 Wasmtime 和 Cranelift 的二进制发行版和文档。
   - 包括 `wasmtime` CLI 的 tarballs、Wasmtime C API 的 tarballs、书籍和 API 文档。
   - 生成一个完全自包含的源代码 tarball，其中所有依赖项都已 vendored，因此不需要网络来构建。
   - 工件是完整 CI 套件的一部分，这意味着默认情况下 PR 不会产生工件，但可以通过 "prtest:full" 请求。所有通过 Merge Queue 的运行都会产生一整套工件。

如果你需要访问上述链接以获取更多信息，但遇到了网络问题，这可能是由于链接本身的问题或网络连接问题。请检查链接的合法性，并在必要时重试。如果问题仍然存在，可能需要联系项目的维护者或查看项目的文档以获取更多帮助。
