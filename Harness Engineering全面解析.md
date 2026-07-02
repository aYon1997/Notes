# Harness Engineering 全面解析

> 从 Prompt Engineering 到 Context Engineering，再到如今的 Harness Engineering——Agent 工程的演进正在重塑开发者的角色。

---

## 一、什么是 Harness Engineering？

### 1.1 定义

**一句话定义**：Harness Engineering 是一种工程实践，通过为 AI Agent 设计、配置和优化其运行环境（"Harness"），使 Agent 能够可靠、安全、高质量地完成复杂任务。

**核心公式**：

```
Coding Agent = AI Model(s) + Harness
```

"Harness" 是 Agent 的"runtime"或"外设"——模型用来与环境交互的一切配置层，包括：

| 配置层 | 说明 |
|--------|------|
| `CLAUDE.md` / `AGENTS.md` | 系统级指令注入 |
| MCP Servers | 外部工具能力扩展 |
| Skills | 可复用的知识/工具模块（渐进式披露） |
| Sub-Agents | 上下文隔离与任务委派 |
| Hooks | 自动化控制流（验证、通知、审批） |
| Back-Pressure | 反压机制（验证、测试、类型检查） |

---

## 二、Agent Engineering 演进

### 2.1 三个阶段的演进

| 阶段 | 时间 | 核心技能 | 核心问题 |
|------|------|----------|----------|
| Prompt Engineering | 2022-2024 | 写好提示词 | "如何跟模型说话" |
| Context Engineering | 2025 | 管理上下文窗口 | "给模型看什么" |
| **Harness Engineering** | **2026+** | **设计 Agent 运行环境** | **"让模型在什么条件下工作"** |

### 2.2 爆发的根本原因

- **模型不再是瓶颈，配置才是**：团队反复发现 Agent 出错，本能反应是"等更好的模型"（GPT-6 会修好的），但实践表明大部分失败是 Harness 配置问题
- **生产级部署的需求**：从 Demo/聊天机器人走向严肃的多日长任务（OpenAI 报告单个 Codex 运行可连续工作 6 小时以上），没有 Harness 就无法保证可靠性
- **非确定性系统的本质挑战**：更强的模型会被分配更难的任务，在新的难度上继续产生意想不到的失败模式——这不是等待更好模型能解决的问题
- **Agent-First 开发成为现实**：OpenAI 用 3-7 人团队、5 个月、~100 万行代码、0 行手写代码交付了内部产品，证明了 Agent-First 工程的可行性

---

## 三、Harness Engineering 的核心原则

### 原则 1：约束优于微操

> **Enforce Invariants, Not Micromanage Implementations**

**做法**：
- 给出严格的边界约束，但在边界内允许 Agent 自由发挥
- 例如：要求在数据边界做解析（parse data shapes at the boundary），但不规定用什么库（模型自己选了 Zod）
- 使用自定义 Linter 和结构化测试来机械地执行规则

**架构示例（OpenAI）**：每个业务域分层为 `Types → Config → Repo → Service → Runtime → UI`，依赖方向严格单向。跨域关注点（auth、telemetry、feature flags）通过单一显式接口 Providers 进入。任何违规都通过自定义 Linter 强制执行。

---

### 原则 2：渐进式披露

> **Progressive Disclosure**

**ETH Zurich 的实证研究**：

- 测试了 138 个 Agent 文件，发现 LLM 自动生成的 `AGENTS.md` 降低了性能，同时成本增加 **20%+**
- 人类编写的也只提升了约 **4%**
- Agent 花费 **14-22%** 的推理 token 处理指令文件，步骤更多、工具调用更多，但成功率没提升
- 代码库概览和目录列表完全没用——Agent 自己就能发现仓库结构

**最佳实践**：

- `CLAUDE.md` / `AGENTS.md` 保持 **60-100 行**以内
- 作为"目录"而非"百科全书"
- 知识库放在结构化的 `docs/` 目录，`CLAUDE.md` 只放指针
- 用 Linter 验证知识库的时效性和交叉引用
- 专门的"doc-gardening" Agent 定期扫描过时文档并开 PR 修复

---

### 原则 3：上下文是稀缺资源

> **Context is a Scarce Resource**

Chroma 的研究表明：模型在更长的上下文下表现更差，即使简单任务也是如此。当上下文中存在低语义相关性的信息时，退化更严重。

**Harness 中的上下文管理策略**：

#### Sub-Agents 充当"上下文防火墙"

- 每个 Sub-Agent 获得独立的、小的、高相关性的上下文窗口
- 只有最终精炼的结果回流到父会话
- 中间的工具调用、文件读取等不会污染主会话

#### 分层模型策略

- 父会话使用昂贵模型（Opus）处理规划/编排等重思考任务
- Sub-Agent 使用便宜模型（Sonnet/Haiku）处理代码搜索等轻量任务
- 没必要在代码库 grep 上烧 Opus token

#### MCP Server 不宜过多

- 每个 MCP Server 的工具描述都注入系统 prompt
- 过多工具会更快进入"dumb zone"
- 能用 CLI 的就不要用 MCP（如 GitHub、Docker）——模型训练数据中已包含这些 CLI 用法，且能与 grep/jq 组合使用

#### 对长上下文模型保持警惕

- 更大的上下文窗口不等于更好的表现——只是让"haystack"更大了
- 如果觉得需要更长上下文，可能只是需要更好的上下文隔离

---

### 原则 4：分离生成与评估

> **Separate Generation from Evaluation**

**实践要点**：

- Agent 生成代码，独立的评估器审查代码
- 验证机制必须上下文高效：成功时静默，只把错误暴露给 Agent

> 💡 **经验教训**：早期 HumanLayer 让 Agent 每次跑完整测试套件，4000 行通过的测试淹没上下文窗口，Agent 反而开始幻觉——现在改为吞掉输出，只暴露错误。

**Back-Pressure（反压）机制栈**：

```
TypeCheck → Build → 单元测试 → 覆盖率报告 → UI 交互测试
```

---

### 原则 5：持续债务清理

> **Garbage Collection for Code**

**OpenAI 的经验**：

- 初期每周五花 20% 时间清理"AI slop"（AI 生成的冗余代码），不可持续
- 后来编码 "Golden Principles" 到仓库中，设置定期清理 Agent 扫描偏差并开 PR

**示例原则**：
1. 优先使用共享工具包而非手工实现的辅助函数
2. 不"YOLO式"探测数据——必须验证边界或使用有类型 SDK

---

## 四、实战架构参考

### 4.1 OpenAI 的 Agent-First 产品实践

**规模**：3-7 人工程团队，5 个月，~100 万行代码，~1500 个 PR，平均每人每天 3.5 个 PR，**0 行手动编写代码**。

#### 仓库结构

```
Repository Structure:
├── AGENTS.md              (约100行，目录/地图)
├── docs/                  (结构化知识库)
│   ├── design/            (设计文档，含验证状态)
│   ├── architecture/      (架构地图和分层)
│   ├── quality/           (质量评分，按域/层追踪)
│   └── plans/             (执行计划，版本化)
├── .agents/               (Agent配置)
├── src/                   (业务域，严格分层)
└── tools/                 (自定义Linter、CI)
```

#### Agent 的完整工作流（单 Prompt 驱动）

1. 验证代码库当前状态
2. 复现报告的 Bug
3. 录制失败视频（通过 Chrome DevTools Protocol）
4. 实现修复
5. 通过驱动应用验证修复
6. 录制修复后视频
7. 开 PR
8. 回复 Agent 和人类审查反馈
9. 检测并修复构建失败
10. 仅在需要判断时升级给人类
11. 合并变更

#### 关键创新

- 让 App 可以按 git worktree 启动，Codex 每个变更启动一个独立实例
- 将 Chrome DevTools Protocol 接入 Agent 运行时，创建 DOM 快照/截图/导航的 Skills
- 接入本地可观测性栈（LogQL/PromQL），Agent 可以查询日志和指标

> 例如："确保服务启动在 800ms 以内" 或 "这四个关键用户旅程中没有一个 span 超过两秒"

---

### 4.2 HumanLayer 的 Harness 配置实战

| ✅ 成功的做法 | ❌ 失败的做法 |
|--------------|--------------|
| 从简单开始，只在 Agent 实际失败时才添加配置 | 试图在遇到实际失败之前设计"完美"的 Harness |
| 设计→测试→迭代，果断扔掉没用的东西（扔掉的 hook 比实际使用的多得多） | 安装几十个 Skills 和 MCP Server "以防万一" |
| 通过仓库级配置在团队间分发经过验证的配置 | 每次会话后跑完整测试套件（5分钟+），应该跑子集 |
| 为 Agent 提供能力（如 Linear CLI），然后精简暴露给模型的接口 | 微观管理 Sub-Agent 能访问哪些工具——导致工具反复切换，结果更差 |

---

### 4.3 典型 Hook 示例

```bash
#!/bin/bash
# Agent 停止时自动运行格式化和类型检查
# 成功→静默；失败→只暴露错误给 Agent，exit code 2 让 Harness 重新激活 Agent
cd "$CLAUDE_PROJECT_DIR"

# 预构建：生成类型和构建内部SDK包
PREBUILD_OUTPUT=$(bun run generate-cache-key && turbo run build --filter=@humanlayer/hld-sdk && bun install 2>&1)
if [ $? -ne 0 ]; then
  echo "prebuild failed:" >&2
  echo "$PREBUILD_OUTPUT" >&2
  exit 2
fi

# biome 格式化 + typecheck 并行执行
# biome --write quirks: 第一次修改后 exit 1，第二次 exit 0
OUTPUT=$(bun run --parallel \
  "biome check . --write --unsafe || biome check . --write --unsafe" \
  "turbo run typecheck" 2>&1)
if [ $? -ne 0 ]; then
  echo "$OUTPUT" >&2
  exit 2
fi
```

---

### 4.4 Skill 的渐进式披露架构

```
example-skill/
├── SKILL.md              (主入口，Agent 激活时加载)
├── response_template.md  (按需读取的模板)
├── CLIs/
│   ├── linear-cli        (打包的命令行工具)
│   └── tunnel-cli
└── docs/
    ├── advanced.md       (高级用法，按需披露)
    └── troubleshooting.md
```

> 💡 **技巧**：`SKILL.md` 中可以指导 Agent："当你需要做 X 时，去读 `advanced.md`"——实现知识的多层渐进披露。

---

## 五、对我们团队的启示

### 5.1 可落地的方向

| 行动 | 优先级 | 预期收益 |
|------|--------|----------|
| 审视 `AGENTS.md` / `CLAUDE.md`，控制在 60-100 行 | **P0** | 减少 14-22% 的推理 token 浪费 |
| 建立 Back-Pressure Hook（Agent 停止时自动 TypeCheck + Lint） | **P0** | Agent 自修复简单错误，减少人工干预 |
| Sub-Agent 化长上下文任务（代码搜索、信息追踪、研究） | P1 | 主会话保持"聪明区"更久 |
| 审计 MCP Server，关掉不常用的，用 CLI 替代高频工具 | P1 | 减少系统 prompt 膨胀 |
| 建立 Eval-Driven Development（评估驱动开发）流程 | P2 | Agent 输出质量可量化追踪 |
| 渐进式建设：遇到失败再添加配置，不要预设完美方案 | 原则 | 避免过度工程化 |

---

### 5.2 战略性思考

| 维度 | 当前状态 | 目标 |
|------|----------|------|
| 开发者角色 | "写代码的人" | "设计 Harness 的人" |
| 质量保证 | 人工 Code Review | Agent-to-Agent Review + 人类抽检 |
| 上下文管理 | 单一大上下文窗口 | 多层 Sub-Agent 隔离 |
| 知识管理 | 散落在 Slack/文档中 | 仓库内结构化、版本化、可 Agent 发现 |
| 债务管理 | 周期性人工清理 | 持续自动化 Agent 清理（GC for Code） |
| 可观测性 | 人类读 Dashboard | Agent 直接查询 LogQL/PromQL |

---

### 5.3 风险与注意事项

- ⚠️ **过度工程化**：不要在遇到实际失败之前设计"完美"的 Harness——HumanLayer 明确表示这样做是失败的
- 🔒 **安全意识**：MCP Server 和 Skill 的引入需要安全审查，不要盲目安装
- 🔄 **模型依赖**：不要过度依赖特定模型的 Harness 耦合特性，保持可移植性
- ✂️ **迭代优先**：扔掉没用的配置比保留它们更重要

---

## 参考来源

### 官方与机构

- OpenAI 官方: *Harness Engineering: Leveraging Codex in an Agent-First World*
- HumanLayer: *Skill Issue - Harness Engineering for Coding Agents*
- LangChain: *State of Agent Engineering*
- LangChain Blog: *Improving Deep Agents with Harness Engineering*

### 媒体与博客

- Aakash Gupta (Medium): *2025 Was Agents. 2026 Is Agent Harnesses. Here's Why That Changes Everything*
- Philschmid: *The Importance of Agent Harness in 2026*
- NXCode: *Complete Guide to Harness Engineering for AI Agent & Codex 2026*
- Epsilla: *Why Harness Engineering Replaced Prompting in 2026*
- Medium: *AI Harness Engineering - The Layer Above Context Engineering*
- Medium: *The Rise of Agent Harness Engineering - Navigating Long-Term AI Autonomy*
- LinkedIn: *Vibe Coding is Dead: The Rise of AI Harness Engineering*
- Harness Engineering AI Blog (Daily News)

### 社区与视频

- 知乎: *Harness时代：2026年AI Agent工程实践与开发者生存法则*
- GitHub: *Learn Claude Code -- Harness Engineering for Real Agents*
- MCPMarket: *Eval Harness Claude Code Skill*
- YouTube: *Harness Engineering - The Skill That Will Define 2026 for Solo Devs*

---

> 📌 **核心洞察**：2026 年，开发者的核心竞争力不再是写代码的速度，而是设计和优化 Agent 运行环境的能力。Harness Engineering 不是一时的潮流，而是 Agent 从玩具走向生产工具的必然结果。
