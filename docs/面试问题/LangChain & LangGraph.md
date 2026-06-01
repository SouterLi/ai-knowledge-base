## 在实现Agent这个场景中，Langgraph和LangChain有哪些区别？

在实现React范式的Agent这个场景中，二者都可以实现，但是LangChain有高度封装，默认就是ReAct范式，实现起来非常简单，但是要想改成其他范式，就不能用LangChain实现了。

但是LangChain最新推出一个新包，DeepAgents，这个新的包可以看作官方对Plan-and-Execute范式的封装。也就是说LangChain是经过封装的，你需要哪种范式，就要看官方没有没有给你实现相应的包，有的话直接用，很简单，没有的话，就得等官方推出。

LangGraph相比Langchain更加灵活，你可以手动写逻辑来实现ReAct范式或者Plan-and-Execute范式，但相对应的，LangGraph需要手动实现，用起来比较复杂。

### LangGraph实现ReAct范式的流程

1.定义State
2.定义tools，并将tool放到ToolNode里
3.初始化模型，并将tools绑定到模型中
4.定义模型调用Node
5.定义一个条件方法，接收模型的思考结果，判断下一步是结束还是继续调用工具
6.初始化图，并将模型Node和ToolNode放进去
7.设置入口
8.添加条件边，以ReAct为例，需要添加一个普通边，连接tool和agent，再添加一个条件边，连接agent和条件方法
9.编译图，返回一个app
10.调用和测试