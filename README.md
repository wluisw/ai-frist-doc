# AI First Demo — 项目文档

AI First Demo 是一个展示如何使用 Anthropic Claude API 构建现代化 AI 应用的前端演示项目。

## 项目简介

基于 Next.js 16 + React 19 + TypeScript 构建，集成 Claude claude-sonnet-4-6 模型，实现了实时流式 AI 对话功能。

## 功能特性

- **AI 对话**：基于 Claude API 的多轮上下文对话，支持实时流式输出
- **现代化 UI**：玻璃态设计，深色主题，响应式布局
- **安全架构**：API Key 保存在服务端，前端不暴露任何密钥
- **流式响应**：SSE (Server-Sent Events) 实现 token 级实时推送

## 技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| Next.js | 16.x | 全栈框架，App Router |
| React | 19.x | UI 框架 |
| TypeScript | 5.x | 类型安全 |
| Tailwind CSS | 4.x | 样式系统 |
| @anthropic-ai/sdk | latest | Claude API 客户端 |

## 仓库结构

```
ai-first-demo/          # 主前端项目
ai-frist-doc/           # 项目文档
```

---

# 快速开始

## 前置要求

- Node.js 18+
- npm 或 pnpm
- Anthropic API Key（[获取地址](https://console.anthropic.com)）

## 安装步骤

```bash
# 克隆主项目
git clone https://github.com/wluisw/ai-first-demo.git
cd ai-first-demo

# 安装依赖
npm install

# 配置环境变量
cp .env.local.example .env.local
# 编辑 .env.local，填入你的 ANTHROPIC_API_KEY

# 启动开发服务器
npm run dev
```

访问 [http://localhost:3000](http://localhost:3000) 查看效果。

---

# 项目架构

## 目录结构

```
src/
├── app/
│   ├── layout.tsx          # 根布局
│   ├── page.tsx            # 首页（Landing Page）
│   ├── chat/
│   │   └── page.tsx        # AI 对话页面
│   ├── features/
│   │   └── page.tsx        # 功能特性页面
│   ├── api/
│   │   └── chat/
│   │       └── route.ts    # Claude API 代理路由
│   └── globals.css         # 全局样式
└── components/
    ├── Navbar.tsx           # 导航栏
    └── ChatInterface.tsx    # 聊天界面客户端组件
```

## 核心流程

```
用户输入
    ↓
ChatInterface (Client Component)
    ↓
POST /api/chat (Next.js Route Handler)
    ↓
Anthropic Claude API (流式)
    ↓
SSE Stream → 前端实时渲染
```

---

# API 参考

## POST /api/chat

处理 AI 对话请求，返回流式 SSE 响应。

### 请求

```json
{
  "messages": [
    { "role": "user", "content": "你好" },
    { "role": "assistant", "content": "你好！有什么我可以帮到你的？" },
    { "role": "user", "content": "帮我写一首诗" }
  ]
}
```

### 响应

SSE 流格式：

```
data: {"text": "春"}
data: {"text": "风"}
data: {"text": "送"}
data: [DONE]
```

### 错误响应

```
data: {"error": "错误信息"}
```

---

# 环境变量

| 变量名 | 必填 | 说明 |
|--------|------|------|
| `ANTHROPIC_API_KEY` | 是 | Anthropic API 密钥 |

---

# 部署

## Vercel 部署（推荐）

1. Fork 仓库到你的 GitHub
2. 在 [Vercel](https://vercel.com) 导入项目
3. 在 Environment Variables 中添加 `ANTHROPIC_API_KEY`
4. 点击 Deploy

## 自托管

```bash
# 构建生产版本
npm run build

# 启动生产服务器
npm start
```

---

# 开发指南

## 添加新功能

### 扩展 AI 能力

修改 `src/app/api/chat/route.ts` 中的系统提示词和模型参数：

```typescript
const anthropicStream = await client.messages.stream({
  model: "claude-sonnet-4-6",   // 或 "claude-opus-4-7"
  max_tokens: 8192,
  system: "你的自定义系统提示词",
  messages,
});
```

### 添加新页面

在 `src/app/` 下创建新目录，添加 `page.tsx` 文件，然后在 `Navbar.tsx` 中添加导航链接。

## 代码规范

- 所有组件使用 TypeScript
- 客户端交互组件使用 `"use client"` 指令
- 样式使用 Tailwind CSS 工具类
- API 路由放在 `src/app/api/` 下

---

# 更新日志

## v0.1.0 (2025-06-30)

- 初始项目搭建
- 实现 Claude API 流式对话
- 创建首页、对话页、功能页
- 完成基础 UI 设计
