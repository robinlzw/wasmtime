这篇文章是关于如何为Wasmtime和Cranelift项目做出贡献的指南。以下是文章的主要内容翻译：

我们很高兴能和你一起在Wasmtime和/或Cranelift上工作！这本指南应该可以帮助你开始进行Wasmtime和Cranelift的开发。但首先，请确保你已经阅读了[行为准则](./contributing-coc.html)！

Wasmtime和Cranelift是非常雄心勃勃的项目，目标众多，虽然我们有信心能够实现其中的一些，但我们看到了许多人参与进来并帮助我们实现更多目标的机会。

## 加入我们的聊天

我们在Zulip上讨论Wasmtime和Cranelift的开发——[加入我们！](https://bytecodealliance.zulipchat.com/)。你也可以加入特定的流：

* [#wasmtime](https://bytecodealliance.zulipchat.com/#narrow/stream/217126-wasmtime)
* [#cranelift](https://bytecodealliance.zulipchat.com/#narrow/stream/217117-cranelift)

如果你在构建Wasmtime或Cranelift时遇到麻烦，不确定为什么测试失败，或者有其他问题，欢迎在Zulip上提问。并非我们希望用这些项目实现的所有内容都已经反映在代码或文档中，所以如果你看到似乎缺失或不合逻辑的东西，或者只是不符合你的预期，我们也有兴趣听听你的意见！

像往常一样，你也非常欢迎[提出一个问题](https://github.com/bytecodealliance/wasmtime/issues/new)！

最后，我们每两周举行一次Wasmtime和Cranelift的项目会议，在Zoom上进行。更多信息，请查看我们的[会议议程/记录仓库](https://github.com/bytecodealliance/meetings)。如果你有兴趣加入，请通过Zulip与我们联系！

## 寻找可以做的事情

如果你想找点事情做，这些是开始的好地方：

* [标记为“good first issue”的问题](https://github.com/bytecodealliance/wasmtime/labels/good%20first%20issue)——这些问题往往很简单，需要做什么很清楚，适合新贡献者来解决。目标是学习Wasmtime的开发工作流程，确保你能够构建和测试Wasmtime。

* [标记为“help wanted”的问题](https://github.com/bytecodealliance/wasmtime/labels/help%20wanted)——这些是我们希望得到一些帮助的问题！

如果你不确定一个问题是否适合你，可以在问题的评论中提问，或者在聊天中提问。

### 指导

我们很高兴指导人们，无论你是在学Rust，学习编译器后端，学习机器代码，学习wasm，学习Cranelift的做事方式，或者同时学习所有这些。

我们使用受[Rust问题标签](https://github.com/rust-lang/rust/blob/master/CONTRIBUTING.md#issue-triage)启发的标签方案在问题跟踪器中对问题进行分类。例如，[E-easy]标记适合初学者的问题，[E-rust]标记可能需要一些熟悉Rust的问题，但不一定需要Cranelift特定或甚至编译器特定的经验。[E-compiler-easy]标记适合对编译器有一定了解的初学者，或者有兴趣获得一些经验的人:-)。

也可以看看[Cranelift标签的完整列表](https://github.com/bytecodealliance/wasmtime/labels?q=cranelift)。

同时，我们鼓励人们四处看看，找他们感兴趣的事情。这是参与进来的好时机，因为还没有很多东西是一成不变的。
