# AgentMemory 项目功能与实现逻辑分析

## 1. 项目概述

**AgentMemory** 是一个为 AI 编程助手（如 Claude Code、GitHub Copilot CLI、Cursor 等）提供持久化记忆的系统。它通过捕获、压缩、索引和检索 AI 会话中的操作，让 AI 能够记住跨会话的项目上下文和之前的决策，从而避免重复解释，提高开发效率。

**核心特性**：
- 多代理兼容（支持 30+ AI 编程助手）
- 混合搜索（BM25 + 向量 + 知识图谱）
- 4层记忆整合系统（工作→事件→语义→程序）
- 实时可视化查看器
- 完整的 MCP 接口支持
- 团队协作记忆共享
- 隐私保护机制

## 2. 系统架构

### 2.1 技术栈

| 组件 | 技术选型 |
|------|---------|
| 运行时 | Node.js (TypeScript) |
| 数据引擎 | iii-engine (内置 SQLite) |
| 搜索索引 | 自定义 BM25 + 向量索引 |
| LLM 提供者 | Anthropic、Gemini、OpenAI、OpenRouter、MiniMax 等 |
| 嵌入模型 | 本地 Transformers (all-MiniLM-L6-v2) 或云端 API |
| 接口方式 | REST API、WebSocket、MCP |
| 可视化 | 内置 Web 服务 (端口 3113) |

### 2.2 核心模块架构

```
src/
├── index.ts              # 主入口，worker 启动
├── config.ts             # 配置加载与管理
├── types.ts              # TypeScript 类型定义
├── cli.ts                # 命令行接口
├── functions/            # 核心功能函数
├── hooks/                # 代理 hook 处理
├── mcp/                  # MCP 服务实现
├── providers/            # LLM/嵌入提供者
├── state/                # 状态管理与索引
├── triggers/             # API/事件触发器
├── viewer/               # 可视化服务器
└── health/               # 健康监控
```

## 3. 数据模型

### 3.1 核心数据类型

#### Session（会话）
```typescript
{
  id: string;
  project: string;
  cwd: string;
  startedAt: string;
  endedAt?: string;
  status: "active" | "completed" | "abandoned";
  observationCount: number;
  model?: string;
  tags?: string[];
  firstPrompt?: string;
  summary?: string;
}
```

#### Observation（观察）
分为原始观察和压缩观察：

**RawObservation**：原始钩子触发数据
**CompressedObservation**：结构化、压缩后的可搜索数据

```typescript
{
  id: string;
  sessionId: string;
  timestamp: string;
  type: ObservationType;  // file_read/write/edit, command_run, search 等
  title: string;
  facts: string[];
  narrative: string;
  concepts: string[];
  files: string[];
  importance: number;
}
```

#### Memory（记忆）
长期保存的重要知识单元：
```typescript
{
  id: string;
  type: "pattern" | "preference" | "architecture" | "bug" | "workflow" | "fact";
  title: string;
  content: string;
  concepts: string[];
  files: string[];
  sessionIds: string[];
  strength: number;  // 重要度，用于检索排序
  version: number;   // 版本化管理
  isLatest: boolean;
}
```

### 3.2 4层记忆整合系统

受人类大脑记忆处理方式启发：

1. **Working（工作记忆）**：原始观察数据
2. **Episodic（事件记忆）**：会话摘要和时间线
3. **Semantic（语义记忆）**：提取的事实和模式
4. **Procedural（程序记忆）**：工作流程和决策模式

### 3.3 知识图谱

节点类型：`file`, `function`, `concept`, `error`, `decision`, `pattern`, `library`, `person` 等

边类型：`uses`, `imports`, `modifies`, `causes`, `fixes`, `depends_on`, `related_to` 等

## 4. 核心功能实现

### 4.1 观察捕获流程

```
PostToolUse 钩子触发
    ↓
SHA-256 去重 (5分钟窗口)
    ↓
隐私过滤器 (移除密钥、API 密钥)
    ↓
存储原始观察 (RawObservation)
    ↓
LLM 压缩 → 结构化事实+概念+叙述 (可选)
    ↓
向量化 (可选)
    ↓
BM25 + 向量索引
```

关键文件：
- `src/functions/observe.ts`：观察捕获函数
- `src/functions/privacy.ts`：隐私保护过滤器
- `src/functions/compress.ts`：LLM 压缩函数

### 4.2 混合搜索系统

#### 三通道检索：

1. **BM25 搜索**：基于词频和逆文档频率的关键词匹配
2. **向量搜索**：基于嵌入相似度的语义匹配
3. **图谱搜索**：知识图谱遍历，基于实体关联

结果融合采用**互惠排序融合 (RRF)**：
```typescript
RRF(score) = 1 / (k + rank)
// k=60 (默认)
```

关键实现：
- `src/state/hybrid-search.ts`：三通道融合算法
- `src/state/search-index.ts`：BM25 索引
- `src/state/vector-index.ts`：向量索引

### 4.3 记忆生命周期管理

#### 自动遗忘机制：
- 时间衰减（Ebbinghaus 遗忘曲线）
- 访问频率强化
- 重要度评估
- 矛盾检测与解决

#### 整合管道：
- 会话结束时自动整合
- 提取知识图谱
- 总结和模式检测
- 长期记忆固化

关键文件：
- `src/functions/consolidate.ts`
- `src/functions/auto-forget.ts`
- `src/functions/retention.ts`

### 4.4 多代理集成

支持的代理类型：
- Claude Code (原生插件 + hooks)
- Codex CLI (原生插件 + 6 hooks)
- GitHub Copilot CLI
- Cursor
- Gemini CLI
- OpenClaw
- Hermes Agent
- Warp
- 以及 30+ 其他代理

集成方式：
1. **Hooks**：会话开始/结束、工具使用前后等钩子
2. **MCP**：Model Context Protocol 标准接口
3. **REST API**：HTTP 接口调用
4. **Skills**：技能文件集成

关键实现：
- `src/cli/connect/` 目录：各代理的连接适配器
- `plugin/` 目录：Claude Code 等的原生插件

## 5. 工作流程

### 5.1 新会话启动流程

```
SessionStart 钩子
    ↓
加载项目 Profile (top concepts, files, patterns)
    ↓
执行混合搜索 (基于项目上下文)
    ↓
生成 Context Block (token 预算限制)
    ↓
注入对话 (可选，默认关闭)
    ↓
会话开始观察记录
```

### 5.2 工具使用观察流程

```
PreToolUse → 预观察捕获
    ↓
工具执行
    ↓
PostToolUse → 后观察捕获 + 压缩 + 索引
    ↓
PostToolUseFailure → 错误观察 (如果失败)
    ↓
PreCompact → 预压缩钩子 (Claude Code 特有)
```

### 5.3 会话结束流程

```
Stop 或 SessionEnd 钩子
    ↓
生成会话摘要
    ↓
知识图谱提取 (如果启用)
    ↓
Slot 反射 (如果启用)
    ↓
整合管道运行 (如果启用)
    ↓
索引持久化
    ↓
会话状态更新为 completed
```

## 6. 关键设计决策

### 6.1 默认关闭 LLM 压缩

从 0.8.8 版本开始，`AGENTMEMORY_AUTO_COMPRESS` 默认关闭，因为：
- 避免 API 费用过高
- 减少延迟
- 使用零 LLM 合成压缩路径
- 用户可以选择性启用

### 6.2 默认关闭上下文注入

从 0.8.10 版本开始，`AGENTMEMORY_INJECT_CONTEXT` 默认关闭，避免：
- Claude Pro token 消耗过快
- 对话被大量上下文干扰
- 用户通过 MCP/技能主动调用回忆

### 6.3 多代理隔离

支持 `AGENT_ID` 和 `AGENTMEMORY_AGENT_SCOPE`：
- `shared` (默认)：共享但标记来源
- `isolated`：完全隔离

### 6.4 索引持久化

- BM25 和向量索引持久化到磁盘
- 启动时快速加载
- 维度变化时自动重建或丢弃

## 7. 性能与优化

### 7.1 基准测试结果

| 指标 | 数值 |
|------|------|
| 检索召回率@5 | 95.2% |
| 检索精度@5 | 57.8% |
| 延迟 | 14ms (平均) |
| Token 节省 | 92% |
| 每日成本 | ~$10 (默认配置) |

### 7.2 扩展能力

- 支持 10万+ 观察记录
- 索引增量更新
- 自动垃圾回收
- 磁盘空间管理

## 8. 部署与配置

### 8.1 环境变量

关键配置项：
```bash
# 服务端口
III_REST_PORT=3111
III_STREAMS_PORT=3112

# LLM 提供者 (选择一个)
ANTHROPIC_API_KEY=...
GEMINI_API_KEY=...
OPENROUTER_API_KEY=...
MINIMAX_API_KEY=...
OPENAI_API_KEY=...

# 嵌入提供者
EMBEDDING_PROVIDER=local|gemini|openai|voyage|cohere

# 功能开关
GRAPH_EXTRACTION_ENABLED=false
CONSOLIDATION_ENABLED=true
AGENTMEMORY_AUTO_COMPRESS=false
AGENTMEMORY_INJECT_CONTEXT=false

# 多代理
AGENT_ID=...
AGENTMEMORY_AGENT_SCOPE=shared|isolated
```

### 8.2 部署方案

- **本地开发**：直接运行 `npx @agentmemory/agentmemory`
- **Docker**：提供 Docker Compose 配置
- **托管服务**：支持 Fly.io、Railway、Render、Coolify 等一键部署

## 9. 测试覆盖

项目包含完整的测试套件：

- 单元测试：950+ 个测试用例
- 集成测试：跨模块功能验证
- 基准测试：检索质量和性能评估
- 评估框架：LongMemEval 和自定义 corpus

关键测试文件：`test/` 目录下所有 `.test.ts` 文件

## 10. 未来路线图

根据 `ROADMAP.md`，主要方向：
- 更强大的知识图谱推理
- 多模态记忆（图像、音频）
- 跨项目知识迁移
- 团队协作增强
- 插件生态系统

## 总结

AgentMemory 是一个设计完善的 AI 记忆系统，通过结合传统信息检索（BM25）、现代向量搜索和知识图谱技术，为 AI 编程助手提供了持久化的记忆能力。其 4层记忆整合模型模拟了人类的记忆处理方式，而多代理兼容性确保了广泛的适用性。默认保守的配置策略（关闭 LLM 压缩和上下文注入）平衡了功能和成本，是一个生产就绪的解决方案。
