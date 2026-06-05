### 如何用LangGraph实现ReAct范式？

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