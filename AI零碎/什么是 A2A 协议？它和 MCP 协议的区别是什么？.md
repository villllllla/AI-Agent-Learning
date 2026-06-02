
[# 快手Agent开发一面：什么是 A2A 协议？它和 MCP 协议的区别是什么？](https://mp.weixin.qq.com/s/9xvKH46At85OMc0PCOvKKQ)：
[# 构建开放智能体生态：AgentScope 如何用 A2A 协议与 Nacos 打通协作壁垒？](https://mp.weixin.qq.com/s/-pp43gOTkTtkuxAt_szFIw)
![](https://mmbiz.qpic.cn/mmbiz_jpg/yvBJb5IiafvkV37mxv4d4G3VnVtBydQSoib6Fq6BnqhodZoSnJtrRc2cLQeL6UxUTgPCbibb3zXMmpchTBcibIM9Rg/640?wx_fmt=jpeg&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=2)

#### 面试总结
面试回答这道题，第一个必须说清楚的点是定位差异：MCP 是 Agent 向下连工具和数据源，A2A 是多个 Agent 之间向外互相通信协作，一纵一横，各管一层。

第二个关键点是 A2A 的核心机制：Agent Card 实现能力声明和自动发现，Task 状态机支持异步长任务协作，本质上就是 Agent 世界的微服务架构。

还要说清楚「为什么需要 A2A」，也就是单 Agent 的天花板问题：工具数量有限、上下文窗口有限、专业能力有限，复杂任务需要拆分给不同的专业 Agent 并行处理。

最后一定要强调两者是互补关系而非竞争关系，在复杂的多 Agent 系统里通常同时在用，缺一不可。
#### 为什么单个 Agent 不够用，上下文和专业边界
一个 Agent 的本质是：一个 LLM + 一组工具 + 一段上下文窗口。这三个维度都有自己的天花板。
1. **工具数量的限制**：不可能给一个 Agent 装 100 个工具，模型处理起来效率极低，容易混乱。
2. **上下文窗口的限制**，复杂任务积累的中间产物，搜索结果、草稿、反思记录，会很快把窗口塞满，后面的生成根本顾不上前面写了什么。
3. **专业能力的限制**，同一个 Agent 既做代码审查又做市场分析，不如专门为各自任务配置或微调的 Agent 效果好。
举一个具体任务：「帮我做一份 AI 编程工具的竞品分析报告，要有行业趋势、技术对比、商业模式分析和 SWOT」。让单个 Agent 做这件事，问题是：搜索结果和草稿会把上下文撑满，等到写 SWOT 时，前面的行业趋势分析早就被挤出了有效注意力范围；而且市场调研和技术分析需要不同的知识侧重，一个 Agent 很难全面兼顾。

![](AI-Agent-Learning/AI零碎/assets/什么是%20a2a%20协议？它和%20mcp%20协议的区别是什么？/file-20260528071935049.jpg)

拆成多个 Agent 协作，上下文压力就真的变小了吗？

关键就在这里：调度 Agent 把「做行业趋势分析」委托给市场 Agent，市场 Agent 自己去搜几十个网页、写草稿、反复迭代，这些中间过程都在它自己的上下文里。任务做完，它只把最终结论（比如一份几百字的摘要）通过 A2A 返回给调度 Agent。

调度 Agent 的上下文里只多了一份摘要，而不是几十个网页的原文。这就把「调研过程」的上下文压力隔离在了市场 Agent 内部，调度 Agent 保持轻量，这是多 Agent 协作在上下文层面的核心收益。

![](AI-Agent-Learning/AI零碎/assets/什么是%20a2a%20协议？它和%20mcp%20协议的区别是什么？/file-20260528072006451.jpg)

解决方案很自然：把任务拆开，交给不同的专业 Agent 并行处理，最后汇总。一个「调度 Agent」负责任务拆分，「市场分析 Agent」专门做趋势调研，「技术研究 Agent」专门做工具对比，每个只需聚焦自己擅长的部分，整体效果远好于一个 Agent 包揽所有。

#### 多 Agent 的基础问题，Agent 之间怎么互相认识？
多 Agent 系统有一个绕不开的基础问题：Agent A 要把任务委托给 Agent B，它得先知道 B 能做什么。但怎么知道呢？
最笨的方案是写死配置：A 的代码里硬编码「B 可以做竞品分析」。这样太脆了，B 的能力一变，A 的代码就得改，根本没法维护。
更好的方案是让 B 主动「发名片」，声明自己能做什么，A 来查。这就是 A2A 里 **Agent Card** 的设计思路。

每个 A2A Agent 都在一个约定位置发布一张 JSON 格式的名片（A2A 规范推荐的路径是 `/.well-known/agent-card.json`，早期版本叫 `agent.json`，两种路径在社区里都能看到），里面写清楚自己叫什么、能做哪类任务（Skill 列表）、支不支持流式返回、支不支持异步回调（push notification，任务完成后主动通知调用方）。任何想和它协作的 Agent，先去拿这张名片，再决定要不要把任务委托给它。

Agent Card 里最关键的是 **Skill 列表**，每个 Skill 描述一类能力，比如「竞品分析」「行业趋势分析」，并带有示例输入。调度 Agent 用这些 Skill 描述来做任务路由决策，「这个任务和哪个 Agent 的哪个 Skill 最匹配？」。

![](AI-Agent-Learning/AI零碎/assets/什么是%20a2a%20协议？它和%20mcp%20协议的区别是什么？/file-20260528072104461.jpg)
这套机制让整个多 Agent 系统变得可插拔：新加一个 Agent，发布它的 Agent Card，调度 Agent 就能自动发现和利用它，完全不需要改调度 Agent 的代码。

![](AI-Agent-Learning/AI零碎/assets/什么是%20a2a%20协议？它和%20mcp%20协议的区别是什么？/file-20260528072136489.jpg)

#### Task，A2A 里的一等公民
A2A 里任务协作的基本单位是 **Task**。
调度 Agent 把一段任务委托给另一个 Agent，就是创建一个 Task；接收方执行这个 Task；完成后把结果作为 Task 的产出（artifacts，可以是文本、文件等）返回。
Task 有完整的生命周期状态管理。一个 Task 刚被创建时是 submitted 状态，表示已提交、等待处理。接收方开始执行后变为 working 状态，最终根据执行结果进入 completed（成功）或 failed（失败）状态。

![](AI-Agent-Learning/AI零碎/assets/什么是%20a2a%20协议？它和%20mcp%20协议的区别是什么？/file-20260528072218172.jpg)
为什么需要这么完整的状态机？因为 A2A 专门为**长时间任务**设计。
一个「竞品分析」任务可能要跑几分钟，先搜索、再整理、再写报告，不可能让调度 Agent 同步等着。
所以调度 Agent 提交任务后可以去处理其他事情，通过轮询状态或者 push notification（任务完成时接收方主动回调通知）来得知任务完成了。这套状态管理机制，正是为了支持这种异步长任务协作的。
调度 Agent 的视角很干净：给 Agent B 提交一个 Task，定期查一下状态，等到 `completed` 了去取 artifacts。整个过程不需要知道 B 内部用了什么工具、调了几次 LLM，完全黑盒。每个专业 Agent 自己的实现对外不可见，这正是解耦的意义所在。

#### A2A技术实现
具体的技术实现主要体现在**数据结构（JSON）** 和 **交互生命周期**两个方面：
##### 1. 核心数据结构与代码示例
A2A 协议主要通过标准的 JSON 对象来传递信息。以下是技术落地中最关键的几个结构：
###### A. 智能体卡片 (Agent Card)
这是 A2A 的“服务发现”机制。作为服务端的智能体，会在一个公开的 URL（例如 `https://api.yourcompany.com/agent/card.json`）挂载这份配置文件，告诉其他智能体自己能干什么、需要什么参数。

```JSON
{
  "protocol_version": "A2A/1.0",
  "agent_id": "hr-leave-agent-01",
  "name": "Leave Approval Agent",
  "description": "Handles employee leave requests and policy checks.",
  "capabilities": [
    {
      "task_type": "submit_leave_request",
      "input_schema": {
        "employee_id": "string",
        "dates": "array",
        "reason": "string"
      },
      "output_schema": {
        "status": "string",
        "approval_id": "string"
      }
    }
  ],
  "auth_requirements": ["Bearer Token"]
}
```

###### B. 任务请求与消息 (Task & Message)
当客户端智能体（Client Agent）想要委派任务时，它会向服务端智能体发送一个标准化的 Task 请求。由于 AI 思考和执行可能需要几分钟甚至更久，A2A 强调**任务 ID (Task ID)** 来追踪状态。

```JSON
// 客户端发起的请求 (POST /a2a/tasks)
{
  "client_agent_id": "calendar-assistant-99",
  "task_type": "submit_leave_request",
  "payload": {
    "employee_id": "EMP-456",
    "dates": ["2026-06-01", "2026-06-02"],
    "reason": "Annual vacation"
  },
  "context": {
    "urgency": "low"
  }
}
```

服务端收到后，不会立刻返回最终结果（因为大模型推理需要时间），而是立刻返回一个带有状态的确认：

```JSON
// 服务端的同步响应
{
  "task_id": "task_883a9f-22b1",
  "status": "PROCESSING",
  "created_at": "2026-05-27T23:16:05Z",
  "message": "Task accepted. Evaluating leave policy."
}
```

##### 2. 标准交互生命周期 (Lifecycle)
A2A 在技术实现上严格规定了任务的状态流转。一次完整的 A2A 交互通常包含以下技术步骤：
1. **发现与握手 (Discovery & Handshake)：** 客户端智能体通过拉取目标智能体的 `Agent Card` JSON 文件，验证其是否具备所需能力，并根据 `auth_requirements` 准备身份验证（如 OAuth 或 API Token）。
2. **任务初始化 (Task Creation)：** 客户端发送 POST 请求创建任务，服务端分配一个唯一的 `task_id`。
3. **状态同步 (Stateful Polling / Callback)：** 由于 AI 处理是长耗时操作，客户端可以通过轮询（GET `/tasks/{task_id}`）或服务端主动回调（Webhook）来更新进度。状态机通常包含：`PENDING` -> `PROCESSING` -> `WAITING_FOR_INPUT` (需要人类或客户端补充信息) -> `COMPLETED` / `FAILED`。
4. **工件交付 (Artifact Return)：** 任务完成后，服务端通过 A2A 协议返回最终的结构化结果（如一段分析报告、一个数据库操作记录的凭证）。
##### 3. A2A 技术实现的三大关键优势
- **黑盒互操作性 (Opaque Collaboration)：** A2A 故意设计成“黑盒”模式。调用方不需要知道接收方用的是 GPT-4 还是 Claude 3.5，也不需要共享对方的系统内存 (Memory) 或工具 (Tools)。这完美契合了企业的安全隐私合规要求。
- **状态持久化 (Stateful)：** 不同于 MCP 侧重于无状态的工具调用，A2A 的通信在底层设计上就是有状态的，专门用来支撑长周期的复杂多轮推理任务。
- **语言与框架无关：** Google 在将其捐赠给 Linux 基金会后，社区提供了 Python、JavaScript、Java、C# 和 Go 等官方 SDK。无论你是用 LangGraph 编排的复杂流程，还是几行代码简单封装的 API，只要套上 A2A 的接口壳子，就能立刻变成多智能体网络中的一个节点。

#### A2A 的架构本质，Agent 的微服务化
它就是 Agent 世界里的**微服务架构**。

在微服务架构里，每个服务是独立部署的 HTTP 服务，有自己的 API 文档，服务之间通过 HTTP 互相调用，支持异步消息队列处理耗时任务。A2A 的设计几乎照搬了这套思路，只不过把「服务」换成了「Agent」。

怎么理解这个对应关系呢？Agent Card 就像 API 文档，告诉别人「我能做什么、怎么调用我」。Task 状态机就像异步消息队列，支持提交任务后去做别的事、完成了再来取结果。而 `.well-known` 下的 Agent Card 就像微服务注册中心里的一条记录，让其他 Agent 能自动发现你。

所以每个 A2A Agent 对外其实就是一个 HTTP 服务，任何支持 A2A 的系统都可以发现它、向它发任务、接收结果，不绑定特定的 AI 框架，也不依赖特定的编程语言。这个设计理念和 MCP 是一脉相承的：MCP 让工具成为独立标准化服务，A2A 让 Agent 成为独立标准化服务。
![](AI-Agent-Learning/AI零碎/assets/什么是%20a2a%20协议？它和%20mcp%20协议的区别是什么？/file-20260528082951469.jpg)

#### A2A 和 MCP 的关系，一纵一横，各管一层

理清两者关系最简单的方式是看方向：MCP 是 Agent 向下连工具，A2A 是 Agent 向外连其他 Agent。

具体来说，一个专业 Agent 内部，用 MCP 连各种工具，比如数据库、浏览器、代码执行器，用 Function Calling 让 LLM 触发这些工具调用，这是「纵向」的连接。而多个 Agent 之间需要分工协作的时候，就用 A2A 来互相通信、委派任务、接收结果，这是「横向」的连接。两个协议解决的是完全不同维度的问题，不存在谁替代谁。

打个比方，MCP 就像公司里每个员工的「工具箱」，决定了这个人能用什么工具干活。A2A 就像公司里的「协作流程」，决定了不同岗位的人怎么分工、怎么交接任务。工具箱和协作流程是两回事，缺了哪个都不行。
![](AI-Agent-Learning/AI零碎/assets/什么是%20a2a%20协议？它和%20mcp%20协议的区别是什么？/file-20260528083029261.jpg)

在一个复杂的多 Agent 系统里，这两者通常同时在用：MCP 负责每个 Agent 和工具之间的纵向连接，A2A 负责 Agent 之间的横向协作通信。两层协议各管一个维度，合在一起才能支撑起真正复杂的 Agent 系统。

![](AI-Agent-Learning/AI零碎/assets/什么是%20a2a%20协议？它和%20mcp%20协议的区别是什么？/file-20260528083044958.jpg)


#### 常见问题
##### 1. 处理时间过久的“超时机制” (Timeout Handling)
A2A 协议并未采用简单的 HTTP 请求超时（因为容易断开连接），而是采用“异步状态机 + 双向超时控制”的机制：
- **客户端超时控制 (Client-side TTL / Timeout)：** 客户端在发起任务时，可以在 `context` 中附带一个**生存时间 (TTL, Time-To-Live)** 或明确的截止时间 (Deadline)。
    - 如果超过这个时间，客户端智能体不再等待，它会主动发起一个 `POST /tasks/{task_id}/cancel` 请求，通知服务端停止处理（以节省昂贵的 Token 和算力成本）。
- **服务端执行上限 (Server-side Execution Limit)：** 服务端智能体也会自我设定一个处理上限（例如最大推理时间 5 分钟）。如果底层模型卡死或循环检索数据超时，服务端会将任务状态从 `PROCESSING` 更新为 `FAILED`，并在 `error` 字段中注明 `TIMEOUT`。
- **心跳与保活机制 (Heartbeat / Keep-Alive)：** 对于预期耗时极长的任务（如代码自动生成与测试），服务端可以在状态轮询中返回进度信息（如 `{"status": "PROCESSING", "progress": "Running unit tests..."}`）。这相当于“心跳”，告诉客户端“我还活着，还在干活，请不要超时中断我”。
    
##### 2. 状态同步的异常处理：断网与重试

当客户端通过轮询 (Polling) 或服务端通过 Webhook 回调同步状态时，极其容易遇到网络闪断。

- **幂等性设计 (Idempotency)：** 这是 A2A 协议的核心要求。客户端在发起任务或重试请求时，必须携带一个唯一的 `Idempotency-Key`（或直接复用 `task_id`）。这样，即使因为网络超时导致客户端重复发送了多次“执行任务”的请求，服务端也只会执行一次，并返回相同的结果，避免了类似“重复扣款”或“重复发邮件”的灾难。
    
- **指数退避重试 (Exponential Backoff)：** 当状态拉取失败（如遇到 HTTP 503 或 429 限流）时，客户端智能体不应疯狂重试，而应采用退避算法（如等待 1秒、2秒、4秒、8秒）来尝试恢复状态同步。
    

##### 3. A2A 其他常见异常及处理规范

除了超时和网络问题，A2A 协议还定义了标准化的大模型协作异常。这些异常通常会在任务状态变为 `FAILED` 时，通过标准的错误对象返回：
###### A. 契约/格式异常 (Schema Violation)
- **场景：** 客户端智能体“幻觉”了，乱编了一些参数，或者少传了必填参数。
    
- **处理：** 服务端在接收到请求的第一时间（同步响应阶段）进行校验。如果不符合 Agent Card 中的 `input_schema` 定义，直接拒绝，并返回类似 HTTP 400 的错误，附带清晰的错误指引（例如："Missing required field: employee_id"）。客户端智能体收到该指引后，可以尝试自我纠正并重新构建请求。
    
###### B. 底层大模型解析异常 (LLM Execution Failure)
- **场景：** 服务端智能体正在努力干活，但它底层的 LLM 突然输出了乱码，或者未能生成符合预期的内部 JSON。
- **处理：** 协议要求服务端“内部消化”一定程度的错误。服务端应在内部进行自动重试（比如提示大模型“格式错误，请重新输出 JSON”）。如果内部重试达到上限依然失败，则向客户端暴露 `EXECUTION_FAILED` 状态。
###### C. 能力越界或被拒绝 (Capability Refusal / Safety Guardrails)

- **场景：** 客户端要求服务端执行违反安全策略或超出其权限的操作（例如要求 HR 智能体查询 CEO 的薪资）。
    
- **处理：** 触发安全护栏。状态立即变更为 `FAILED`，错误码通常为 `PERMISSION_DENIED` 或 `SAFETY_VIOLATION`。
    

###### D. 死锁与无限循环等待 (Deadlocks)
- **场景：** 智能体 A 在等待智能体 B 的结果，而智能体 B 为了完成任务又反向调用了智能体 A，形成死循环。
- **处理：** A2A 协议通常引入 **分布式追踪链路 (Trace ID) 机制**。当请求头中携带相同的 Trace ID 且调用层级 (Depth) 超过协议预设的阈值（如最多允许嵌套 5 层调用）时，系统将强制熔断，抛出 `MAX_DEPTH_EXCEEDED` 异常。
**标准化异常载荷示例**
为了让智能体“看懂”错误并决定下一步行动，A2A 的异常信息通常是高度结构化的，而非给人类看的一长串报错堆栈：

```JSON
{
  "task_id": "task_883a9f-22b1",
  "status": "FAILED",
  "error": {
    "code": "SCHEMA_VALIDATION_ERROR",
    "message": "The provided dates format is invalid.",
    "details": {
      "field": "dates",
      "expected_format": "YYYY-MM-DD",
      "received_value": "tomorrow and the day after"
    },
    "suggested_action": "RETRY_WITH_CORRECT_FORMAT"
  }
}
```

通过这种标准化的 `error.code` 和 `suggested_action`，客户端的 AI 才能明白自己错在哪里，并结合上下文重新生成正确的 A2A 请求。

#### 子智能体异常处理流程
主智能体处理子智能体异常的标准流程和技术策略：
##### 1. 异常诊断与快速重试 (Level 1: 自我恢复)

当主智能体接收到子智能体返回的 `FAILED` 状态和错误载荷（Error Payload）时，第一步是根据错误类型决定重试策略。
- **提示词自我修正 (Self-Correction)：** 如果子智能体返回的错误是“契约异常”（Schema Violation）或“输出无法解析”，主智能体会将错误信息（例如：_“你刚才输出的 JSON 缺少了日期字段”_）作为上下文，连同原任务一起发回给该子智能体，要求其自我反思并重新生成。
- **指数退避重试 (Exponential Backoff)：** 如果错误类型是网络超时、API 限流（429）或服务端暂不可用（503），主智能体不应立刻重发请求，而是采用退避算法（等待 2s、4s、8s 后重试），避免加重系统负担。
##### 2. 动态路由与降级策略 (Level 2: 改变路径)
如果子智能体经过重试依然失败，主智能体必须改变策略，不能在同一棵树上吊死。
- **故障转移 (Failover) 与寻找替补：** 主智能体可以查询全局的 Agent 注册表（如 A2A 里的 Agent Cards），寻找具备相同 `capabilities` 的备用智能体。例如，原本负责分析数据的 `Data-Agent-Claude` 挂了，主智能体自动把任务路由给 `Data-Agent-GPT`。
    
- **任务拆解与重规划 (Re-planning)：** 有时子智能体报错是因为主智能体派发的任务**太复杂**或**超出上下文窗口**。主智能体会捕获 `EXECUTION_FAILED`，重新审视目标，将原本的大任务拆解成 2-3 个更小、更具体的子任务，然后重新分发。

##### 3. 边界防护与兜底机制 (Level 3: 止损与上报)

当所有技术手段都无法让任务继续推进时，主智能体需要确保整个系统不会因为一个子节点的崩溃而陷入死循环或全面瘫痪。

- **熔断机制 (Circuit Breaking)：** 为防止过度消耗昂贵的 Token 算力，主智能体会设置阈值（例如：连续失败 3 次）。触发阈值后，主智能体对该子智能体执行“熔断”，在一段时间内不再向其派发任何任务。
    
- **局部失败与优雅降级 (Graceful Degradation)：** 如果该子智能体的任务并非核心阻断路径，主智能体可以拼凑现有的**部分成功结果**返回。
    
    > **示例：** 主智能体正在撰写一份《行业分析报告》。负责“拉取实时股票数据”的子智能体异常了，但负责“历史行业分析”的子智能体成功了。主智能体会生成报告，并在财报部分标注：_“注：实时股票数据暂无法获取，以下基于历史数据分析。”_
    
- **人类在环 (Human-in-the-Loop, HITL)：** 当关键任务遇到不可逾越的障碍（如需要特定权限、子智能体频繁陷入死锁），主智能体会主动挂起整个任务流，将状态置为 `WAITING_FOR_INPUT`，并向用户发出结构化的求助信息，等人类介入解决或补充数据后，再继续执行后续流程。