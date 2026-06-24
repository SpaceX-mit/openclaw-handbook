# OpenClaw 扩展点指南

## 目录

1. [概述](#1-概述)
2. [创建Plugin](#2-创建plugin)
3. [创建Tool](#3-创建tool)
4. [创建Skill](#4-创建skill)
5. [创建Channel](#5-创建channel)
6. [创建Model Provider](#6-创建model-provider)
7. [创建MCP Server](#7-创建mcp-server)
8. [最佳实践](#8-最佳实践)

---

## 1. 概述

### 1.1 扩展点类型

| 扩展点 | 说明 | 难度 |
|--------|------|------|
| Tool | 添加新工具 | ⭐ |
| Skill | 添加技能模块 | ⭐⭐ |
| Channel | 添加消息渠道 | ⭐⭐⭐ |
| Provider | 添加模型提供商 | ⭐⭐⭐ |
| MCP Server | 添加MCP服务器 | ⭐⭐ |

### 1.2 扩展架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        OpenClaw 扩展架构                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │                    Your Extension                          │  │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │  │
│   │  │  Tool   │ │  Skill  │ │ Channel │ │Provider │       │  │
│   │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘       │  │
│   │       └──────────┬┴──────────┬┴──────────┬┘            │  │
│   └──────────────────┼───────────┼───────────┼───────────────┘  │
│                      │           │           │                 │
│                      ▼           ▼           ▼                 │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │                    Plugin SDK                             │  │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │  │
│   │  │ToolReg   │  │SkillReg  │  │ChannelReg│              │  │
│   │  └──────────┘  └──────────┘  └──────────┘              │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 创建Plugin

### 2.1 Plugin基础结构

```
extensions/
└── my-plugin/
    ├── package.json
    ├── tsconfig.json
    ├── index.ts
    ├── src/
    │   ├── tools/
    │   ├── skills/
    │   └── runtime/
    └── README.md
```

### 2.2 package.json

```json
{
  "name": "openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": "./index.ts"
  },
  "openclaw": {
    "plugin": {
      "id": "my-plugin",
      "name": "My Plugin",
      "version": "1.0.0"
    }
  }
}
```

### 2.3 Plugin主文件

```typescript
// extensions/my-plugin/index.ts
import type { Plugin, PluginRuntime } from "@openclaw/plugin-sdk"

export const myPlugin: Plugin = {
  id: "my-plugin",
  name: "My Plugin",
  version: "1.0.0",
  description: "A custom plugin for OpenClaw",
  
  async setup(runtime: PluginRuntime) {
    // 注册工具
    runtime.registerTools([
      {
        name: "my_tool",
        description: "Does something useful",
        inputSchema: {
          type: "object",
          properties: {
            input: { type: "string" }
          },
          required: ["input"]
        },
        owner: { kind: "plugin", pluginId: "my-plugin" },
        executor: { kind: "plugin", pluginId: "my-plugin", toolName: "my_tool" }
      }
    ])
    
    // 注册技能
    runtime.registerSkills([
      {
        name: "my-skill",
        description: "A useful skill",
        content: "You are an expert at my-skill...",
        filePath: "/path/to/SKILL.md"
      }
    ])
    
    runtime.logger.info("My plugin initialized")
  },
  
  async teardown() {
    // 清理资源
  }
}

export default myPlugin
```

---

## 3. 创建Tool

### 3.1 Tool Descriptor格式

```typescript
// Tool Descriptor 示例
const myToolDescriptor: ToolDescriptor = {
  name: "calculate",
  title: "Calculator",
  description: "Performs mathematical calculations",
  inputSchema: {
    type: "object",
    properties: {
      expression: {
        type: "string",
        description: "Mathematical expression to evaluate"
      }
    },
    required: ["expression"]
  },
  outputSchema: {
    type: "object",
    properties: {
      result: { type: "number" }
    }
  },
  owner: { kind: "plugin", pluginId: "my-plugin" },
  executor: { kind: "plugin", pluginId: "my-plugin", toolName: "calculate" }
}
```

### 3.2 Tool Executor实现

```typescript
// src/tools/executors/calculate.ts
export async function executeCalculate(
  args: { expression: string },
  context: ToolExecutionContext
): Promise<ToolExecutionResult> {
  try {
    // 安全评估数学表达式（不使用 eval）
    const result = math.evaluate(args.expression)
    
    return {
      success: true,
      output: { result }
    }
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error.message : "Calculation failed"
    }
  }
}
```

### 3.3 Tool注册到Plugin

```typescript
// 在Plugin的setup中
runtime.registerTools([
  {
    name: "calculate",
    description: "Performs mathematical calculations",
    inputSchema: { /* ... */ },
    owner: { kind: "plugin", pluginId: "my-plugin" },
    executor: { kind: "plugin", pluginId: "my-plugin", toolName: "calculate" }
  }
])

// 注册对应的executor
runtime.registerToolExecutor({
  pluginId: "my-plugin",
  toolName: "calculate",
  execute: executeCalculate
})
```

### 3.4 Tool可用性条件

```typescript
const authenticatedTool: ToolDescriptor = {
  name: "api_call",
  description: "Make an authenticated API call",
  inputSchema: { /* ... */ },
  owner: { kind: "plugin", pluginId: "my-plugin" },
  executor: { kind: "plugin", pluginId: "my-plugin", toolName: "api_call" },
  availability: {
    anyOf: [
      { kind: "auth", providerId: "my-api" },
      { kind: "env", name: "MY_API_KEY" }
    ]
  }
}
```

---

## 4. 创建Skill

### 4.1 Skill文件格式

```markdown
# skills/my-skill/SKILL.md

## Name
my-skill

## Description
Performs specialized task X with expertise.

## Triggers
- When user mentions task X
- When user asks about topic Y
- Automatically for requests matching: /x.*

## Instructions
You are an expert at specialized task X.

You have access to the following tools:
- tool_a: For doing part A
- tool_b: For doing part B

Follow these steps:
1. Understand the user's request
2. Gather necessary information
3. Execute using appropriate tools
4. Present the results clearly

## Best Practices
- Always verify inputs before processing
- Provide clear progress updates
- Handle errors gracefully
- Include relevant context in responses

## Examples

### Example 1: Simple Request
User: "Help me with task X"
Assistant: "I'll help you with task X. Let me..."

### Example 2: Complex Request
User: "Handle this complex task"
Assistant: "This requires multiple steps. I'll..."
```

### 4.2 Skill元数据

```typescript
// Skill元数据格式
interface Skill {
  name: string
  description: string
  content: string
  filePath: string
  promptVersion?: string
  disableModelInvocation?: boolean
}
```

### 4.3 Skill注册

```typescript
// 方式1: 通过文件
runtime.registerSkills([
  {
    name: "my-skill",
    description: "A useful skill",
    content: readFileSync("/path/to/SKILL.md", "utf-8"),
    filePath: "/path/to/SKILL.md"
  }
])

// 方式2: 通过目录（自动扫描）
const skillsDir = "/path/to/skills"
const skills = await loadSkillsFromDirectory(skillsDir)
runtime.registerSkills(skills)
```

---

## 5. 创建Channel

### 5.1 Channel Plugin结构

```typescript
// extensions/my-channel/index.ts
import type { ChannelPlugin, ChannelTransport, PluginRuntime } from "@openclaw/plugin-sdk"

export const myChannelPlugin: ChannelPlugin = {
  id: "my-channel",
  name: "My Channel",
  
  async setup(runtime: PluginRuntime) {
    // 注册工具
    runtime.registerChannelTools({
      channelId: "my-channel",
      tools: [
        {
          name: "my_channel_send",
          description: "Send message via My Channel",
          inputSchema: { /* ... */ },
          owner: { kind: "channel", channelId: "my-channel" }
        }
      ]
    })
  },
  
  createTransport(config: MyChannelConfig): ChannelTransport {
    return new MyChannelTransport(config)
  },
  
  createSession(config: MyChannelSessionConfig): ChannelSession {
    return new MyChannelSession(config)
  }
}
```

### 5.2 ChannelTransport实现

```typescript
// src/transport/my-channel-transport.ts
export class MyChannelTransport implements ChannelTransport {
  private socket?: WebSocket
  private handlers: MessageHandler[] = []
  
  constructor(private config: MyChannelConfig) {}
  
  async send(message: ChannelMessage): Promise<void> {
    const payload = this.transformToChannelFormat(message)
    await this.sendToChannel(payload)
  }
  
  receive(handler: MessageHandler): void {
    this.handlers.push(handler)
  }
  
  async close(): Promise<void> {
    await this.socket?.close()
  }
  
  private onMessage(payload: any): void {
    const message = this.transformFromChannelFormat(payload)
    for (const handler of this.handlers) {
      handler(message)
    }
  }
}
```

### 5.3 配置Schema

```typescript
// src/config.ts
export const myChannelConfigSchema = {
  type: "object",
  properties: {
    apiKey: { type: "string" },
    webhookUrl: { type: "string" },
    botToken: { type: "string" },
    allowFrom: {
      type: "array",
      items: { type: "string" }
    },
    dmPolicy: {
      type: "string",
      enum: ["open", "pairing", "closed"]
    }
  },
  required: ["apiKey"]
}
```

---

## 6. 创建Model Provider

### 6.1 Provider结构

```typescript
// extensions/my-provider/index.ts
import type { ModelProvider, ProviderRuntime, PluginRuntime } from "@openclaw/plugin-sdk"

export class MyProvider implements ModelProvider {
  readonly id = "my-provider"
  readonly api = "openai-compatible" // 或 "anthropic", "google"
  
  async discover(): Promise<Model[]> {
    return [
      {
        id: "my-model-1",
        name: "My Model 1",
        api: this.api,
        provider: this.id,
        contextWindow: 128000,
        maxTokens: 16384,
        input: ["text", "images"],
        cost: { input: 1, output: 1 }
      },
      {
        id: "my-model-reasoning",
        name: "My Reasoning Model",
        api: this.api,
        provider: this.id,
        contextWindow: 200000,
        maxTokens: 32768,
        reasoning: true,
        input: ["text"],
        cost: { input: 2, output: 2 }
      }
    ]
  }
  
  createRuntime(config: MyProviderConfig): ProviderRuntime {
    return new MyProviderRuntime(config)
  }
  
  validateConfig(config: unknown): ValidationResult {
    // 验证配置
    if (!config.apiKey) {
      return { valid: false, error: "Missing apiKey" }
    }
    return { valid: true }
  }
  
  getAuthInstructions(): AuthInstructions {
    return {
      type: "api-key",
      url: "https://my-provider.example.com/api-keys",
      instructions: "Get your API key from the dashboard"
    }
  }
}
```

### 6.2 ProviderRuntime实现

```typescript
// src/runtime/my-provider-runtime.ts
export class MyProviderRuntime implements ProviderRuntime {
  constructor(private config: MyProviderConfig) {}
  
  async *stream(
    model: Model,
    messages: Message[],
    options: StreamOptions
  ): AsyncIterable<ProviderEvent> {
    const response = await fetch(`${this.config.baseUrl}/chat/completions`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${this.config.apiKey}`
      },
      body: JSON.stringify({
        model: model.id,
        messages: this.transformMessages(messages),
        stream: true,
        ...options
      })
    })
    
    const reader = response.body?.getReader()
    if (!reader) return
    
    const decoder = new TextDecoder()
    
    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      
      const chunk = decoder.decode(value)
      
      for (const line of chunk.split("\n")) {
        if (line.startsWith("data: ")) {
          const data = JSON.parse(line.slice(6))
          yield this.transformEvent(data)
        }
      }
    }
    
    yield { type: "done", reason: "stop", usage: {} }
  }
  
  private transformMessages(messages: Message[]): any[] {
    return messages.map(msg => ({
      role: msg.role,
      content: this.formatContent(msg.content),
      name: msg.name
    }))
  }
  
  private formatContent(content: Content[]): string | object[] {
    return content.map(c => {
      if (c.type === "text") {
        return { type: "text", text: c.text }
      }
      if (c.type === "image") {
        return { type: "image_url", image_url: { url: c.url } }
      }
      return c
    })
  }
}
```

---

## 7. 创建MCP Server

### 7.1 MCP Server结构

```typescript
// mcp/my-server/src/index.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js"
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js"
import { CallToolRequestSchema, ListToolsRequestSchema } from "@modelcontextprotocol/sdk/types.js"

const server = new Server(
  {
    name: "my-mcp-server",
    version: "1.0.0"
  },
  {
    capabilities: {
      tools: {}
    }
  }
)

// 列出工具
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "my_tool",
        description: "A useful MCP tool",
        inputSchema: {
          type: "object",
          properties: {
            input: { type: "string" }
          }
        }
      }
    ]
  }
})

// 调用工具
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params
  
  if (name === "my_tool") {
    try {
      const result = await myToolLogic(args.input)
      return {
        content: [
          { type: "text", text: JSON.stringify(result) }
        ]
      }
    } catch (error) {
      return {
        content: [
          { type: "text", text: `Error: ${error.message}` }
        ],
        isError: true
      }
    }
  }
  
  throw new Error(`Unknown tool: ${name}`)
})

// 启动
async function main() {
  const transport = new StdioServerTransport()
  await server.connect(transport)
  console.error("My MCP Server running on stdio")
}

main()
```

### 7.2 MCP Server配置

```yaml
# openclaw.json
{
  "mcp": {
    "servers": {
      "my-server": {
        "command": "node",
        "args": ["/path/to/my-server/dist/index.js"],
        "env": {
          "API_KEY": "xxx"
        }
      }
    }
  }
}
```

### 7.3 MCP工具到OpenClaw的映射

```typescript
// MCP工具自动映射为OpenClaw Tool
const mcpToolDescriptor: ToolDescriptor = {
  name: "my-server/my_tool",  // server-name/tool-name
  description: "A useful MCP tool (via my-server)",
  inputSchema: {
    type: "object",
    properties: {
      input: { type: "string" }
    }
  },
  owner: { kind: "mcp", serverId: "my-server" },
  executor: { kind: "mcp", serverId: "my-server", toolName: "my_tool" },
  availability: {
    allOf: [
      { kind: "config", path: ["mcp", "servers", "my-server"] }
    ]
  }
}
```

---

## 8. 最佳实践

### 8.1 Plugin开发

```typescript
// ✅ 正确: 清晰的初始化
async setup(runtime: PluginRuntime) {
  runtime.logger.info("Starting plugin setup")
  
  try {
    // 验证配置
    const config = runtime.getConfig("my-plugin")
    if (!config) {
      throw new Error("Missing configuration")
    }
    
    // 注册资源
    runtime.registerTools(...)
    runtime.registerSkills(...)
    
    runtime.logger.info("Plugin setup complete")
  } catch (error) {
    runtime.logger.error("Plugin setup failed", error)
    throw error
  }
}

// ❌ 错误: 不处理错误
setup(runtime: PluginRuntime) {
  runtime.registerTools(...) // 可能失败但不处理
}
```

### 8.2 Tool设计

```typescript
// ✅ 正确: 清晰的Schema
{
  name: "create_file",
  description: "Creates a new file with the given content",
  inputSchema: {
    type: "object",
    properties: {
      path: {
        type: "string",
        description: "Absolute path where the file will be created"
      },
      content: {
        type: "string",
        description: "Content to write to the file"
      }
    },
    required: ["path", "content"]
  }
}

// ❌ 错误: 模糊的Schema
{
  name: "file_op",
  description: "Does file stuff",
  inputSchema: {
    type: "object",
    properties: {
      data: { type: "string" }
    }
  }
}
```

### 8.3 错误处理

```typescript
// ✅ 正确: 详细的错误信息
async function executeTool(args: any, context: ToolContext) {
  try {
    const result = await doSomething(args)
    return {
      success: true,
      output: result
    }
  } catch (error) {
    context.logger.error("Tool execution failed", { args, error })
    return {
      success: false,
      error: `Failed to execute: ${error.message}`,
      details: {
        code: error.code,
        recoverable: error.recoverable ?? true
      }
    }
  }
}
```

### 8.4 资源清理

```typescript
// ✅ 正确: 完整的生命周期
class MyPlugin implements Plugin {
  private connections: Connection[] = []
  
  async setup(runtime: PluginRuntime) {
    // 初始化连接池
    this.connections = await createConnectionPool()
    
    // 注册资源
    runtime.registerTools(...)
  }
  
  async teardown() {
    // 清理连接
    for (const conn of this.connections) {
      await conn.close()
    }
    this.connections = []
    
    // 清理临时文件
    await cleanupTempFiles()
  }
}
```

---

## 附录: 扩展点清单

| 扩展点 | 注册方法 | 位置 |
|--------|----------|------|
| Tool (Core) | `runtime.registerTools()` | Plugin setup |
| Tool (Channel) | `runtime.registerChannelTools()` | Plugin setup |
| Skill | `runtime.registerSkills()` | Plugin setup |
| Channel | `runtime.registerChannel()` | Plugin setup |
| Provider | `runtime.registerProvider()` | Plugin setup |
| Hook | `runtime.registerHook()` | Plugin setup |
| MCP | 配置文件中定义 | openclaw.json |

---

*文档生成时间: 2026-06-23*
