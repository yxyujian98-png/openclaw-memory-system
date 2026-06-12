# OpenClaw 记忆系统

> **让 OpenClaw Agent 拥有持久记忆。**
>
> 自动同步 Obsidian 知识库 → Agent 可搜索。会话观察零成本压缩 → 不丢失。向量库 + 嵌入服务 + 同步链路 → 自动健康监控。

## 它解决什么问题

你用 Obsidian 记了大量笔记，但 OpenClaw Agent 的 `memory_search` 搜不到。
你和 Agent 聊了半天，做了很多决策和发现，对话结束就丢了。
你的 Qdrant 向量库、LM Studio 嵌入服务、vault 同步链路，哪个断了都不知道。

这个技能把这三件事自动化了。

## 快速开始

```bash
# 1. 克隆到 OpenClaw 技能目录
cd ~/.openclaw/workspace/skills
git clone https://github.com/yxyujian98-png/openclaw-memory-system.git
cd openclaw-memory-system

# 2. 安装依赖
pip install -r requirements.txt

# 3. 启动 Qdrant 向量数据库
docker-compose up -d

# 4. 运行安装向导（自动检查依赖、创建配置、初始化 Qdrant 集合）
python scripts/setup.py --vault-dir /path/to/your/vault

# 5. 按 setup.py 输出的指引配置 hooks 和 cron
```

## 前置条件

| 组件 | 必需 | 说明 |
|------|:----:|------|
| Python 3.10+ | ✅ | 脚本运行环境 |
| Qdrant | ✅ | 向量数据库，`docker-compose up -d` 一行启动 |
| 嵌入服务 | ✅ | LM Studio / Ollama / 任何 OpenAI 兼容的 embedding 端点 |
| Obsidian Vault | ✅ | 你的 Markdown 知识库 |
| LLM API | 可选 | 只有高重要性记忆才需要（DeepSeek / OpenAI 等） |

## 配置

### config.json

安装向导会自动创建，你也可以手动编辑 `scripts/config.json`：

```json
{
  "vault_dir": "/path/to/your/vault",
  "llm": {
    "baseUrl": "https://api.deepseek.com/v1",
    "apiKey": "你的 API Key",
    "model": "deepseek-chat"
  },
  "embedder": {
    "baseUrl": "http://localhost:1234/v1",
    "apiKey": "你的嵌入服务 Key",
    "model": "text-embedding-nomic-embed-text-v1.5",
    "embeddingDims": 768
  },
  "qdrant": {
    "host": "localhost",
    "port": 6333,
    "collection": "knowledge_base"
  },
  "sync_dirs": ["01-日记", "02-知识", "04-教训", "07-项目"]
}
```

### 环境变量（替代方案）

```bash
OPENCLAW_VAULT_DIR="/path/to/vault"
OPENCLAW_LMSTUDIO_URL="http://localhost:1234/v1"
OPENCLAW_LMSTUDIO_KEY="你的 Key"
OPENCLAW_QDRANT_HOST="localhost"
OPENCLAW_QDRANT_PORT=6333
```

## 它做了什么

```
你的 Obsidian Vault                          OpenClaw memory_search
  ├── 01-日记/     ── sync_vault_memory.py ──→  memory/*.md (SQLite)
  ├── 02-知识/     ── vault_to_qdrant.py   ──→  Qdrant (向量库)
  ├── 04-教训/
  └── 07-项目/

Agent 工具调用     ── observe.py + compress.py ──→ Qdrant (结构化观察)

每 45 分钟自动：vault 健康检查 → 增量同步 → 记忆压缩 → 健康报告
```

### 核心设计

**零 LLM 成本**：compress.py 用规则驱动（不调 LLM）把工具调用结构化为类型/概念/重要性。只有重要性 ≥ 8 的决策/发现才走 LLM 蒸馏。

**三级嵌入降级**：LM Studio → 本地 ONNX → numpy 哈希。嵌入服务挂了也能跑，搜索质量降级但不崩。

**版本追踪**：Qdrant 向量带 version/is_latest/supersedes 字段，知识演化不冲突。

**PRISM 意图路由**：根据查询意图（事实型/过程型/反思型/时序型）自动选择最优检索策略。

**抗体自愈**：错误模式自动提取为抗体规则，下次遇到同类错误自动匹配修复方案。

## 脚本一览

### 核心管线

| 脚本 | Token 成本 | 功能 |
|------|:----------:|------|
| `shared_config.py` | 0 | 集中配置（环境变量 > config.json > 默认值） |
| `qdrant_utils.py` | 0 | Qdrant 增删改查 |
| `embedder.py` | 0 | 三级降级嵌入 + LRU 缓存（500 条，5 分钟 TTL） |
| `compress.py` | 0 | 零 LLM 观察结构化（类型/概念/重要性/叙述） |
| `observe.py` | 0 | 工具调用观察队列（追加写入 JSONL） |
| `sync_vault_memory.py` | 0 | vault → workspace/memory/ 增量同步 |
| `vault_to_qdrant.py` | 0 | vault → Qdrant 向量同步（按标题分块 + 版本追踪） |
| `extract_memories.py` | 高重要性才用 LLM | 会话压缩 → Qdrant；--full: 概念聚合 + 蒸馏 |
| `unified_memory.py` | 分类用 LLM | Mem0 + Qdrant 统一搜索 + PRISM 路由 |

### 健康与维护

| 脚本 | 功能 |
|------|------|
| `vault_guardian.py` | vault 健康检查 + 过时检测 + 增量同步 + 断链检查 |
| `memory_health.py` | 4 链路健康检查（vault→memory→qdrant→embedding） |
| `lmstudio_guardian.py` | 嵌入服务健康 + 降级模式标记 |
| `maintenance_orchestrator.py` | DAG 任务调度器（拓扑排序 + 并行执行） |
| `context_snapshot.py` | compaction 前上下文快照备份 |
| `compress_to_rule.py` | 执行模式 → 抗体/规则候选提取 |
| `setup.py` | 7 步安装向导 |

### 辅助脚本

| 脚本 | 功能 |
|------|------|
| `vault_watcher.py` | vault 文件变化实时监听（轮询 mtime） |
| `vault_maintainer.py` | vault 自动修复（归档/索引/备份） |
| `process_inbox.py` | 收件箱自动分类归档 |
| `auto_link_vault.py` | 自动补全 + 修复 [[wiki 链接]] |
| `elevate_frequent.py` | vault 活跃文件评分升格 |
| `build_project_profile.py` | 项目 Profile 聚合 |
| `session_cleaner.py` | 过期 session 文件清理 |
| `health_check_v2.py` | 抗体健康巡检 |
| `smoke_test.py` | 跨管线冒烟测试 |
| `heartbeat_alert.py` | 趋势告警 |
| `health_scoreboard.py` | 管道可靠性指标 |
| `system_health.py` | 系统健康状态记录 |

## 运行时流程

### 用户聊天时

1. 用户发消息 → Agent 在会话中处理
2. Agent 调用工具（read/edit/exec/search）→ `observe.py` 注册观察到队列
3. 用户输入 `/new` →
   - `session-memory` hook 保存最近 15 条消息到 `memory/2026-06-12-1000.md`
   - `memory-extract` hook 提取有价值记忆
   - OpenClaw 文件监听器重建索引（1.5s 去抖）
4. 下次 `memory_search` → SQLite 通过混合搜索找到新内容

### Heartbeat 定时任务

Cron 每 45 分钟触发 → routine agent 运行 `maintenance_orchestrator.py --cycle light --parallel`：

```
Level 0（并行）：
  ├── vault_guardian      → vault 过时文件扫描 + 增量同步
  ├── vault_to_qdrant     → vault 变更嵌入 → Qdrant
  ├── extract_memories    → 会话观察压缩 → Qdrant
  ├── memory_health       → 4 链路健康检查
  ├── lmstudio_guardian   → 嵌入服务健康检查
  ├── context_snapshot    → compaction 前上下文备份
  ├── process_inbox       → 收件箱自动分类
  ├── auto_link_vault     → 补全 [[wiki 链接]]
  ├── health_check_v2     → 抗体健康巡检
  ├── smoke_test          → 跨管线冒烟测试
  ├── heartbeat_alert     → 趋势告警
  ├── health_scoreboard   → 管道可靠性指标
  ├── system_snapshot     → 系统状态快照
  └── vault_maintainer    → vault 修复

Level 1（依赖 vault_guardian）：
  └── sync_vault_memory   → vault → memory/ 全量兜底同步
```

## Hook 说明

| Hook | 触发事件 | 作用 |
|------|----------|------|
| `session-memory` | `/new`, `/reset` | 保存最近 15 条消息到 memory/ |
| `memory-compact` | compaction | 上下文压缩前提取记忆 |
| `memory-extract` | `/new`, `/reset` | 提取有价值记忆 |

启用：
```bash
openclaw hooks enable session-memory
openclaw hooks enable memory-compact
openclaw hooks enable memory-extract
```

## memory_search 索引内容

```
~/.openclaw/workspace/
  ├── MEMORY.md                    ← 始终索引，会话开始时注入
  └── memory/
      ├── *.md                     ← 已索引（1544 文件，4777 chunks）
      │   ├── 2026-06-12-1000.md   ← session-memory hook 产出
      │   ├── 02-知识_*.md          ← vault 同步产出
      │   ├── 04-教训_*.md          ← vault 同步产出
      │   └── 07-项目_*.md          ← vault 同步产出
      └── .dreams/                 ← dreaming 系统（默认关闭）

额外索引路径：
  ├── data/skills.memory.md        ← 技能使用记录
  └── ~/self-improving/            ← 执行质量记忆
```

## Qdrant 存储内容

```
Qdrant localhost:6333
  └── knowledge_base 集合
      ├── vault_sync 管线     ← vault markdown 分块（vault_to_qdrant.py）
      ├── compress 管线       ← 工具调用观察（compress.py）
      ├── consolidate 管线    ← 融合概念（extract_memories.py --full）
      └── trajectory 管线     ← JSONL 轨迹中的工具调用
```

## 排障

```bash
# 完整健康检查
python scripts/memory_health.py

# 检查 OpenClaw 记忆索引
openclaw memory status
openclaw memory status --deep

# 检查 Qdrant
curl http://localhost:6333/collections/knowledge_base

# 测试嵌入服务
python scripts/embedder.py "测试文本"

# 强制 vault 同步
python scripts/vault_to_qdrant.py

# 检查配置
python scripts/shared_config.py

# 强制重建索引
openclaw memory index --force

# 检查 hooks
openclaw hooks list

# 检查 cron
openclaw cron list
```

## License

MIT
