---
name: openclaw-memory-system
description: "OpenClaw 记忆系统：Obsidian vault 同步、Qdrant 向量检索、零 LLM 压缩、自愈监控 | Memory system for OpenClaw agents: vault sync, vector search, zero-LLM compression, self-healing"
---

# OpenClaw 记忆系统 / OpenClaw Memory System

> **让 OpenClaw Agent 拥有持久记忆。**
>
> 自动同步 Obsidian 知识库、向量检索、零 LLM 成本压缩、自愈健康监控。
>
> **Give your OpenClaw Agent persistent memory.**
>
> Auto-sync Obsidian vault, vector search, zero-LLM compression, self-healing health monitoring.

## 关键词 / Keywords

记忆系统、长期记忆、向量检索、知识库同步、Obsidian vault、Qdrant、embedding、agent memory、零 LLM 压缩、自愈监控、vault 同步、概念聚合、记忆蒸馏、健康检查

memory system, long-term memory, vector search, knowledge sync, Obsidian vault, Qdrant, embedding, agent memory, zero-LLM compression, self-healing, vault sync, concept consolidation, memory distillation, health monitoring

---

## 中文

### 这是什么

一个 OpenClaw 的记忆增强技能。解决了三个问题：

1. **Vault 内容进不了记忆** — 你用 Obsidian 记了大量笔记，但 Agent 的 `memory_search` 搜不到
2. **会话记忆丢失** — 工具调用、决策、发现等有价值信息，对话结束就没了
3. **记忆系统维护成本高** — 向量库、嵌入服务、同步链路，哪个断了都不知道

### 安装

```bash
cd ~/.openclaw/workspace/skills
git clone https://github.com/yxyujian98-png/vault-memory-system.git
cd openclaw-memory-system
pip install -r requirements.txt
docker-compose up -d
python scripts/setup.py --vault-dir /path/to/vault
```

### 前置条件

| 组件 | 必需 | 说明 |
|------|:----:|------|
| Python 3.10+ | ✅ | 脚本运行环境 |
| Qdrant | ✅ | 向量数据库 |
| 嵌入服务 | ✅ | LM Studio / Ollama / OpenAI 兼容 |
| Obsidian Vault | ✅ | Markdown 知识库 |
| LLM API | 可选 | 高重要性记忆才需要 |

### 运行时数据流

```
┌─────────────────────────────────────────────────────┐
│                Layer 1: OpenClaw 内置                 │
│                                                     │
│  session-memory hook → memory/YYYY-MM-DD-HHMM.md   │
│  memory-compact hook → compaction 前提取记忆          │
│  memory-extract hook → /new、/reset 时提取           │
│                       ↓                             │
│  memory_search ← SQLite (FTS5 + sqlite-vec + 混合)  │
└─────────────────────────────────────────────────────┘
         │ sync_vault_memory.py
         ↓
┌─────────────────────────────────────────────────────┐
│                Layer 2: 自定义脚本                     │
│                                                     │
│  Cron 每 45 分钟 → orchestrator --light --parallel  │
│    → vault_guardian / extract_memories / memory_health│
│    → 12 个任务按 DAG 拓扑并行执行                      │
│                                                     │
│  Qdrant (knowledge_base)                            │
│    → vault 分块 / 工具观察 / 融合概念                  │
└─────────────────────────────────────────────────────┘
```

### 核心设计

- **零 LLM 成本**：compress.py 纯规则驱动
- **三级嵌入降级**：LM Studio → ONNX → numpy 哈希
- **版本追踪**：version / is_latest / supersedes
- **PRISM 意图路由**：事实型 / 过程型 / 反思型 / 时序型
- **抗体自愈**：错误模式 → 自动修复规则

---

## English

### What is this

An OpenClaw memory enhancement skill. Solves three problems:

1. **Vault content not in memory** — You have extensive Obsidian notes, but Agent's `memory_search` can't find them
2. **Session memory lost** — Tool calls, decisions, discoveries — all gone when session ends
3. **Memory system maintenance costly** — Vector DB, embedding service, sync chain — which one broke?

### Installation

```bash
cd ~/.openclaw/workspace/skills
git clone https://github.com/yxyujian98-png/vault-memory-system.git
cd openclaw-memory-system
pip install -r requirements.txt
docker-compose up -d
python scripts/setup.py --vault-dir /path/to/vault
```

### Prerequisites

| Component | Required | Description |
|-----------|:--------:|-------------|
| Python 3.10+ | ✅ | Script runtime |
| Qdrant | ✅ | Vector database |
| Embedding server | ✅ | LM Studio / Ollama / OpenAI-compatible |
| Obsidian Vault | ✅ | Markdown knowledge base |
| LLM API | Optional | Only for high-importance memories |

### Runtime data flow

```
┌─────────────────────────────────────────────────────┐
│                Layer 1: OpenClaw Built-in            │
│                                                     │
│  session-memory hook → memory/YYYY-MM-DD-HHMM.md   │
│  memory-compact hook → extract before compaction    │
│  memory-extract hook → extract on /new, /reset      │
│                       ↓                             │
│  memory_search ← SQLite (FTS5 + sqlite-vec + hybrid)│
└─────────────────────────────────────────────────────┘
         │ sync_vault_memory.py
         ↓
┌─────────────────────────────────────────────────────┐
│                Layer 2: Custom Scripts               │
│                                                     │
│  Cron every 45m → orchestrator --light --parallel   │
│    → vault_guardian / extract_memories / memory_health│
│    → 15 tasks in DAG topological parallel           │
│                                                     │
│  Qdrant (knowledge_base)                            │
│    → vault chunks / tool observations / fused concepts│
└─────────────────────────────────────────────────────┘
```

### Core design

- **Zero LLM cost**: compress.py is purely rule-driven
- **3-level embedding fallback**: LM Studio → ONNX → numpy hash
- **Version tracking**: version / is_latest / supersedes
- **PRISM intent routing**: factual / procedural / reflective / recency
- **Antibody self-healing**: error patterns → auto-fix rules

---

## License

MIT
