# AgentMemory 学习教程

本教程将帮助你从零开始，逐步深入地掌握 AgentMemory 项目的核心概念、架构设计和实现细节。

---

## 第一阶段：环境准备

### 1.1 基础环境要求

在开始学习 AgentMemory 之前，请确保你具备以下基础知识：

| 知识点 | 掌握程度 | 学习资源推荐 |
|--------|----------|--------------|
| TypeScript | 基础到中级 | [官方文档](https://www.typescriptlang.org/docs/) |
| Node.js | 基础 | [Node.js 入门](https://nodejs.org/en/learn) |
| Git | 基础操作 | [Git 教程](https://git-scm.com/doc) |
| SQLite | 了解 | [SQLite 教程](https://www.sqlite.org/docs.html) |
| AI/LLM 基础 | 了解 | 相关概念可自行搜索 |

### 1.2 安装配置

#### 步骤 1：克隆项目

```bash
git clone https://github.com/aifec001/localmemory.git
cd localmemory
```

#### 步骤 2：安装依赖

```bash
npm install
```

#### 步骤 3：配置环境变量

创建配置文件：

```bash
mkdir -p ~/.agentmemory
cat > ~/.agentmemory/.env << 'EOF'
# LLM 提供者配置（选择一个）
ANTHROPIC_API_KEY=your_api_key_here
# 或者
GEMINI_API_KEY=your_api_key_here
# 或者
OPENROUTER_API_KEY=your_api_key_here

# 可选：嵌入提供者（推荐使用本地免费版本）
EMBEDDING_PROVIDER=local

# 可选：功能开关
GRAPH_EXTRACTION_ENABLED=true
CONSOLIDATION_ENABLED=true
EOF
```

#### 步骤 4：构建项目

```bash
npm run build
```

#### 步骤 5：验证安装

```bash
npx @agentmemory/agentmemory doctor
```

---

## 第二阶段：核心概念

### 2.1 什么是 AgentMemory？

**一句话理解**：AgentMemory 是一个"记忆数据库"，让 AI 编程助手能够跨会话记住项目的上下文和之前的决策。

**核心价值**：想象你每次找新同事合作都要重新解释项目的技术栈、代码规范和之前的决策。AgentMemory 就是让 AI 避免这种重复解释的工具。

### 2.2 核心概念图谱

```
┌─────────────────────────────────────────────────────────────────┐
│                        AgentMemory 架构                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │  AI 编程助手  │    │    MCP       │    │   Hooks      │      │
│  │  (Claude等)   │───▶│  Protocol    │───▶│  (生命周期)   │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│         │                                       │                │
│         ▼                                       ▼                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      记忆引擎                             │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │
│  │  │ 观察捕获 │  │  压缩    │  │  索引    │  │  检索    │    │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                               │                                  │
│         ┌─────────────────────┼─────────────────────┐          │
│         ▼                     ▼                     ▼          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │  BM25 索引  │    │ 向量索引    │    │ 知识图谱    │        │
│  └─────────────┘    └─────────────┘    └─────────────┘        │
│                               │                                  │
│                               ▼                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    SQLite 数据库                           │   │
│  │  Sessions │ Observations │ Memories │ Graph │ Profiles   │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 数据结构详解

#### 观察（Observation）

观察是 AI 执行操作时的记录，类似于"快照"：

```typescript
// 查看源代码：src/types.ts (第 30-64 行)
interface RawObservation {
  id: string;                    // 唯一标识
  sessionId: string;             // 所属会话
  timestamp: string;             // 时间戳
  hookType: HookType;            // 触发钩子类型
  toolName?: string;             // 工具名称
  toolInput?: unknown;           // 工具输入
  toolOutput?: unknown;          // 工具输出
  userPrompt?: string;           // 用户提示
  // ...更多字段
}

interface CompressedObservation {
  // 压缩后的可搜索结构
  id: string;
  sessionId: string;
  type: ObservationType;         // file_read, file_write, command_run...
  title: string;                // 简短标题
  facts: string[];              // 提取的事实
  narrative: string;            // 叙述性描述
  concepts: string[];           // 概念标签
  files: string[];              // 相关文件
  importance: number;           // 重要度评分
}
```

#### 记忆（Memory）

记忆是重要的、长期保存的知识单元：

```typescript
// 查看源代码：src/types.ts (第 83-105 行)
interface Memory {
  id: string;
  type: "pattern" | "preference" | "architecture" | "bug" | "workflow" | "fact";
  title: string;                 // 记忆标题
  content: string;               // 记忆内容
  concepts: string[];            // 概念标签
  files: string[];               // 相关文件
  sessionIds: string[];          // 来源会话
  strength: number;              // 重要度 (0-100)
  version: number;              // 版本号
  isLatest: boolean;             // 是否最新
  // 版本管理
  parentId?: string;
  supersedes?: string[];
}
```

#### 会话（Session）

会话是一次 AI 工作会话的容器：

```typescript
// 查看源代码：src/types.ts (第 1-15 行)
interface Session {
  id: string;
  project: string;               // 项目路径
  cwd: string;                   // 工作目录
  startedAt: string;
  endedAt?: string;
  status: "active" | "completed" | "abandoned";
  observationCount: number;
  model?: string;
  summary?: string;              // 会话摘要
}
```

### 2.4 钩子（Hooks）系统

钩子是 AI 代理生命周期中的触发点，AgentMemory 在这些时机自动捕获信息：

| 钩子名称 | 触发时机 | 捕获内容 |
|----------|----------|----------|
| SessionStart | 会话开始 | 项目路径、会话ID |
| UserPromptSubmit | 用户发送提示 | 用户指令 |
| PreToolUse | 工具执行前 | 文件访问意图 |
| PostToolUse | 工具执行后 | 工具输入输出 |
| PostToolFailure | 工具执行失败 | 错误信息 |
| PreCompact | 压缩前 | 上下文 |
| Stop | 会话停止 | 结束摘要 |
| SessionEnd | 会话结束 | 完成标记 |

---

## 第三阶段：项目结构探索

### 3.1 目录结构速览

```
/workspace/
├── src/                          # 核心源代码
│   ├── index.ts                  # 主入口
│   ├── config.ts                 # 配置管理
│   ├── types.ts                  # 类型定义
│   ├── cli.ts                    # CLI 命令
│   │
│   ├── functions/                # 核心功能函数 (70+ 个)
│   │   ├── observe.ts            # 观察捕获
│   │   ├── compress.ts           # LLM 压缩
│   │   ├── search.ts             # 搜索功能
│   │   ├── smart-search.ts       # 智能搜索
│   │   ├── remember.ts           # 记忆保存
│   │   ├── forget.ts             # 记忆删除
│   │   ├── consolidate.ts        # 记忆整合
│   │   ├── graph.ts              # 知识图谱
│   │   └── ...
│   │
│   ├── state/                    # 状态管理
│   │   ├── kv.ts                 # 键值存储
│   │   ├── search-index.ts       # BM25 索引
│   │   ├── vector-index.ts       # 向量索引
│   │   └── hybrid-search.ts      # 混合搜索
│   │
│   ├── hooks/                    # Hook 处理器
│   ├── mcp/                      # MCP 服务
│   ├── providers/                # LLM 提供者
│   ├── viewer/                   # 可视化服务器
│   └── cli/connect/              # 各代理连接器
│
├── test/                         # 测试文件
├── plugin/                       # 插件配置
└── package.json                  # 项目配置
```

### 3.2 核心文件解读

#### 主入口：src/index.ts

**任务**：理解项目的初始化流程

```typescript
// 查看源代码：src/index.ts (第 159-100 行)
// 这是 worker 进程的主函数

async function main() {
  // 1. 加载配置
  const config = loadConfig();
  
  // 2. 创建 LLM 提供者
  const provider = createProvider(config.provider);
  
  // 3. 创建嵌入提供者
  const embeddingProvider = createEmbeddingProvider();
  
  // 4. 注册 iii-sdk worker
  const sdk = registerWorker(config.engineUrl, {...});
  
  // 5. 注册核心函数
  registerObserveFunction(sdk, kv, dedupMap, ...);
  registerSearchFunction(sdk, kv);
  registerSmartSearchFunction(sdk, kv, ...);
  // ... 更多函数
  
  // 6. 启动可视化服务器
  startViewerServer(viewerPort, kv, sdk, ...);
}
```

**练习**：阅读 [src/index.ts](file:///workspace/src/index.ts#L159-L100) 源码，标记每个 `register*Function` 调用的作用。

#### 配置管理：src/config.ts

**任务**：理解配置加载机制

```typescript
// 查看源代码：src/config.ts (第 52-157 行)
// detectProvider 函数展示了提供者检测优先级

function detectProvider(env: Record<string, string>): ProviderConfig {
  // 优先级顺序：
  if (hasRealValue(env["OPENAI_API_KEY"])) return {...};      // 1. OpenAI
  if (hasRealValue(env["MINIMAX_API_KEY"])) return {...};     // 2. MiniMax
  if (hasRealValue(env["ANTHROPIC_API_KEY"])) return {...};   // 3. Anthropic
  if (hasRealValue(env["GEMINI_API_KEY"])) return {...};      // 4. Gemini
  if (hasRealValue(env["OPENROUTER_API_KEY"])) return {...};  // 5. OpenRouter
  // 默认：noop (无操作)
  return { provider: "noop", ... };
}
```

**练习**：修改 `detectProvider` 的优先级顺序，观察变化。

#### 类型定义：src/types.ts

**任务**：掌握核心数据结构

建议按顺序阅读以下接口定义：

1. `Session` (第 1-15 行)
2. `RawObservation` / `CompressedObservation` (第 30-64 行)
3. `Memory` (第 83-105 行)
4. `HookType` (第 119-131 行)
5. `HybridSearchResult` (第 254-262 行)

---

## 第四阶段：核心功能实践

### 4.1 观察捕获流程

**目标**：理解数据如何从 AI 代理流入系统

#### 源码路径
- `src/functions/observe.ts`

#### 执行流程

```
PostToolUse 钩子触发
    │
    ▼
┌─────────────────────────────────────────┐
│ 1. 去重检查 (DedupMap)                   │
│    - SHA-256 计算内容哈希                │
│    - 5分钟窗口内重复检查                 │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 2. 隐私过滤 (Privacy Filter)            │
│    - 移除 API 密钥                       │
│    - 移除 <private> 标签内容             │
│    - 移除敏感配置                        │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 3. 存储原始观察                          │
│    - 写入 KV store                       │
│    - 生成唯一 ID                         │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 4. LLM 压缩 (可选)                      │
│    - 生成结构化 facts                     │
│    - 提取 concepts                       │
│    - 编写 narrative                      │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 5. 向量化 + 索引                         │
│    - 生成嵌入向量                        │
│    - BM25 索引                          │
│    - 向量索引                            │
└─────────────────────────────────────────┘
```

#### 关键代码片段

```typescript
// src/functions/observe.ts (约第 100-200 行)
// 观察处理的核心逻辑

async function processObservation(hook: HookPayload) {
  // 1. 去重检查
  const hash = computeHash(hook.data);
  if (dedupMap.has(hash)) return; // 跳过重复
  
  // 2. 隐私过滤
  const filtered = privacyFilter(hook.data);
  
  // 3. 存储原始数据
  const rawObs = {
    id: generateId(),
    sessionId: hook.sessionId,
    timestamp: hook.timestamp,
    hookType: hook.hookType,
    toolName: hook.data.toolName,
    toolInput: filtered.input,
    toolOutput: filtered.output,
    // ...
  };
  await kv.set(`obs:${rawObs.id}`, rawObs);
  
  // 4. LLM 压缩 (如果启用)
  if (isAutoCompressEnabled()) {
    const compressed = await compressWithLLM(rawObs);
    await kv.set(`comp:${rawObs.id}`, compressed);
  }
  
  // 5. 索引
  await indexObservation(rawObs);
}
```

**练习**：
1. 在 [src/functions/observe.ts](file:///workspace/src/functions/observe.ts) 中找到去重逻辑
2. 理解 `DedupMap` 的实现
3. 测试观察捕获流程

### 4.2 混合搜索系统

**目标**：理解如何检索记忆

#### 搜索架构

```
用户查询
    │
    ▼
┌─────────────────────────────────────────┐
│         查询处理                         │
│  - 查询扩展 (可选)                       │
│  - 同义词处理                            │
│  - CJK 分词 (可选)                       │
└─────────────────────────────────────────┘
    │
    ├─────────────────┬─────────────────┐
    ▼                 ▼                 ▼
┌─────────┐    ┌─────────┐    ┌─────────┐
│  BM25   │    │ 向量    │    │  图谱   │
│  搜索   │    │ 搜索    │    │  搜索   │
└────┬────┘    └────┬────┘    └────┬────┘
     │              │              │
     ▼              ▼              ▼
┌─────────┐    ┌─────────┐    ┌─────────┐
│ 分数 1  │    │ 分数 2  │    │ 分数 3  │
└────┬────┘    └────┬────┘    └────┬────┘
     │              │              │
     └──────────────┼──────────────┘
                    ▼
         ┌─────────────────┐
         │  RRF 融合算法   │
         │ 1/(k+rank)     │
         └────────┬────────┘
                  ▼
         ┌─────────────────┐
         │    结果排序      │
         │  (session 多样化) │
         └────────┬────────┘
                  ▼
              Top-K 结果
```

#### 源码路径
- `src/state/hybrid-search.ts` - 混合搜索核心
- `src/state/search-index.ts` - BM25 实现
- `src/state/vector-index.ts` - 向量索引

#### RRF 融合算法

```typescript
// src/state/hybrid-search.ts (约第 50-100 行)

// 互惠排序融合 (Reciprocal Rank Fusion)
function rrfFusion(results: SearchResult[][], k = 60): SearchResult[] {
  const scoreMap = new Map<string, number>();
  
  for (const resultList of results) {
    for (let rank = 0; rank < resultList.length; rank++) {
      const item = resultList[rank];
      const score = 1 / (k + rank + 1);  // RRF 公式
      const current = scoreMap.get(item.id) || 0;
      scoreMap.set(item.id, current + score);
    }
  }
  
  // 按融合分数排序
  return Array.from(scoreMap.entries())
    .sort((a, b) => b[1] - a[1])
    .map(([id]) => findById(id));
}
```

#### BM25 算法要点

BM25 是一种改进的词频-逆文档频率算法：

```typescript
// src/state/search-index.ts (约第 100-200 行)

function bm25Score(doc: string, query: string, avgDL: number): number {
  const k1 = 1.5;   // 词频饱和参数
  const b = 0.75;   // 文档长度归一化参数
  
  let score = 0;
  const docLen = doc.length;
  
  for (const term of query.split(/\s+/)) {
    const tf = countTermFrequency(term, doc);
    const idf = computeIDF(term);  // 逆文档频率
    const tfComponent = (tf * (k1 + 1)) / (tf + k1 * (1 - b + b * docLen / avgDL));
    score += idf * tfComponent;
  }
  
  return score;
}
```

**练习**：
1. 阅读 [src/state/hybrid-search.ts](file:///workspace/src/state/hybrid-search.ts)
2. 实现一个简单的 BM25 搜索
3. 测试不同 k 值对 RRF 结果的影响

### 4.3 记忆生命周期

**目标**：理解记忆如何创建、更新和遗忘

#### 记忆流程

```
创建 (Observe/Remember)
    │
    ▼
┌─────────────────────────────────────────┐
│  Working Memory (工作记忆)               │
│  - 原始观察                              │
│  - 短期保留                              │
└─────────────────────────────────────────┘
    │
    ▼ (Consolidation Pipeline)
┌─────────────────────────────────────────┐
│  Episodic Memory (事件记忆)              │
│  - 会话摘要                              │
│  - 时间线                                │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│  Semantic Memory (语义记忆)               │
│  - 提取的事实                            │
│  - 模式识别                              │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│  Procedural Memory (程序记忆)            │
│  - 工作流程                              │
│  - 决策模式                              │
└─────────────────────────────────────────┘
```

#### 遗忘机制

```typescript
// src/functions/auto-forget.ts

interface MemoryWithScore {
  memory: Memory;
  retentionScore: number;  // 保留分数
}

// Ebbinghaus 遗忘曲线模型
function computeDecay(lastAccessed: Date, strength: number): number {
  const daysSinceAccess = (Date.now() - lastAccessed.getTime()) / (1000 * 60 * 60 * 24);
  const lambda = 0.05;  // 衰减率
  return strength * Math.exp(-lambda * daysSinceAccess);
}

async function autoForget(dryRun: boolean = false) {
  const memories = await kv.list<Memory>(KV.memories);
  
  for (const memory of memories) {
    // 计算保留分数
    const decayScore = computeDecay(new Date(memory.lastAccessedAt), memory.strength);
    const reinforcement = memory.accessCount * 0.1;  // 访问强化
    const retentionScore = decayScore + reinforcement;
    
    if (retentionScore < 10) {  // 阈值
      if (!dryRun) {
        await kv.delete(`memory:${memory.id}`);
        await audit("forget", memory.id);
      }
    }
  }
}
```

**练习**：
1. 阅读 [src/functions/consolidate.ts](file:///workspace/src/functions/consolidate.ts)
2. 阅读 [src/functions/auto-forget.ts](file:///workspace/src/functions/auto-forget.ts)
3. 理解遗忘曲线的参数调节

---

## 第五阶段：高级特性

### 5.1 知识图谱

知识图谱存储实体之间的关系，用于更智能的检索。

#### 图谱结构

```typescript
// src/types.ts (第 366-442 行)

interface GraphNode {
  id: string;
  type: "file" | "function" | "concept" | "error" | "decision" | ...;
  name: string;
  properties: Record<string, unknown>;
  sourceObservationIds: string[];
}

interface GraphEdge {
  id: string;
  type: "uses" | "imports" | "modifies" | "causes" | "fixes" | ...;
  sourceNodeId: string;
  targetNodeId: string;
  weight: number;
  confidence?: number;
}
```

#### 图谱检索

```typescript
// src/functions/graph.ts

async function queryGraph(
  entity: string,
  depth: number = 2
): Promise<GraphQueryResult> {
  // 1. 查找起始节点
  const startNode = await findNodeByName(entity);
  if (!startNode) return { nodes: [], edges: [], depth: 0 };
  
  // 2. BFS 遍历
  const visited = new Set<string>();
  const queue: string[] = [startNode.id];
  const result: GraphQueryResult = { nodes: [startNode], edges: [], depth: 0 };
  
  while (queue.length > 0 && result.depth < depth) {
    const currentDepth = queue.length;
    for (let i = 0; i < currentDepth; i++) {
      const nodeId = queue.shift()!;
      if (visited.has(nodeId)) continue;
      visited.add(nodeId);
      
      // 查找关联边和节点
      const edges = await getEdgesFromNode(nodeId);
      for (const edge of edges) {
        result.edges.push(edge);
        const targetNode = await getNode(edge.targetNodeId);
        if (targetNode && !visited.has(targetNode.id)) {
          result.nodes.push(targetNode);
          queue.push(targetNode.id);
        }
      }
    }
    result.depth++;
  }
  
  return result;
}
```

### 5.2 MCP 服务

MCP (Model Context Protocol) 是 AgentMemory 与 AI 代理通信的标准方式。

#### 工具列表

AgentMemory 提供 53+ 个 MCP 工具：

| 类别 | 工具示例 |
|------|----------|
| 核心 | `memory_recall`, `memory_save`, `memory_smart_search` |
| 管理 | `memory_sessions`, `memory_timeline`, `memory_profile` |
| 高级 | `memory_consolidate`, `memory_graph_query`, `memory_team_share` |
| 治理 | `memory_audit`, `memory_governance_delete` |

#### 实现位置

- `src/mcp/server.ts` - MCP 服务端
- `src/mcp/tools-registry.ts` - 工具注册表

### 5.3 多代理支持

#### 连接适配器

```typescript
// src/cli/connect/

// 支持的代理类型
const agents = [
  "claude-code",    // Claude Code
  "copilot-cli",    // GitHub Copilot CLI
  "codex",          // OpenAI Codex
  "cursor",         // Cursor
  "gemini-cli",     // Google Gemini CLI
  "continue",       // Continue.dev
  "zed",            // Zed
  "warp",           // Warp
  // ... 30+ 更多
] as const;

type AgentType = typeof agents[number];
```

#### 连接流程

```bash
# 基本用法
agentmemory connect claude-code

# 带 hooks 的完整连接
agentmemory connect claude-code --with-hooks

# 查看支持的所有代理
agentmemory connect --list
```

---

## 第六阶段：测试与调试

### 6.1 运行测试

```bash
# 运行所有单元测试
npm test

# 运行特定测试文件
npm test -- test/search.test.ts

# 运行带监视模式的测试
npm run test:watch

# 运行集成测试
npm run test:integration
```

### 6.2 调试技巧

#### 启用详细日志

```bash
# 查看详细日志
npx @agentmemory/agentmemory --verbose

# 查看特定日志
DEBUG=agentmemory:* npx @agentmemory/agentmemory
```

#### 健康检查

```bash
# API 健康检查
curl http://localhost:3111/agentmemory/health

# 检查连接状态
npx @agentmemory/agentmemory doctor
```

### 6.3 常用调试端点

```bash
# 会话列表
curl http://localhost:3111/agentmemory/sessions

# 观察列表
curl "http://localhost:3111/agentmemory/observations?sessionId=<id>"

# 搜索测试
curl -X POST http://localhost:3111/agentmemory/smart-search \
  -H "Content-Type: application/json" \
  -d '{"query": "JWT auth", "limit": 5}'
```

---

## 第七阶段：贡献指南

### 7.1 代码风格

- 使用 TypeScript 严格模式
- 遵循项目现有的命名约定
- 添加 JSDoc 注释（除非被要求不添加）
- 确保测试覆盖新功能

### 7.2 提交流程

```bash
# 1. 创建功能分支
git checkout -b feature/your-feature-name

# 2. 开发并测试

# 3. 运行 lint 和测试
npm run lint  # 如果存在
npm test

# 4. 提交
git add .
git commit -m "feat: add your feature description"

# 5. 推送并创建 PR
git push origin feature/your-feature-name
```

### 7.3 常用脚本

```bash
# 构建
npm run build

# 类型检查
npx tsc --noEmit

# 代码格式化 (如果有)
npx prettier --write src/**/*.ts
```

---

## 附录：快速参考

### A.1 常用环境变量

```bash
# API 密钥
ANTHROPIC_API_KEY=sk-ant-...
GEMINI_API_KEY=...
OPENAI_API_KEY=...

# 嵌入提供者
EMBEDDING_PROVIDER=local  # 推荐：免费离线
# 或: openai, gemini, voyage, cohere

# 功能开关
AGENTMEMORY_AUTO_COMPRESS=false  # 默认关闭，节省费用
AGENTMEMORY_INJECT_CONTEXT=false  # 默认关闭，控制 token 消耗
GRAPH_EXTRACTION_ENABLED=true     # 启用知识图谱

# 多代理
AGENT_ID=my-agent
AGENTMEMORY_AGENT_SCOPE=isolated  # 隔离模式
```

### A.2 端口说明

| 端口 | 用途 |
|------|------|
| 3111 | REST API |
| 3112 | WebSocket 流 |
| 3113 | 可视化查看器 |

### A.3 API 基础路径

```
http://localhost:3111/agentmemory/
```

---

## 学习路径总结

```
┌─────────────────────────────────────────────────────────────┐
│  第一周：基础                                        ████░░ │
├─────────────────────────────────────────────────────────────┤
│  □ 完成环境配置                                          │
│  □ 运行 demo 体验功能                                      │
│  □ 理解核心数据结构                                        │
│  □ 阅读主入口代码                                          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  第二周：核心功能                                    ████░░ │
├─────────────────────────────────────────────────────────────┤
│  □ 深入理解观察捕获流程                                    │
│  □ 学习搜索系统原理                                        │
│  □ 掌握记忆生命周期                                        │
│  □ 运行和修改现有测试                                       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  第三周：高级特性                                    ██░░░░ │
├─────────────────────────────────────────────────────────────┤
│  □ 探索知识图谱实现                                        │
│  □ 学习 MCP 服务架构                                       │
│  □ 理解多代理集成                                          │
│  □ 实践调试和扩展                                          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  第四周：贡献代码                                    █░░░░░ │
├─────────────────────────────────────────────────────────────┤
│  □ 选择一个 issue 开始                                      │
│  □ 编写测试用例                                            │
│  □ 提交第一个 PR                                            │
│  □ 参与代码审查                                            │
└─────────────────────────────────────────────────────────────┘
```

---

祝你学习愉快！如果遇到问题，可以：
1. 查看 [issues](https://github.com/aifec001/localmemory/issues)
2. 阅读项目 Wiki
3. 运行 `npx @agentmemory/agentmemory doctor` 获取诊断信息
