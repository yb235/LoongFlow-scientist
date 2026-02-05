# 入门指南

欢迎使用 LoongFlow！本指南面向**完全的初学者** — 从未使用过该框架的开发者。

---

## 什么是 LoongFlow？

LoongFlow 是一个 **AI 智能体框架**，实现自动化的进化式问题求解：

1. **接收问题**（数学优化、ML 竞赛、通用编码）
2. **生成初始方案**
3. **通过结构化的规划→执行→总结循环迭代改进**方案
4. **从每次尝试中学习**以做出更好的决策

---

## 前提条件

- **Python 3.12 或更高版本**
- **LLM API 密钥**（OpenAI、Google Gemini、DeepSeek 或任何 OpenAI 兼容提供商）
- **`uv` 包管理器**（推荐）或 `conda`

---

## 安装

```bash
# 克隆仓库
git clone https://github.com/baidu-baige/LoongFlow.git
cd LoongFlow

# 使用 uv（推荐）
uv venv .venv --python 3.12
source .venv/bin/activate
uv pip install -e .

# 或使用 conda
conda create -n loongflow python=3.12
conda activate loongflow
pip install -e .
```

---

## 核心概念

### PES 范式

| 阶段 | 做什么 | 类比 |
|------|-------|------|
| **规划（Plan）** | 分析当前最优方案，识别弱点，设计改进策略 | 科学家撰写研究提案 |
| **执行（Execute）** | 生成实现计划的新代码，运行并测量结果 | 在实验室进行实验 |
| **总结（Summarize）** | 将新结果与之前最优比较，提取经验教训 | 撰写研究论文 |

### 智能体类型

| 智能体 | 适用场景 |
|--------|---------|
| **数学智能体** | 算法优化、数学问题 |
| **ML 智能体** | Kaggle 竞赛、AutoML |
| **通用智能体** | 任何有明确评估指标的问题 |

---

## 快速运行

```bash
# 安装任务依赖
uv pip install -r ./agents/math_agent/examples/packing_circle_in_unit_square/requirements.txt

# 配置 LLM（编辑 task_config.yaml）
# 运行任务
./run_math.sh packing_circle_in_unit_square --background

# 监控进度
tail -f ./agents/math_agent/examples/packing_circle_in_unit_square/run.log

# 停止任务
./run_math.sh stop packing_circle_in_unit_square
```

---

## 术语表

| 术语 | 定义 |
|------|------|
| **PES** | 规划-执行-总结 — 核心思维范式 |
| **Worker** | 处理一个 PES 阶段的可插拔组件 |
| **进化循环** | 一次完整的 规划→执行→总结 迭代 |
| **岛屿** | 半独立进化的方案子群体 |
| **Boltzmann 选择** | 选择父方案的概率方法 |
| **EvoCoder** | ML 智能体的代码生成系统 |

---

更多详细内容，请参考 [英文版入门指南](../../en/overview/getting-started-guide.md)。
