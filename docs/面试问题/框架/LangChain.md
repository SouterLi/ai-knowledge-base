## 在实现Agent这个场景中，Langgraph和LangChain有哪些区别？

在实现React范式的Agent这个场景中，二者都可以实现，但是LangChain有高度封装，默认就是ReAct范式，实现起来非常简单，但是要想改成其他范式，就不能用LangChain实现了。

但是LangChain最新推出一个新包，DeepAgents，这个新的包可以看作官方对Plan-and-Execute范式的封装。也就是说LangChain是经过封装的，你需要哪种范式，就要看官方没有没有给你实现相应的包，有的话直接用，很简单，没有的话，就得等官方推出。

LangGraph相比Langchain更加灵活，你可以手动写逻辑来实现ReAct范式或者Plan-and-Execute范式，但相对应的，LangGraph需要手动实现，用起来比较复杂。

