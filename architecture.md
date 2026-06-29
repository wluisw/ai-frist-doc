# 系统架构文档

## 整体架构

AI First Demo 采用 Next.js 全栈架构，分为前端展示层、API 代理层和 AI 服务层三个部分。

```
┌─────────────────────────────────────────────────┐
│                 客户端（浏览器）                    │
│  ┌──────────────┐    ┌──────────────────────┐   │
│  │  Landing Page │    │    ChatInterface      │   │
│  │  (首页)      │    │  (Client Component)   │   │
│  └──────────────┘    └──────────┬───────────┘   │
└──────────────────────────────── │ ──────────────┘
                                  │ HTTP POST /api/chat
┌─────────────────────────────────┼───────────────┐
│             Next.js 服务端       │               │
│  ┌──────────────────────────────▼────────────┐  │
│  │           Route Handler (/api/chat)        │  │
│  │  - 验证请求                                │  │
│  │  - 调用 Anthropic SDK                     │  │
│  │  - 转发流式响应 (SSE)                      │  │
│  └──────────────────────────────┬────────────┘  │
└──────────────────────────────── │ ──────────────┘
                                  │ HTTPS
┌─────────────────────────────────▼───────────────┐
│              Anthropic Claude API                │
│  - 模型：claude-sonnet-4-6                       │
│  - 流式输出（Streaming）                         │
│  - 多轮对话上下文                                │
└──────────────────────────────────────────────────┘
```

## 组件架构

### Server Components（服务端渲染）

| 组件 | 路径 | 说明 |
|------|------|------|
| RootLayout | `app/layout.tsx` | 全局布局，字体，元数据 |
| HomePage | `app/page.tsx` | 首页，静态内容 |
| ChatPage | `app/chat/page.tsx` | 对话页容器，读取 URL 参数 |
| FeaturesPage | `app/features/page.tsx` | 功能介绍，静态内容 |

### Client Components（客户端交互）

| 组件 | 路径 | 说明 |
|------|------|------|
| Navbar | `components/Navbar.tsx` | 导航栏，含路由高亮 |
| ChatInterface | `components/ChatInterface.tsx` | 核心对话 UI，状态管理 |

## 数据流

### 对话数据流

```
用户输入文字
    ↓
[State] input: string
    ↓
用户按 Enter 或点击发送
    ↓
sendMessage(content)
    ↓
[State] messages.push({ role: "user", content })
[State] messages.push({ role: "assistant", content: "" })  ← 占位
    ↓
fetch("POST /api/chat", { messages })
    ↓
ReadableStream (SSE)
    ↓
每收到 chunk → 解析 "data: {...}" → 更新最后一条 assistant 消息
    ↓
收到 "data: [DONE]" → setIsLoading(false)
```

### URL 参数传递

首页的示例提示词通过 URL query 参数传递给对话页：

```
/chat?q=帮我写一段关于人工智能改变未来的科技感文案
    ↓
ChatPage 读取 searchParams.q
    ↓
传给 ChatInterface 的 initialPrompt prop
    ↓
useEffect 自动触发 sendMessage(initialPrompt)
```

## 安全设计

### API Key 保护

```
❌ 错误做法（前端直接调用）：
浏览器 → Anthropic API（暴露 API Key）

✅ 正确做法（后端代理）：
浏览器 → Next.js API Route（无 Key）→ Anthropic API（服务端 Key）
```

API Key 通过 `.env.local` 文件或部署平台的环境变量配置，仅在服务端运行时读取，**永远不会发送到浏览器**。

## 性能优化

### 静态预渲染

首页（`/`）和功能页（`/features`）为纯静态内容，构建时预渲染为 HTML，无需服务端计算。

### 流式响应

对话 API 使用流式输出（Streaming），用户看到第一个 token 的延迟最低，无需等待完整响应再显示。

### Prompt Caching

系统提示词（System Prompt）通过 Anthropic 的 Prompt Cache 机制缓存，后续请求可复用缓存，降低延迟和成本。

## 扩展点

### 添加新的 AI 功能

1. **图像识别**：在 messages 中添加 image content block
2. **文件分析**：使用 Anthropic Files API 上传文件
3. **工具调用**：定义 tools 数组，让 Claude 调用外部 API
4. **结构化输出**：使用 `output_config.format` 获取 JSON 格式响应

### 添加用户系统

1. 集成 NextAuth.js 实现用户认证
2. 使用数据库（如 PostgreSQL + Prisma）持久化对话历史
3. 按用户 ID 隔离对话上下文
