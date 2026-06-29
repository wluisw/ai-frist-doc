# 系统架构文档

## 整体架构

AI First Demo 采用 **Vite + React 18 单页应用** 架构，配合 AI-First harness 运行在多仓体系中。

```
┌─────────────────────────────────────────────────────────────┐
│                      代码仓体系                              │
│                                                             │
│  ai-first-demo/              ai-frist-doc/（本文档仓）       │
│  ├── apps/web/   ←──────────── docs/ submodule ────────────→│
│  ├── flags/                                                 │
│  ├── scripts/                接口契约 / 数据模型             │
│  ├── .claude/                架构决策 / 多模型适配           │
│  └── .github/                                               │
└─────────────────────────────────────────────────────────────┘
```

## 前端应用架构（apps/web）

```
┌─────────────────────────────────────────────────────────────┐
│                    浏览器（SPA）                             │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    App.tsx                           │   │
│  │  ┌──────────┐  ┌────────────┐  ┌─────────────────┐  │   │
│  │  │  Hero    │  │ PillarCard │  │  MetricsPanel   │  │   │
│  │  │          │  │  ×4        │  │  (feature flag) │  │   │
│  │  └──────────┘  └────────────┘  └─────────────────┘  │   │
│  │                                       ↑              │   │
│  │                          isEnabled(FLAGS.LIVE_METRICS)│  │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  lib/flags.ts ← VITE_FLAG_* 环境变量（构建时注入）           │
└─────────────────────────────────────────────────────────────┘
                              │
                              │（未来：REST / GraphQL / SSE）
                              ↓
                     后端服务（独立仓）
```

## AI-First Harness 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    AI-First Harness                         │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  特性开关     │  │   AI 评审    │  │   Triage 自愈    │  │
│  │ flags/       │  │ .github/     │  │ scripts/         │  │
│  │ feature-     │  │ workflows/   │  │ triage_engine.py │  │
│  │ flags.ts     │  │ ai-review.yml│  │ goal_loop.py     │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│         │                 │                  │              │
│         ↓                 ↓                  ↓              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                   Sub-Agents                         │   │
│  │  explorer(Haiku) → implementer(Sonnet)               │   │
│  │  verifier-{security,quality,perf,dep}(Sonnet)        │   │
│  │  checker(Sonnet) — 独立判 done，不能是写代码的        │   │
│  │  triage-scorer(Sonnet) — 九维打分                    │   │
│  └──────────────────────────────────────────────────────┘   │
│         │                                                   │
│         ↓                                                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              State（外置记忆，append-only）           │   │
│  │  triage-history.jsonl / token-usage.jsonl            │   │
│  │  comprehension-log.jsonl / tasks/<id>.json           │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 特性开关数据流

```
代码仓中集中登记 flags/feature-flags.ts
           │
           ├── 后端：FlagProvider（Statsig / LocalProvider）
           │     └── flags.isEnabled(FLAGS.XXX, { userId })
           │
           └── 前端：VITE_FLAG_<key>=true pnpm build
                     └── isEnabled(FLAGS.XXX) 读 import.meta.env
```

## Goal Loop 流程

```
架构师建 Task（prompts/architect-task.md）
    ↓
implementer sub-agent 推一步变更
    ↓
checker sub-agent（独立）判定：done? ──→ 是 → 停止
    ↓ 否
回到 implementer（下一轮）
    ↓
超时 / 最大轮次 → 告警 + 人工介入
```

## CI/CD 流水线（.github/workflows）

```
push → main
    ↓
ci.yml: build + typecheck + lint + test(Vitest)
    ↓
ai-review.yml: security-verifier + quality-verifier
             + performance-verifier + dependency-verifier
    ↓（均 PASS）
deploy.yml: 六阶段部署
    ├── 1. pnpm build
    ├── 2. 静态资产上传 CDN
    ├── 3. Playwright E2E smoke
    ├── 4. 指标基线采集
    ├── 5. 流量切换（蓝绿）
    └── 6. 熔断监控（指标恶化 → 自动回退）
    ↓
daily-health.yml: token_report + comprehension_metrics + health_report
triage.yml: 错误聚类 → 九维打分 → 去重建单
```

## 多仓 docs 共享机制

```
ai-first-demo/
└── docs/              ← git submodule
    └── ────────────→  https://github.com/wluisw/ai-frist-doc.git
                           ├── README.md          # 项目总览
                           ├── architecture.md    # 架构（本文件）
                           └── 多模型适配.md      # 模型无关接入指南
```

各代码仓通过 `git submodule update --init --recursive` 拉取最新文档。
文档更新后，各仓需 `git submodule update --remote docs` 跟进。

## 安全设计

```
❌ 禁止：前端硬编码 API Key / Token
✅ 要求：
  - LLM_API_KEY 存服务端环境变量（.env.local / secrets）
  - 前端只读 VITE_FLAG_* 构建时注入值，不含任何密钥
  - 所有外部请求走后端代理，前端不直调 LLM API
  - 用户输入一律参数化 / 转义，不拼入 SQL / shell / 模板
  - 新端点默认鉴权，IDOR 零容忍
```

## 扩展点

### 添加新特性开关

1. 在 `flags/feature-flags.ts` 的 `FLAGS` 对象中登记新 key
2. 在 `FLAG_DEFAULTS` 里设保守默认值（`false`）
3. 业务代码调用 `flags.isEnabled(FLAGS.NEW_KEY, ctx)` / 前端调用 `isEnabled(FLAGS.NEW_KEY)`
4. PR 描述里注明 flag 名与灰度计划

### 切换 LLM 厂商

只需设置两个环境变量，代码不动：

```bash
LLM_PROVIDER=openai
LLM_API_KEY=sk-...
```

详见 [多模型适配](./多模型适配.md)。

### 添加新 Sub-Agent

在 `.claude/agents/` 下新建 `<name>.toml`，参考现有 agent 配置。
