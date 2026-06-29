# AI First Demo — 项目文档

AI First Demo 是一个演示 **AI-First 工程实践**的前端示例应用。

## 项目简介

基于 **Vite + React 18 + TypeScript** 构建的单页 Web 应用（SPA），展示在 AI-First harness 约束下如何
开发前端：特性开关、AI 评审、Triage 自愈、Goal Loop 认知护栏。

本文档仓库（`ai-frist-doc`）是跨服务共享的**唯一事实来源**，通过 git submodule 挂到各代码仓的 `docs/`
目录。前端仓 `ai-first-demo` 的仓内 `CLAUDE.md` 描述仓级细节，本仓描述跨服务共识。

## 核心特性

- **特性开关**：每个新功能藏在 `flags/feature-flags.ts` 后，fail-safe 默认关闭，支持 kill switch
- **AI 评审**：每个 PR 过 security / performance / quality 多趟 AI 评审，BLOCK 级问题开发期即拦
- **Triage 自愈**：错误按指纹聚类、九维打分、自动去重建单，goal loop 推到可验证停止条件
- **认知护栏**：comprehension-coverage / pr-read-rate / agent-modification-rate 三项指标防止「没看懂就合并」
- **多模型支持**：模型无关架构，切厂商只需两个 env，详见 [多模型适配](./多模型适配.md)

## 技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| Vite | 5.x | 构建工具，开发服务器 |
| React | 18.x | UI 框架 |
| TypeScript | 5.x（strict） | 类型安全 |
| pnpm | 9.x | 包管理器（workspace） |
| Vitest | latest | 单元测试 |
| Playwright | latest | E2E 测试（关键路径） |

## 仓库结构（多仓体系）

```
ai-first-demo/          ← 前端代码仓（git submodule: docs/ → ai-frist-doc）
ai-frist-doc/           ← 共享 docs-repo（本仓）
```

---

# 快速开始

## 前置要求

- Node.js 20+
- pnpm 9+（`npm i -g pnpm`）

## 安装步骤

```bash
# 克隆含 submodule 的前端仓
git clone --recurse-submodules https://github.com/wluisw/ai-first-demo.git
cd ai-first-demo

# 安装依赖（pnpm workspace）
pnpm install

# 启动开发服务器（Vite，默认 http://localhost:5173）
pnpm dev
```

已克隆但未拉取 submodule：

```bash
git submodule update --init --recursive
```

---

# 项目架构

## 目录结构

```
apps/web/                   # 前端主应用（Vite + React + TS）
  src/
    components/             # 可复用 UI 组件（Hero, PillarCard, MetricsPanel…）
    lib/flags.ts            # 前端侧特性开关读取器（Vite env 注入）
    App.tsx                 # 根组件
    main.tsx                # 挂载入口
  index.html                # HTML 模板
  vite.config.ts            # 构建配置

flags/feature-flags.ts      # 特性开关封装（发布安全阀，集中登记）
docs/                       # ← git submodule → ai-frist-doc（本仓）
.github/workflows/          # CI/CD + AI 评审工作流
scripts/                    # harness 自动化（triage / goal-loop / token 报告）
state/                      # agent 外置记忆（append-only，入仓可审计）
prompts/                    # 任务模板与评审 prompt
.claude/agents/             # sub-agent 角色配置（explorer/implementer/verifier…）
.claude/skills/             # 按域拆分的项目知识（clean-code/secure-coding…）
```

## 开发命令

| 命令 | 说明 |
|------|------|
| `pnpm dev` | 启动 Vite 开发服务器（http://localhost:5173） |
| `pnpm build` | 生产构建 |
| `pnpm test` | Vitest 单元测试 |
| `pnpm typecheck` | TypeScript 类型检查 |
| `pnpm lint` | ESLint 代码规范检查 |

---

# AI-First Harness 说明

## 特性开关

所有新功能必须藏在开关后。发布节奏：

```
团队开（teamOnly: true）
  → 灰度放量（rolloutPct: 5 → 25 → 50）
  → 全量（rolloutPct: 100）
  → 或 kill switch（enabled: false，即时关闭，无需部署）
```

前端读取方式（Vite 环境变量注入）：

```typescript
import { FLAGS, isEnabled } from './lib/flags';

if (isEnabled(FLAGS.LIVE_METRICS_PANEL)) {
  // 功能代码
}
```

开启方式：

```bash
VITE_FLAG_live_metrics_panel=true pnpm dev
```

## Sub-Agents

角色化 agent，见 `.claude/agents/`：

| Agent | 模型层 | 职责 |
|-------|--------|------|
| explorer | Haiku | 快速定位代码 |
| implementer | Sonnet | 写代码 |
| verifier-security | Sonnet | 安全评审 |
| verifier-quality | Sonnet | 质量评审 |
| verifier-performance | Sonnet | 性能评审 |
| verifier-dependency | Sonnet | 依赖评审 |
| checker | Sonnet | 判定 done（独立于 implementer） |
| triage-scorer | Sonnet | 九维打分 |

## Skills

项目知识按域拆分，见 `.claude/skills/`。规范 skill 强制加载规则：

| 场景 | 必读 skill |
|------|-----------|
| 用户输入 / 鉴权 / 密钥 | `secure-coding` |
| 外部调用 / 缓存 / 日志 | `performance-review` |
| 改接口 | `api-doc-output` |
| 改数据模型 | `data-model-output` |
| 任何改动底线 | `clean-code` + `testing-standards` + `feature-flag-setup` |

## Goal Loop

`scripts/goal_loop.py`：implementer 推一步 → checker（独立 sub-agent）判定 done → 循环直到停止条件成立。长任务、回归修复、CI 自愈都套这个范式。

## State（外置记忆）

`state/` 目录，append-only，入仓可审计：

- `triage-history.jsonl` — triage 去重历史
- `token-usage.jsonl` — token 花费记录
- `comprehension-log.jsonl` — 认知护栏指标
- `tasks/<id>.json` — 任务状态
- `known-flakes.txt` — 已知 flake 测试（自动降权）

---

# 环境变量

| 变量名 | 必填 | 说明 |
|--------|------|------|
| `LLM_PROVIDER` | 否 | 模型厂商（默认 `anthropic`），详见 [多模型适配](./多模型适配.md) |
| `LLM_API_KEY` | 是 | 对应厂商的 API Key |
| `STATSIG_SERVER_SECRET` | 否 | Statsig 特性开关 provider（省略则用本地 LocalProvider） |
| `VITE_FLAG_<key>` | 否 | 构建时注入前端特性开关（如 `VITE_FLAG_live_metrics_panel=true`） |

---

# 部署

合并到 `main` 后六阶段流水线自动部署（见 `.github/workflows/deploy.yml`）：

1. 构建（`pnpm build`）
2. 类型检查 + lint
3. 单元测试（Vitest）
4. E2E 测试（Playwright）
5. AI 评审门禁（security / quality / performance / dependency）
6. 发布 + 熔断监控（指标恶化自动回退）

---

# 更新日志

## v0.2.0 (2026-06-30)

- 切换为 Vite + React 18 + TypeScript 单页应用架构
- 引入 AI-First harness（特性开关、AI 评审、Triage 自愈、Goal Loop）
- 多模型支持（Anthropic / OpenAI / DeepSeek / Qwen / Kimi / GLM）
- 共享 docs-repo 以 git submodule 形式挂载（`docs/`）
- Sub-agent 角色化配置（explorer / implementer / verifier / checker）

## v0.1.0 (2026-06-29)

- 初始 Next.js 原型（已废弃，切换为 Vite 架构）
