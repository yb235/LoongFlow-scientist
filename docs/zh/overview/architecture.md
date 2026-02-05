# 架构详解

本页面为首次使用的用户、贡献者和开发者提供了 LoongFlow 内部架构的全面详细指南。帮助理解**代码如何组织**、**组件之间如何关联**以及**数据如何流转**。

---

## 仓库结构

```
LoongFlow/
├── src/loongflow/               # 核心框架（可安装包）
│   ├── agentsdk/                # 底层构建模块
│   │   ├── logger/              # 结构化日志工具
│   │   ├── memory/              # 记忆系统（进化 + 等级）
│   │   ├── message/             # 消息与元素抽象
│   │   ├── models/              # LLM 模型封装
│   │   ├── token/               # Token 计数
│   │   └── tools/               # 工具框架（BaseTool, Toolkit）
│   └── framework/               # 高层智能体框架
│       ├── base/                # AgentBase 抽象类
│       ├── pes/                 # PESAgent（规划-执行-总结）
│       ├── react/               # ReActAgent（推理-行动-观察）
│       └── claude_code/         # Claude Code Agent 集成
├── agents/                      # 领域特定智能体实现
│   ├── general_agent/           # 通用进化智能体
│   ├── math_agent/              # 数学/算法优化智能体
│   └── ml_agent/                # 机器学习竞赛智能体
├── tests/                       # 单元和集成测试
├── docs/                        # 文档（本站点）
├── run_general.sh               # 通用智能体启动脚本
├── run_math.sh                  # 数学智能体启动脚本
├── run_ml.sh                    # ML 智能体启动脚本
└── pyproject.toml               # Python 项目配置
```

---

## 分层架构

LoongFlow 遵循**三层架构**：

```
┌─────────────────────────────────────────────────────────────────┐
│                     智能体实现层                                  │
│         (general_agent, math_agent, ml_agent)                   │
│   领域特定的规划器、执行器、总结器、评估器                          │
├─────────────────────────────────────────────────────────────────┤
│                     框架层                                       │
│              (PESAgent, ReActAgent, AgentBase)                   │
│   编排、生命周期、并发、Worker 注册                                │
├─────────────────────────────────────────────────────────────────┤
│                     Agent SDK                                   │
│      (Message, Tools, Models, Memory, Logger, Token)            │
│   所有智能体共享的基础构建模块                                     │
└─────────────────────────────────────────────────────────────────┘
```

### 第一层：Agent SDK (`src/loongflow/agentsdk/`)

SDK 提供所有智能体使用的**基础原语**：

| 模块 | 用途 |
|------|------|
| **message/** | 统一的 `Message` 和 `Element` 抽象，用于所有数据交换 |
| **tools/** | `BaseTool`、`FunctionTool`、`Toolkit` — 工具注册与执行框架 |
| **models/** | `BaseLLMModel`、`LiteLLMModel` — LLM 提供商抽象，支持 OpenAI、Gemini、DeepSeek 等 |
| **memory/** | 两种记忆系统：`EvolveMemory`（用于 PES 智能体）和 `GradeMemory`（用于 ReAct 智能体） |
| **logger/** | 带 trace ID 的结构化日志 |
| **token/** | 用于成本追踪的 Token 计数 |

### 第二层：框架 (`src/loongflow/framework/`)

框架层提供**智能体编排模式**：

| 模块 | 用途 |
|------|------|
| **base/** | `AgentBase` — 带钩子、生命周期和错误处理的异步智能体抽象基类 |
| **pes/** | `PESAgent` — 核心规划-执行-总结进化智能体，支持并发循环 |
| **react/** | `ReActAgent` — 推理-行动-观察循环智能体，用于交互式工具使用任务 |
| **claude_code/** | `ClaudeCodeAgent` — 与 Anthropic Claude 的代码生成集成 |

### 第三层：智能体实现 (`agents/`)

实现框架抽象接口的具体领域智能体：

| 智能体 | 使用场景 | 框架 |
|--------|---------|------|
| **general_agent/** | 通用问题求解 | PESAgent + Claude Code |
| **math_agent/** | 数学优化、算法设计 | PESAgent + LiteLLM |
| **ml_agent/** | ML 竞赛（Kaggle）、AutoML | PESAgent + EvoCoder |

---

## 类继承关系

```
                    AgentBase (ABC)
                    ├── PESAgent
                    └── ReactAgentBase
                        └── ReActAgent

                    Worker (ABC)
                    ├── Planner workers
                    ├── Executor workers
                    └── Summary workers

                    BaseLLMModel (ABC)
                    └── LiteLLMModel

                    EvolveMemory (ABC)
                    ├── InMemoryEvolveMemory
                    ├── RedisEvolveMemory
                    └── BoltzmannMemory

                    BaseTool (ABC)
                    ├── FunctionTool
                    ├── ShellTool / ReadTool / WriteTool
                    ├── ExecuteCodeTool / LsTool
                    ├── TodoReadTool / TodoWriteTool
                    └── AgentTool
```

---

## PES 智能体详解

**PESAgent** 是 LoongFlow 的核心。它编排一个进化循环，每次迭代遵循三个阶段：

### 阶段 1：规划（Plan）

**Planner** Worker 接收当前上下文（任务描述、历史方案数据库）并产生策略计划：

- 从进化数据库中采样一个父方案
- 分析父方案的优缺点
- 从记忆中检索相关的历史经验
- 生成详细的改进蓝图

### 阶段 2：执行（Execute）

**Executor** Worker 接收计划并产生新的候选方案：

- 解析规划器输出消息中的计划
- 克隆父方案（写时复制模式）
- 基于计划生成或修改代码
- 使用评估器评估新方案
- 返回方案路径和评估结果

### 阶段 3：总结（Summarize）

**Summary** Worker 反思执行结果：

- 收集证据：计划、方案、评估、父方案对比
- 评估结果：改进（IMPROVEMENT）、退化（REGRESSION）或停滞（STALE）
- 提取可复用的见解
- 将新方案以计算的权重记录到数据库中

---

## ReAct 智能体详解

**ReActAgent** 实现推理-行动-观察循环，用于交互式任务：

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  推理    │────▶│  行动    │────▶│  终止检查 │────▶│  观察    │
│ (LLM)   │     │ (工具)   │     │ (检查)   │     │ (更新)   │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
     ▲                                                    │
     └────────────────────────────────────────────────────┘
                    （循环最多 max_steps 次）
```

---

## 记忆系统

### 进化记忆（用于 PES 智能体）

- **多岛屿**：方案组织到"岛屿"中以保持多样性
- **MAP-Elites**：基于特征的分桶以维护多样化的方案生态
- **自适应 Boltzmann 选择**：平衡探索与利用
- **检查点**：持久化快照用于容错

### 等级记忆（用于 ReAct 智能体）

| 层级 | 描述 | 用途 |
|------|------|------|
| **STM**（短期） | 最近的消息 | 即时上下文 |
| **MTM**（中期） | 压缩的摘要 | 对话连续性 |
| **LTM**（长期） | 知识库 | 跨会话学习 |

---

更多详细内容，请参考 [英文版架构文档](../../en/overview/architecture.md)。
