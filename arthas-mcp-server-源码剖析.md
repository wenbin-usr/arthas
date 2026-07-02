# Arthas MCP Server 源码深度剖析

> 文档日期：2026-06-04  
> 分析范围：Arthas 仓库 `arthas-mcp-server` 模块 + `core` 模块 MCP 集成层  
> 协议版本：支持 `2024-11-05`、`2025-03-26`、`2025-06-18`、`2025-11-25`

---

## 1. 概述

Arthas MCP Server 是 Arthas 的实验性模块，实现了 [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) 服务端。其核心目标是：**让 AI 客户端（Claude Desktop、Cherry Studio、Cline 等）通过标准化 JSON-RPC 2.0 接口，以 Tool Call 方式执行 Arthas 诊断命令**。

与独立进程型 MCP Server 不同，Arthas 的实现是 **嵌入式** 的：MCP 服务挂载在 Arthas 已有的 HTTP 端口（默认 8563）上，复用 Netty HTTP 管线与 Arthas Session/Job 执行体系，而非另起一套进程或 stdio 传输。

### 1.1 设计要点

| 维度 | 实现方式 |
|------|----------|
| 传输层 | HTTP + SSE（Streamable）或 HTTP 单次请求（Stateless） |
| 协议层 | JSON-RPC 2.0 + MCP 规范方法 |
| 异步模型 | `CompletableFuture`（非 Reactor） |
| 工具注册 | 注解扫描 `@Tool` + `@ToolParam` |
| 命令执行 | 桥接 `CommandExecutor` → Arthas `SessionManager` / `JobController` |
| 长耗时命令 | Task 模式 + 流式结果收集（watch/trace/monitor 等） |

---

## 2. 模块结构与依赖关系

```
arthas/
├── arthas-mcp-server/          # MCP 协议实现（通用层，不依赖 Arthas 命令细节）
│   ├── protocol/
│   │   ├── spec/               # MCP Schema、Session、Transport 接口
│   │   ├── server/             # McpNettyServer、Handler、Transport 实现
│   │   └── config/             # McpServerProperties
│   ├── tool/                   # ToolCallback、注解、JSON Schema 生成
│   ├── task/                   # MCP Task 生命周期（长任务）
│   └── session/                # MCP Session ↔ Arthas Session 绑定
│
├── core/
│   ├── mcp/
│   │   ├── ArthasMcpBootstrap.java   # 启动入口
│   │   ├── ArthasMcpServer.java      # 组装 MCP Server
│   │   └── tool/function/            # 29 个 Arthas 诊断工具
│   └── shell/term/impl/http/
│       └── HttpRequestHandler.java   # HTTP 路由，将 /mcp 转发给 MCP Handler
│
└── arthas-mcp-integration-test/  # 集成测试
```

**Maven 依赖**：`core` 依赖 `arthas-mcp-server`；`arthas-mcp-server` 依赖 Netty、Jackson，不反向依赖 `core`。工具实现放在 `core`，协议框架放在 `arthas-mcp-server`，职责分离清晰。

---

## 3. 启动与集成流程

### 3.1 配置入口

在 `arthas.properties` 或启动参数中配置：

```properties
arthas.mcpEndpoint=/mcp
arthas.mcpProtocol=STREAMABLE   # 或 STATELESS
```

对应 `Configure` 类中的 `mcpEndpoint` 与 `mcpProtocol` 字段。

### 3.2 ArthasBootstrap 启动链路

```
ArthasBootstrap.bind()
  ├── 绑定 Telnet / HTTP 端口
  ├── 创建 SessionManagerImpl、HttpApiHandler
  └── 若 mcpEndpoint 非空：
        CommandExecutor commandExecutor = new CommandExecutorImpl(sessionManager)
        arthasMcpBootstrap = new ArthasMcpBootstrap(commandExecutor, mcpEndpoint, mcpProtocol)
        mcpRequestHandler = arthasMcpBootstrap.start().getMcpRequestHandler()
```

关键代码位于 `core/src/main/java/com/taobao/arthas/core/server/ArthasBootstrap.java`（约 486–494 行）。

### 3.3 ArthasMcpServer 组装

`ArthasMcpServer.start()` 完成以下步骤：

1. **注册 JSON 过滤器**：`McpObjectVOFilter.register()`，过滤 Arthas 结果中的敏感/冗余字段。
2. **构建 `McpServerProperties`**：服务名、版本、endpoint、协议模式、ObjectMapper 等。
3. **扫描工具**：`DefaultToolCallbackProvider` 扫描 `com.taobao.arthas.core.mcp.tool.function` 包下所有 `@Tool` 方法。
4. **工具分类**（仅 Streamable 模式）：
   - `FORBIDDEN` → 普通同步工具（jvm、memory 等）
   - `OPTIONAL` / `REQUIRED` → Task 感知工具（watch、trace、monitor 等）
5. **创建统一 HTTP 入口**：`McpHttpRequestHandler`
6. **按协议分支**：
   - `STREAMABLE` → `startStreamableServer()`
   - `STATELESS` → `startStatelessServer()`

### 3.4 HTTP 路由挂载

MCP 不单独监听端口，而是嵌入现有 HTTP Handler：

```java
// HttpRequestHandler.channelRead0()
if (mcpRequestHandler != null) {
    String mcpEndpoint = mcpRequestHandler.getMcpEndpoint();
    if (mcpEndpoint.equals(path)) {
        mcpRequestHandler.handle(ctx, request);
        return;  // 不再走 WebUI 静态资源逻辑
    }
}
```

因此客户端访问 `http://localhost:8563/mcp` 即进入 MCP 协议处理。

---

## 4. 传输层：两种模式对比

Arthas 实现了 MCP 规范中的两种 HTTP 传输模式，通过 `McpServerProperties.ServerProtocol` 枚举切换。

### 4.1 Streamable HTTP（默认，推荐）

**类**：`NettyStreamableServerTransportProvider` → `McpStreamableHttpRequestHandler`

**特点**：
- 有状态 Session（`mcp-session-id` Header）
- 支持 GET 建立 SSE 长连接，接收服务端推送（通知、流式响应）
- POST 发送 JSON-RPC 请求；对 `tools/call` 等请求返回 SSE 流
- 支持 DELETE 删除 Session
- 支持 Task、Keep-Alive Ping（默认 15 秒）

**HTTP 方法语义**：

| 方法 | 用途 |
|------|------|
| POST + `initialize` | 创建 Session，响应头返回 `mcp-session-id` |
| POST + 其他 RPC | 处理请求，多数返回 SSE 流 |
| GET + `Accept: text/event-stream` | 建立监听流，接收服务端通知 |
| DELETE | 销毁 Session |

**Initialize 握手流程**：

```
Client                          Server
  | POST /mcp (initialize)         |
  |------------------------------->|
  |                                | sessionFactory.startSession()
  |                                | → 创建 McpStreamableServerSession (UUID)
  |                                | → 存入 sessions ConcurrentHashMap
  |<-------------------------------|
  | 200 JSON + mcp-session-id      |
  |                                |
  | GET /mcp (SSE listen)          |
  |------------------------------->|
  |<-------------------------------|
  | SSE stream (notifications)     |
```

**SSE 消息格式**（`NettyStreamableMcpSessionTransport.sendSseEvent`）：

```
id: {sessionId}
event: message
data: {JSON-RPC message}

```

**POST 非 initialize 请求的处理**（`handlePostRequest`）：
- `JSONRPCResponse` → 202 Accepted（客户端回调响应）
- `JSONRPCNotification` → 202 Accepted
- `JSONRPCRequest` → 200 + SSE 流，通过 `session.responseStream()` 异步写回

**限制**：不支持 `Last-Event-ID` 断线重连回放（收到该 Header 直接 404，强制客户端重新 Initialize）。

### 4.2 Stateless HTTP

**类**：`NettyStatelessServerTransport` → `McpStatelessHttpRequestHandler`

**特点**：
- 无 Session，每个 POST 独立处理
- 仅支持 POST；GET 返回 405
- 每个请求临时创建 Arthas Session，执行完毕后关闭
- **不支持 MCP Task**（所有工具按普通工具注册）
- 响应为单次 JSON（非 SSE 流）

适用于简单客户端或调试场景；长耗时、流式命令（watch/trace）在 Stateless 模式下能力受限。

### 4.3 统一分发入口

`McpHttpRequestHandler` 根据 `protocol` 字段将请求转发：

```java
if (protocol == ServerProtocol.STREAMABLE) {
    streamableHandler.handle(ctx, request);
} else {
    statelessHandler.handle(ctx, request);
}
```

---

## 5. 协议层：MCP Server 核心

### 5.1 架构分层

```
McpHttpRequestHandler          ← HTTP 层
    ↓
McpStreamableHttpRequestHandler / McpStatelessHttpRequestHandler
    ↓
McpStreamableServerSession / DefaultMcpStatelessServerHandler
    ↓
McpNettyServer / McpStatelessNettyServer    ← 方法路由表
    ↓
McpRequestHandler<?> / McpStatelessRequestHandler<?>
    ↓
ToolSpecification.call / ServerTaskToolHandler
    ↓
ToolCallback → AbstractArthasTool → CommandExecutor
```

### 5.2 McpNettyServer：方法路由表

`McpNettyServer` 在构造时注册 MCP 标准方法 Handler（`prepareRequestHandlers()`）：

| MCP 方法 | Handler 职责 |
|----------|-------------|
| `ping` | 返回空 Map |
| `initialize` | 协商协议版本，返回 ServerCapabilities |
| `tools/list` | 列出普通工具 + Task 工具定义 |
| `tools/call` | 分发到普通工具或 `ServerTaskToolHandler` |
| `resources/list` | 资源列表（Arthas 默认未注册） |
| `resources/read` | 读取资源 |
| `resources/templates/list` | 资源模板 |
| `prompts/list` | Prompt 列表 |
| `prompts/get` | 获取 Prompt |
| `logging/setLevel` | 设置日志级别 |
| `tasks/*` | 由 `ServerTaskToolHandler` 注册 |

**协议版本协商**（`initializeRequestHandler`）：

```java
// 若客户端版本在支持列表中，采用客户端版本；否则采用服务器最高版本
String serverProtocolVersion = protocolVersions.get(protocolVersions.size() - 1);
if (protocolVersions.contains(initializeRequest.getProtocolVersion())) {
    serverProtocolVersion = initializeRequest.getProtocolVersion();
}
```

支持版本列表定义在 `ProtocolVersions` 接口。

### 5.3 McpStreamableServerSession：有状态会话

每个 MCP 客户端连接对应一个 `McpStreamableServerSession`：

- **ID**：UUID
- **requestHandlers / notificationHandlers**：方法路由表（构造时注入）
- **listeningStreamRef**：当前 SSE 监听流（GET 建立）
- **eventStore**：`InMemoryEventStore`，存储 SSE 事件（用于潜在 replay）
- **commandSessionManager**：绑定 Arthas Session
- **taskStore / taskMessageQueue**：Task 状态与消息队列

**请求处理核心**（`responseStream`）：

```java
ArthasCommandContext commandContext = createCommandContext(transportContext.get(MCP_AUTH_SUBJECT_KEY));
return requestHandler.handle(
    new McpNettyServerExchange(sessionId, stream, clientCapabilities, clientInfo, transportContext, ...),
    commandContext,
    jsonrpcRequest.getParams()
).handle((result, ex) -> {
    // 序列化为 JSONRPCResponse，通过 SSE 发送，存储到 eventStore
});
```

`McpNettyServerExchange` 封装了向客户端发送通知、Progress、Sampling 等能力，供流式工具使用。

### 5.4 ServerCapabilities 声明

`ArthasMcpServer.buildServerCapabilities()` 构建能力声明：

```java
ServerCapabilities.builder()
    .prompts(...)
    .resources(...)
    .tools(...)           // listChanged 通知
    .tasks(...)           // 仅当有 Task 工具时启用
    .build();
```

Task 能力包括 `list`、`cancel`、`toolsCall`（含 tasks/get、tasks/result）。

---

## 6. Session 管理：MCP ↔ Arthas 双 Session 映射

MCP 协议 Session 与 Arthas 命令 Session 是两套概念，通过 `ArthasCommandSessionManager` 桥接。

### 6.1 绑定模型

```
MCP Session ID (UUID)
    ↓ 1:1 映射
Arthas Session ID + Consumer ID
```

`CommandSessionBinding` 记录：
- `mcpSessionId`
- `arthasSessionId`
- `consumerId`（结果消费端）
- `createdTime` / `lastAccessTime`

### 6.2 生命周期

| 事件 | 行为 |
|------|------|
| 首次 tools/call | `getCommandSession(mcpSessionId, authSubject)` 创建 Arthas Session |
| 25 分钟无访问 | 认为过期，关闭旧 Session 并重建 |
| MCP Session DELETE | `closeCommandSession(mcpSessionId)` |
| Task 创建 | `createIsolatedTaskSession(taskId)` 独立 Session，避免阻塞主 Session |

### 6.3 ArthasCommandContext

工具执行时拿到的 `ArthasCommandContext` 封装了：

- `executeSync(command, authSubject, userId)` — 同步命令
- `executeAsync(command)` — 异步命令（创建 Job）
- `getResults(consumerId, ...)` — 拉取 Job 输出
- `interruptJob()` — 中断前台 Job
- `setSessionAuth` / `setSessionUserId` — 写入认证与统计信息

底层全部委托给 `CommandExecutorImpl`，后者操作 Arthas 原生 `SessionManager` 和 `JobController`。

---

## 7. 工具（Tool）体系

### 7.1 注解驱动注册

工具定义采用声明式注解：

```java
@Tool(
    name = "watch",
    description = "Watch 方法执行观察工具...",
    streamable = true,
    taskSupport = McpSchema.TaskSupportMode.OPTIONAL
)
public String watch(
    @ToolParam(description = "类名表达式", required = true) String classPattern,
    @ToolParam(description = "方法名", required = false) String methodPattern,
    ToolContext toolContext
) { ... }
```

**扫描流程**（`DefaultToolCallbackProvider`）：

1. 扫描 `ARTHAS_TOOL_BASE_PACKAGE`（`com.taobao.arthas.core.mcp.tool.function`）
2. 支持 file/jar 两种 ClassPath 来源
3. 对每个 `@Tool` 方法：
   - `ToolDefinitions.from(method)` 生成 `ToolDefinition`（含 JSON Schema）
   - `DefaultToolCallback` 包装为可调用对象

**JSON Schema 生成**：`JsonSchemaGenerator.generateForMethodInput(method)` 根据 `@ToolParam` 注解自动生成 inputSchema，供 AI 客户端理解参数。

### 7.2 工具分类

| 类型 | taskSupport | 示例 | 执行方式 |
|------|-------------|------|----------|
| 同步工具 | FORBIDDEN | jvm, memory, sc | `executeSync()` |
| 流式工具 | OPTIONAL | watch, trace, monitor | `executeStreamable()` + 轮询结果 |
| 强制 Task 工具 | REQUIRED | （部分长任务） | 必须走 MCP Task 协议 |

### 7.3 AbstractArthasTool：执行模板

所有 Arthas 工具继承 `AbstractArthasTool`，提供两种执行路径：

**同步**（`executeSync`）：

```
ToolContext → ArthasCommandContext → commandExecutor.executeSync()
         → JsonParser.toJson(result)
```

**流式**（`executeStreamable`）：

```
1. setSessionAuth / setSessionUserId
2. executeAsyncWithRetry() — 启动 Job，处理 "Another job is running" 重试
3. executeAndCollectResults() — 轮询 consumer 输出，支持 progressToken 推送
4. finally: interruptJob() — 释放前台 Job
```

`StreamableToolUtils` 负责与 `McpNettyServerExchange` 交互，发送 MCP Progress Notification。

### 7.4 ToolCallback → MCP ToolSpecification 桥接

`McpToolUtils.toToolSpecification()` 将 `ToolCallback` 适配为 MCP 层：

```java
McpServerFeatures.ToolCallFunction callFunction = (exchange, commandContext, request) -> {
    ToolContext toolContext = new ToolContext(Map.of(
        EXCHANGE, exchange,
        COMMAND_CONTEXT, commandContext,
        PROGRESS_TOKEN, request.progressToken(),
        MCP_TRANSPORT_CONTEXT, exchange.getTransportContext()
    ));
    String result = toolCallback.call(requestJson, toolContext);
    return CompletableFuture.completedFuture(createSuccessResult(result));
};
```

Stateless 模式使用 `McpStatelessServerFeatures.ToolSpecification`，不注入 `EXCHANGE`（为 null）。

### 7.5 tools/call 分发逻辑

`McpNettyServer.toolsCallRequestHandler()`：

```
1. 解析 CallToolRequest（tool name + arguments + optional task params）
2. 查 toolsByName → 普通工具
   - 若带 task 参数 → 报错（不支持 task 增强）
   - 否则 normalTool.getCall().apply(...)
3. 否则委托 ServerTaskToolHandler.handleToolCall(...)
4. 工具不存在 → INVALID_PARAMS
```

---

## 8. Task 子系统（长耗时命令）

MCP Task 是 MCP 2025-06-18+ 规范引入的能力，用于 **异步、可取消、可轮询** 的长任务。Arthas 将 watch/trace/monitor 等流式诊断命令映射为 Task 工具。

### 8.1 组件

| 组件 | 职责 |
|------|------|
| `TaskStore` | 内存存储 Task 元数据与结果（TTL 30 分钟） |
| `TaskMessageQueue` | Task 消息队列 |
| `ServerTaskToolHandler` | Task 工具注册、tools/call 分发、tasks/list/get/cancel |
| `ToolCallbackCreateTaskHandler` | 创建 Task 并在线程池执行工具 |
| `ArthasCommandSessionManager.createIsolatedTaskSession` | 每个 Task 独立 Arthas Session |

### 8.2 Task 专用线程池

```java
// 固定大小 = DEFAULT_MAX_CONCURRENT_TASK_SESSIONS
// SynchronousQueue + AbortPolicy — 无隐式排队，超限直接拒绝
ThreadPoolExecutor taskExecutor = new ThreadPoolExecutor(
    maxSessions, maxSessions, 0L, MILLISECONDS,
    new SynchronousQueue<>(),
    new McpTaskThreadFactory(),
    new AbortPolicy()
);
```

避免 I/O 密集型诊断任务污染 `ForkJoinPool.commonPool()`。

### 8.3 Task 工具注册

Streamable 模式下，`configureTaskSupport()` 为每个 Task 感知工具创建：

```java
TaskAwareToolSpecification.builder()
    .name(def.getName())
    .inputSchema(def.getInputSchema())
    .taskSupport(def.taskSupport())
    .createTaskHandler(new ToolCallbackCreateTaskHandler(callback, taskExecutor))
    .build();
```

普通工具（`FORBIDDEN`）与 Task 工具分表注册，避免 `tools/list` 重复。

### 8.4 取消支持

`AbstractArthasTool.buildCancellationChecker()` 在 Task 模式下定期检查 `CreateTaskContext.isCancellationRequested(taskId)`，流式收集循环中响应取消。

---

## 9. 认证与上下文传递

### 9.1 Bearer Token 认证

Arthas HTTP 层已有 `BasicHttpAuthenticatorHandler`。认证成功后，将 Subject 写入 Netty Channel Attribute：

```java
ctx.channel().attr(SUBJECT_ATTRIBUTE_KEY).set(subject);
```

MCP Handler 通过 `McpAuthExtractor.extractAuthSubjectFromContext(ctx)` 读取，放入 `McpTransportContext`：

```java
transportContext.put(MCP_AUTH_SUBJECT_KEY, authSubject);
transportContext.put(MCP_USER_ID_KEY, userId);  // 来自 X-User-Id Header
```

### 9.2 传递到 Arthas Session

```
MCP TransportContext
  → ArthasCommandSessionManager.getCommandSession(mcpSessionId, authSubject)
  → commandExecutor.setSessionAuth(arthasSessionId, authSubject)
  → Session.put(SUBJECT_KEY, authSubject)
```

工具执行时 `AbstractArthasTool` 再次确保 Session 上绑定了认证主体，保证 Arthas 命令鉴权一致。

客户端配置示例：

```json
{
  "mcpServers": {
    "arthas-mcp": {
      "type": "streamableHttp",
      "url": "http://localhost:8563/mcp",
      "headers": {
        "Authorization": "Bearer {password}"
      }
    }
  }
}
```

---

## 10. 完整请求时序（Streamable + tools/call）

以 AI 客户端调用 `jvm` 工具为例：

```
┌─────────┐     ┌──────────────────┐     ┌─────────────────────┐     ┌──────────────────┐
│ AI Client│     │ McpStreamableHttp│     │ McpStreamableServer │     │ CommandExecutor  │
│         │     │ RequestHandler   │     │ Session             │     │ (Arthas Core)    │
└────┬────┘     └────────┬─────────┘     └──────────┬──────────┘     └────────┬─────────┘
     │ POST initialize   │                          │                         │
     │──────────────────>│ startSession()           │                         │
     │                   │─────────────────────────>│                         │
     │<──────────────────│ mcp-session-id           │                         │
     │ GET SSE listen    │                          │                         │
     │──────────────────>│ listeningStream()        │                         │
     │                   │                          │                         │
     │ POST tools/call   │                          │                         │
     │ {name:"jvm"}      │                          │                         │
     │──────────────────>│ responseStream()         │                         │
     │                   │─────────────────────────>│                         │
     │                   │                          │ toolsCallRequestHandler │
     │                   │                          │ → JvmTool.executeSync   │
     │                   │                          │────────────────────────>│
     │                   │                          │                         │ jvm Job
     │                   │                          │<────────────────────────│
     │<─ SSE: result ───│<─────────────────────────│                         │
     │                   │                          │                         │
```

对于 `watch`（Task + 流式）：

1. Client 可选择带 `task` 参数调用 → `ServerTaskToolHandler` 创建 Task
2. `ToolCallbackCreateTaskHandler` 在线程池执行 `WatchTool.executeStreamable()`
3. 执行过程中通过 `exchange` 发送 Progress Notification
4. Client 通过 `tasks/get` / `tasks/result` 轮询或等待 SSE 推送

---

## 11. 关键类索引

| 类 | 模块 | 职责 |
|----|------|------|
| `ArthasMcpBootstrap` | core | MCP 启动门面 |
| `ArthasMcpServer` | core | 组装 Server、扫描工具、选协议 |
| `McpHttpRequestHandler` | arthas-mcp-server | HTTP 统一入口 |
| `McpStreamableHttpRequestHandler` | arthas-mcp-server | Streamable 传输实现 |
| `McpStatelessHttpRequestHandler` | arthas-mcp-server | Stateless 传输实现 |
| `McpNettyServer` | arthas-mcp-server | Streamable MCP 方法路由 |
| `McpStatelessNettyServer` | arthas-mcp-server | Stateless MCP 方法路由 |
| `McpStreamableServerSession` | arthas-mcp-server | 有状态 Session |
| `McpServer` | arthas-mcp-server | Builder API（`McpServer.netty(...).build()`） |
| `DefaultToolCallbackProvider` | arthas-mcp-server | 注解扫描注册工具 |
| `McpToolUtils` | core | ToolCallback → ToolSpecification 适配 |
| `AbstractArthasTool` | core | 工具执行基类 |
| `CommandExecutorImpl` | core | Arthas 命令执行桥接 |
| `ArthasCommandSessionManager` | arthas-mcp-server | MCP/Arthas Session 映射 |
| `ServerTaskToolHandler` | arthas-mcp-server | Task 生命周期管理 |
| `McpSchema` | arthas-mcp-server | MCP 协议数据模型 |
| `HttpRequestHandler` | core | HTTP 路由（含 /mcp） |

---

## 12. 设计亮点与权衡

### 12.1 亮点

1. **嵌入式集成**：复用 Arthas HTTP 端口与 Session/Job 体系，零额外部署成本。
2. **注解驱动工具扩展**：新增诊断能力只需在 `core/mcp/tool/function` 加 `@Tool` 方法，自动扫描注册。
3. **双传输模式**：Streamable 满足生产 AI 客户端；Stateless 便于轻量调试。
4. **Task 隔离**：长耗时命令独立 Arthas Session + 专用线程池，不阻塞短命令。
5. **CompletableFuture 全链路**：与 Netty 线程模型契合，无 Reactor 依赖。

### 12.2 权衡与限制

1. **不支持 Last-Event-ID 重连**：SSE 断线需完整重新 Initialize。
2. **Stateless 无 Task**：长命令在 Stateless 模式下体验受限。
3. **Task/Session 内存存储**：`InMemoryTaskStore`、`InMemoryEventStore`，进程重启即丢失。
4. **并发上限**：Task Session 数受 `DEFAULT_MAX_CONCURRENT_TASK_SESSIONS` 限制，超限抛 `RejectedExecutionException`。
5. **实验性**：模块标注为 experimental，API 可能演进。

---

## 13. 扩展指南

### 13.1 新增一个 MCP 工具

1. 在 `core/src/main/java/com/taobao/arthas/core/mcp/tool/function/` 对应子包新建类
2. 继承 `AbstractArthasTool`
3. 添加 `@Tool` + `@ToolParam` 注解方法
4. 同步命令调用 `executeSync()`；流式命令调用 `executeStreamable()` 并设置 `streamable = true`
5. 若需 Task 支持，设置 `taskSupport = OPTIONAL` 或 `REQUIRED`
6. 重启 Arthas，工具自动出现在 `tools/list`

### 13.2 切换传输模式

```properties
# Streamable（默认，推荐 AI 客户端）
arthas.mcpProtocol=STREAMABLE

# Stateless（简单调试）
arthas.mcpProtocol=STATELESS
```

### 13.3 自定义 MCP Endpoint

```properties
arthas.mcpEndpoint=/custom-mcp-path
```

客户端 URL 需同步修改。

---

## 14. 参考资料

- [Arthas MCP Server 官方文档](site/docs/doc/mcp-server.md)
- [arthas-mcp-server README](arthas-mcp-server/README.md)
- [MCP 规范](https://modelcontextprotocol.io/)
- [ARTHAS_ARCHITECTURE_CN.md](Arthas%20项目架构与实现原理分析.md)

---

*本文档基于 Arthas 源码静态分析生成，覆盖 arthas-mcp-server 与 core/mcp 模块的主要实现路径。*
