工具集成是一个无底洞

### 没有MCP之前：重复造轮子的噩梦
假设正在开发一个Agent，需要它能读取Github代码仓，查询公司数据库，操作本地文件系统，还能发送Slack消息
每一个工具都需要自己来做：
- 研究工具API文档
- 将API封装为函数
- 给每个函数写Function Calling定义（name、description、parameters）
- 自己处理认证、错误处理、数据 格式转换
在 MCP 出现之前，行业里存在着经典的 **“M × N 适配难题”**： 假设市面上有 M 个大模型应用（ChatGPT, Claude, 各种定制 Agent），同时世界上有 N 种数据源（Slack, GitHub, 本地数据库, Notion）。以前，每个应用想要读取每种数据，都要单独写一套鉴权、API 调用和解析的“胶水代码”。
[[不同大模型对于Function Calling的差异]]
### MCP的出现：给AI工具世界定一个标准
MCP是Anthropic在24年11月推出的模型上下文协议(Model Context Protocol)
它要解决的核心问题：**把工具写好和用起来分离**

![[智能OnCall/assets/什么是mcp/file-20260508081322106.jpg]]
**MCP 的诞生改变了这一切。** 它制定了一套统一的通信标准。只要数据源提供方写好了一个“MCP 服务器”（就像做了一个 USB-C 转换头），那么**任何支持 MCP 标准的大模型应用**（客户端），都可以直接插上去使用这个数据源，做到真正的**即插即用 (Plug-and-Play)**。
### MCP的三大角色
![[智能OnCall/assets/什么是mcp/file-20260508081546139.jpg]]
MCP 采用了经典的 **客户端-服务器 (Client-Server)** 架构，它的结构极其精简而优雅，主要由以下核心部件构成：
#### 1. 物理结构
- **MCP Host（宿主应用）：** 比如 Claude Desktop 桌面端、或者你正在用的 IDE（如 Cursor、Windsurf）。这是用户直接交互的界面。
- **MCP Client（客户端）：** 嵌在宿主应用内部，负责向外发送标准化的请求。
- **MCP Server（服务端）：** 这是一个轻量级的本地或远程程序。它就像一个“翻译官”，一边通过 MCP 协议与客户端对话，另一边通过私有 API 与真实的数据源（如本地 SQLite 数据库、云端 GitHub）对接。
#### 2. 逻辑结构（三大核心能力）
MCP 规定了服务端必须通过三种标准格式向大模型暴露能力：
- **Resources (资源)：** 暴露给大模型的**只读数据**。就像给大模型塞入一张光盘。比如一段代码文件、一条数据库记录。
- **Tools (工具)：** 暴露给大模型的**可执行函数**。大模型可以通过它来改变世界（这其实就是标准化之后的 Function Calling）。比如：执行一段 SQL、给某人发消息。
- **Prompts (提示模板)：** 服务端可以预先写好一些标准的系统指令模板供客户端随时调用。


### MCP和Function Calling的关系
Function Calling 解决的问题是：大模型做出【要调用什么工具】的决策之后，怎么把这个决策用标准化JSON格式传给Agent。这是大模型和Agent之间的通信协议，负责【大模型怎么开口下指令】这一段
MCP解决的问题是：工具怎么被统一注册、统一发现、统一调用。这是Agent和工具服务之间的连接协议，负责【Agent怎么找到并执行工具】这一段。
两者的配合：

![[智能OnCall/assets/什么是mcp/file-20260508084124200.jpg]]

### MCP调用流程
以【帮我查一下GitHub上React仓库最近的Commit】为例
![[assets/不同大模型对于function calling的差异/file-20260508084908247.jpg]]

1. 用户提问：在Host(比如 Claude Desktop)里输入「帮我查一下 React 仓库最近的commit」
2. Host分析任务：大模型判断需要调用GitHub 工具，生成 Function Call 格式的调用指令
3. Client 接收指令：Host 把调用指令交给内置的 MCP Client
4. Client 路由到对应 Server： Client 根据工具名，找到负责GitHub能力的 MCP Server，把请求发过去
5. Server 执行：GitHub MCP Server 调用GitHub API，拿到最近的 commit 列表
6. 结果回传：Server 把结果按 MCP协议格式回传给Client， Client 转交给Host
7. Host 生成回复：大模型拿到结果，整理成自然语言回复给用户
整个过程中，Host和背后的大模型完全不需要知道GitHub API的任何细节，它只管说「我要调 GitHub 工具」，剩下的事情 MCP Server 全权负责。这就是「解耦」的价值：工具的实现细节，和AI的调用决策，完全分离。

	