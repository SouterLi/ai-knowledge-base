## 简单介绍一下DeepAgent

deepagent是langchain开发的一款Agent智能体实现框架。

它类似于langchian，都是基于LangGraph作为底层实现的，deepagent默认实现了plan-and-execute范式，先进行规划，再按照计划一步一步完成。

deepagent还有3个比较突出的能力：
1.todo list，在开始执行之前，deepagent会生成一个todo list，将复杂任务拆解为一个个可以完成的小目标；
2.上下文与记忆管理，deepagent提供了一个虚拟文件管理系统，可以将上下文记忆保存在文件中，只在上下文保留文件的描述和地址，如果有需要再从文件中取；
3.子Agent，deepagent可以创建并委托给子Agent，子Agent有独立的上下文空间。

## DeepAgent是如何管理上下文的？

deepagent使用一个虚拟文件管理系统来管理上下文。

对于用户输入的或者工具返回的大文件，DeepAgent不会放到上下文中，而是直接存到文件系统中，根据后续的需求再读取文件；

当上下文长度到达一定比例，DeepAgent会对上下文进行压缩摘要，并将对话原文保存到文件系统中，在摘要中保存了原文的链接，方便后续读取；

子Agent的上下文单独保存，当子Agent任务结束后，就将上下文全部删掉。