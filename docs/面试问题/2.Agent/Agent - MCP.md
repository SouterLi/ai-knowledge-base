## mcp和本地方法的比较重要的区别有哪些？

1.mcp可以写在任何项目中，只要暴露出来mcp server，我们作为调用方就可以调用，本地方法就只能写在同一个项目中，也就是说mcp利于解耦；
2.MCP是语言无关的，你可以用另一种语言写mcp server；
3.mcp对接不同大模型不需要单独适配，而本地方法需要进行适配；
4.mcp对于调用方来说不需要写工具描述，因为在mcp提供方就已经写好了；

## 作为mcp 服务提供方，你要如何发布mcp服务给别人用？

推荐使用**Streamable HTTP**的方式部署到内网服务器中。

比如使用Python的fastmcp包：
```python
from fastmcp import FastMCP

mcp = FastMCP("billing-service")

@mcp.tool()
def query_invoice(customer_id: str, month: str) -> str:
	// 具体功能

mcp.run(transport="http", host="0.0.0.0", port=8000)
```

用上面这种方式，将mcp服务配置到8000端口
然后把整个代码部署到服务器上，部署完成后，就可以得到一个内网访问地址
类似：- `http://mcp-billing.internal.yourcompany.com:8000/mcp`

然后调用方就可以调用了。

## MCP在Agent的调用过程和本地方法有什么区别？

作为接口调用方，你只需要把mcp添加到Agent的tools里，就不需要做别的了，Agent会自动识别tools，根据问题自动决定调用哪个tool，并且自动调用；

而本地方法就麻烦很多，你把方法注册到tools里后，大模型只会把想调用的方法名和参数给你，你需要自己去调用具体的方法，拿到结果后，你还需要自己把结果拼到Prompt中，给大模型。