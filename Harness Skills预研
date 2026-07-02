# 1. 概述

Harness Skills 是一套为 AI Agent 构建的开发基础设施技能系统，核心理念是："Agent Harness 是操作系统，LLM 只是 CPU"。该系统旨在让 AI Agent 能够在代码库中独立、可靠地执行开发任务，而非仅作为人类开发的辅助工具。

## 1.1 核心哲学

"Intelligence without infrastructure is just a demo."

地图而非手册：什么都重要等于什么都不重要
嵌入而非外挂：领域知识、架构规约放在代码仓库，不放外置知识库
机械验证而非人工检查：代码规范写到RULES可能会表达不清晰，Agent未必自觉遵守，变成Lint规则跑在验证阶段就能明确阻塞
迭代自愈而非等待介入：写完代码之后自动验证，失败了自动分析修复，1-3轮收敛，反复失败之后才升级给人类




## 1.2 技能组成
技能名称	职责	调用时机
harness-creator	创建 AI Agent 基础设施	项目初始化或基础设施缺失时
harness-executor	执行开发任务并强制验证	实现功能、修复 Bug、重构代码等任务

两者的关系可概括为：harness-creator 搭建舞台，harness-executor 在舞台上表演。

# 2. harness-creator
## 2.1 功能定义

harness-creator 用于为代码库设计和创建 AI Agent 基础设施，使 Agent 能够可靠地理解和工作于该代码库。其创建的组件包括：

组件	路径	说明
Agent 导航文件	AGENTS.md	

80-120 行的导航地图

，链接到详细文档


文档架构	docs/	ARCHITECTURE.md、DEVELOPMENT.md、design-docs/
Linter 脚本	scripts/lint-*	带有可操作错误信息的自动化检查脚本
Harness 配置	harness/	environment.json、eval 任务、memory 结构
CI 集成	Makefile、.github/workflows/	构建与验证自动化

重要约束：该技能仅创建基础设施文件，不编写业务或应用代码。

## 2.2 工作原理

采用 5 阶段统一工作流，通过检测当前状态与目标状态的差异来决定操作：

Phase 1: 快速检测 + 意图确认
         ↓
Phase 2: 并行分析（代码架构、Harness 状态、环境依赖）
         ↓
Phase 3: 差异合成（计算需要创建/更新的内容）
         ↓
Phase 4: 并行创建/更新（文档、Linters、配置）
         ↓
Phase 5: 验证 + 交接
#### 2.2.1 项目状态分类

基于检测结果，将项目划分为四种状态并采取相应行动：

状态	判断条件	处理策略
Empty（空项目）	文件数 < 5 且无代码文件	引导用户选择技术栈与项目类型
Code Only（仅代码）	存在代码但无 AGENTS.md	完整分析 + 完整基础设施创建
Partial Harness（部分基础设施）	有 AGENTS.md 但组件不全	差异分析 + 填补缺失组件
Full Harness（完整基础设施）	所有组件存在	审计现状 + 提供改进建议
### 2.2.2 并行分析阶段

Phase 2 通过并行子 Agent 加速分析过程：

子 Agent	分析内容	输出
Code Architecture Agent	导入依赖图、层级结构、接口类型、关键路径、循环依赖检测	harness/.analysis/architecture.json
Harness State Agent	AGENTS.md 状态、文档准确性、linter 质量、eval 覆盖度	harness/.analysis/audit.json
Environment Agent	数据库驱动、服务 SDK、环境变量、部署配置	harness/.analysis/environment.json
### 2.2.3 核心设计原则

AGENTS.md 是地图而非手册：控制在 80-120 行，通过链接指向详细文档，避免信息膨胀

Linter 错误必须 Agent 可操作：错误信息包含 WHAT（问题）、WHY（原因）、HOW（修复方案）

基于实际代码生成文档：每个声明必须引用具体的 file:line，杜绝占位符内容

Linter 错误信息设计示例：

✗ 错误示例（不可操作）:
  "Forbidden import in core/types/user.go"
​
✓ 正确示例（可操作）:
  "core/types/user.go:15 imports core/config (layer 0 → layer 2).
   Layer 0 packages must have NO internal dependencies.
   
   Fix options:
   1. Move config-dependent logic to a higher layer
   2. Pass the config value as a parameter
   3. Use dependency injection via an interface"
## 2.3 目录结构
.qoder/skills/harness-creator/
├── SKILL.md              # 技能定义与工作流程
├── agents/               # 子 Agent 提示词
│   ├── analyzer.md       # 代码架构分析
│   ├── auditor.md        # Harness 状态审计
│   ├── creator-config.md # 配置创建
│   ├── creator-docs.md   # 文档创建
│   └── creator-linters.md # Linter 创建
└── references/           # 参考模板与指南
    ├── greenfield-templates.md        # 空项目模板
    ├── documentation-templates.md     # 文档模板
    ├── linter-templates.md            # Linter 模板
    ├── environment-detection-guide.md # 环境检测指南
    ├── environment-config-guide.md    # 环境配置指南
    └── ...更多参考文件
# 3. harness-executor
## 3.1 功能定义

harness-executor 用于自主执行开发任务并进行强制验证。其核心能力包括：

能力	说明
自动 Bootstrap	检测基础设施缺失时自动调用 harness-creator
代码执行	实现功能、修复 Bug、重构代码
静态验证	构建、Lint、测试的自动化检查
功能验证	启动应用、发送真实 HTTP 请求、验证实际行为
状态管理	任务中断恢复、Checkpoint 机制
经验记录	Episodic Memory 系统积累经验教训

架构原则：Coordinator（协调器）管理状态，Subagent（子 Agent）执行代码。协调器通过 spawn 子 Agent 进行代码变更和验证，子 Agent 不调用状态管理脚本。

## 3.2 工作原理

采用 7 步强制执行流程，无例外、无跳过：

Step 1: SETUP     → Bootstrap → 检查中断任务 → 查询 Memory → 加载上下文
Step 2: PLAN      → 确定范围 → 初始化状态 → (多阶段: 计划文件 + 用户审批)
Step 3: EXECUTE   → Spawn Executor Subagent → 代码变更 → Checkpoint
Step 4: VALIDATE  → 静态验证（Build + Lint + Test）
Step 5: VERIFY    → Spawn Verifier Subagent → 功能验证（强制）
Step 6: RECORD    → Complete → Episodic Memory → AutoHarness
Step 7: PRESENT   → 结果摘要展示

关键约束：Step 4（静态验证）和 Step 5（功能验证）均为强制步骤。静态验证证明代码可编译，功能验证证明代码能工作。

### 3.2.1 各步骤详解
步骤	执行内容	失败处理
Setup	检测 AGENTS.md、查询中断任务、加载架构文档	缺少 AGENTS.md 则调用 harness-creator
Plan	确定修改/创建文件列表，初始化任务状态，多阶段任务需用户审批	用户拒绝则终止
Execute	Spawn Executor Subagent 执行代码变更，返回结构化 JSON 结果	失败重试最多 2 次，blocked 则上报用户
Validate	运行 build + lint + test 静态检查	失败则返回 Step 3
Verify	Spawn Verifier Subagent 启动应用，发送 HTTP 请求验证行为	失败则返回 Step 3，最多 2 次重试
Record	调用 task_state.py complete，移动计划文件，触发 AutoHarness	缺少验证报告则拒绝
Present	展示变更摘要、验证结果、经验教训	无
### 3.2.2 防护栏机制（Guardrails）

系统在代码层面强制执行以下约束：

约束	强制方式	违规后果
必须Spawn Verifier Subagent	complete 命令检查	缺少 verification-report.json 则拒绝
必须有 HTTP 证据	complete 命令检查	缺少 request/response 数据则拒绝
必须启动应用	complete 命令检查	server.started=false 则拒绝（除非 skip）

验证报告结构：

{
  "overall_status": "pass | partial | fail | skip",
  "server": { "started": true },
  "task_specific_scenarios": [
    {
      "name": "scenario-name",
      "request": { "method": "POST", "url": "...", "body": "..." },
      "response": { "status": 200, "body": "..." },
      "passed": true
    }
  ],
  "summary": { "task_specific_total": 3, "task_specific_passed": 3, "pass_rate": 1.0 }
}
### 3.2.3 功能验证场景设计

根据变更类型设计验证场景：

变更类型	验证场景
新增 API 接口	创建成功、参数验证错误、持久化检查
修改 API 接口	新行为生效、旧行为保持不变
新增验证逻辑	有效输入接受、无效输入拒绝
权限变更	授权用户成功、未授权用户拒绝
Bug 修复	具体问题已解决
## 3.3 目录结构
.qoder/skills/harness-executor/
├── SKILL.md              # 技能定义与 7 步流程
├── agents/               # 子 Agent 提示词
│   ├── verifier.md       # 功能验证子 Agent
│   ├── templates/        # 模板文件
│   └── mixins/           # 可复用片段
├── references/           # 指南文档
│   ├── scenario-design-guide.md      # 验证场景设计
│   ├── functional-verification-guide.md # 功能验证流程
│   ├── validation-guide.md           # 静态验证流程
│   ├── state-management.md           # 任务状态管理
│   ├── environment-schema.md         # environment.json 规范
│   ├── verify-schema.md              # 验证报告 Schema
│   └── ...更多参考文件
├── evals/                # 评估配置
│   └── evals.json        #评估任务定义
└── scripts/              # Python 脚本工具集
    ├── task_state.py     # 任务状态管理（init/checkpoint/complete）
    ├── memory_query.py   # Memory 查询接口
    ├── validate.py       # 静态验证执行
    ├── verify.py         # 功能验证执行
    ├── preflight.py      # 预检脚本
    ├── harness_critic.py # AutoHarness 优化建议
    └── ...更多脚本
# 4. 与 Spec 编程的关系
## 4.1 设计理念差异

Spec 编程方法（如 speckit、openspec）与 Harness Skills 均采用"计划→执行→验证"的工作流模式，但在设计理念、目标用户、基础设施策略等维度存在本质差异。

维度	Spec 编程	Harness Skills
核心目标	规范化开发流程、管理需求规格	为 AI Agent 构建"操作系统级"基础设施
主要使用者	人类开发者（AI 为辅助）	AI Agent（人类为监督者）
基础设施策略	依赖项目已有工具和文档	主动创建 AGENTS.md、linters、harness/
验证强度	通常为流程建议，可绕过	代码层面强制（Guardrails机制）
错误信息设计	面向人类理解	面向 Agent 自修复（WHAT + WHY + HOW）
记忆系统	通常无跨任务记忆	Episodic Memory + 经验教训积累
架构角色	单一流程	Coordinator + Subagent 分层架构
## 4.2 AGENTS.md 与 Spec 文件的定位差异
特性	Spec 文件	AGENTS.md
目的	定义任务需求与验收标准	Agent 的入口点与导航地图
位置	通常在 specs/ 或类似目录	仓库根目录，为 Agent 第一站
内容规模	可详细扩展	控制 80-120 行，链接到详细文档
生命周期	任务完成后归档或删除	长期存在，随项目持续演进
## 4.3 关系定位

两者并非互斥关系，而是分层互补：

┌─────────────────────────────────────────────┐
│  harness-creator → 创建基础设施（环境层）    │
└─────────────────────────────────────────────┘
                    ↓ 提供上下文
┌─────────────────────────────────────────────┐
│  speckit → 定义任务规格（规格层）            │
└─────────────────────────────────────────────┘
                    ↓ 提供计划
┌─────────────────────────────────────────────┐
│  harness-executor → 执行 + 强制验证（执行层）│
└─────────────────────────────────────────────┘

层级职责划分：

层级	工具/Skill	职责
环境层	harness-creator	创建 Agent 可理解的基础设施（一次性或按需）
规格层	speckit	定义需求规格、验收标准、执行计划
执行层	harness-executor	执行任务、强制验证、记录经验
## 4.4 组合使用建议
场景	建议方案
日常开发小需求	直接使用 speckit，无需 harness-executor
复杂多文件改动	使用 speckit + 借用 harness-executor 的验证脚本增强验证
AI Agent 自动执行	使用 harness-executor（其设计专为 Agent 独立执行）
新项目初始化	先用 harness-creator 建基础设施，后续用 speckit 管理需求
## 4.5 对接注意事项

组合使用时需注意以下对接点：

对接点	说明	处理方案
计划文件格式	harness-executor 使用 docs/exec-plans/active/*.md，speckit 有独立 spec 格式	speckit spec可作为 harness-executor Step 2 的输入
验证报告	harness-executor 要求 verification-report.json	可同时满足两者：完成 speckit 验收后生成 harness 报告
状态管理	harness-executor 用 task_state.py，speckit 有独立状态追踪	以 speckit 管理需求流程时，可不启用 harness 状态管理
# 5. 使用案例
## 5.1 harness-creator 调用
/harness-creator

最终生成的文件集合列表：

AGENTS.md

# AI 智能体平台
​
> RuoYi 微服务平台 + Spring AI 增强的企业级 AI 智能体平台
​
## 1. 技术栈
​
| 维度 | 选型 |
|------|------|
| **语言** | Java 21 |
| **构建** | Maven 3.9+ |
| **框架** | Spring Boot 3.3.5 + Spring Cloud 2023.0.3 + Spring Cloud Alibaba 2023.0.1.2 |
| **AI** | Spring AI 1.1.2 + Alibaba AI Extensions 1.1.2.2 |
| **前端** | Vue 3 + Element Plus + Vite |
| **测试** | JUnit 5 + Mockito |
​
## 2. 项目结构
​
```
com.ruoyi
├── ruoyi-gateway         // 网关服务 [8088]
├── ruoyi-auth            // 认证中心 [9200]
├── ruoyi-api             // 接口模块
│   ├── ruoyi-api-system      // 系统接口
│   ├── ruoyi-api-ai          // AI 接口
│   ├── ruoyi-api-file        // 文件接口
│   ├── ruoyi-api-tools       // 工具接口
│   └── ruoyi-api-integration // 集成接口
├── ruoyi-common          // 通用模块 (18个子模块)
│   ├── ruoyi-common-core     // 核心工具类
│   ├── ruoyi-common-redis    // 缓存服务
│   ├── ruoyi-common-security // 安全模块
│   └── ... (详见 docs/ARCHITECTURE.md)
├── ruoyi-modules         // 业务模块
│   ├── ruoyi-system         // 系统模块 [9201]
│   ├── ruoyi-ai             // AI 服务 [9204]
│   ├── ruoyi-agent          // 智能体 [9207]
│   ├── ruoyi-gen            // 代码生成 [9202]
│   ├── ruoyi-job            // 定时任务 [9203]
│   ├── ruoyi-file           // 文件服务 [9300]
│   └── ... (详见 docs/ARCHITECTURE.md)
└── ruoyi-visual          // 监控模块 [9100]
```
​
## 3. 核心依赖服务
​
| 服务 | 端口 | 用途 |
|------|------|------|
| MySQL 5.7 | 3308 | 主数据库 |
| Redis | 6381 | 缓存/权限认证 |
| Nacos v3 | 8848 | 服务注册/配置中心 |
| MinIO | 9010/9011 | 对象存储 |
| Milvus | 19530 | 向量数据库 |
| Nginx | 8090 | 前端代理 |
​
## 4. 常用命令
​
```bash
# 构建
mvn clean install -DskipTests
​
# 启动依赖服务 (Docker)
cd docker && docker-compose up -d ruoyi-mysql ruoyi-redis ruoyi-nacos
​
# 启动网关
java -jar ruoyi-gateway/target/ruoyi-gateway.jar
​
# 启动认证服务
java -jar ruoyi-auth/target/ruoyi-auth.jar
​
# 启动系统模块
java -jar ruoyi-modules/ruoyi-system/target/ruoyi-modules-system.jar
​
# 本地开发启动脚本
bin/run-gateway.bat
bin/run-auth.bat
bin/run-modules-system.bat
```
​
## 5. 数据库规范
​
- **表命名**: 单数形式，如 `sys_user` 而非 `sys_users`
- **主键**: `id` (BIGINT, 自增或雪花算法)
- **通用字段**: `create_by`, `create_time`, `update_by`, `update_time`, `remark`
​
## 6. 配置管理
​
- **配置中心**: Nacos (namespace: `ry-config`)
- **本地配置**: `application.yml` + `bootstrap.yml`
- **敏感配置**: 使用 Nacos 加密或环境变量
​
## 7. API 规范
​
- **文档**: SpringDoc OpenAPI (访问 `/swagger-ui.html`)
- **认证**: JWT Token (Header: `Authorization: Bearer {token}`)
- **统一响应**: `R<T>` 包装 (`code`, `msg`, `data`)
​
## 8. 模块依赖规则
​
```
ruoyi-modules/* → ruoyi-common/* → ruoyi-api/*
                         ↓
               Spring Boot/Cloud 依赖
​
禁止反向依赖！
禁止 common 模块之间的循环依赖！
```
​
## 9. 相关文档
​
| 文档 | 路径 | 说明 |
|------|------|------|
| 架构设计 | [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | 模块依赖、层级架构 |
| 开发指南 | [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md) | 构建测试、调试技巧 |
| 部署文档 | [doc/部署文档/](doc/部署文档/) | 生产环境部署 |
| 设计文档 | [doc/设计/](doc/design/) | 功能设计文档 |
​
## 10. 功能模块
​
| 模块 | 说明 |
|------|------|
| 统一认证 | 账号密码/微信扫码/SSO 登录 |
| AI 智能体 | 会话管理、消息管理、智能体反馈 |
| 智能内容生成 | PPT 智能生成、文档解析比对 |
| 专业领域 AI | 合同审查、财税问答、电商数据分析 |
| RAG 检索 | 知识库检索、多模式对话 |
| 系统支撑 | 定时任务、日志监控、文件服务 |
​
---
​
**版本**: v3.6.5 | **维护团队**: xxx
​




5.2 harness-executor 调用
/harness-executor

或通过 Skill 工具：

Skill(skill="harness-executor")

执行流程：

Setup：检测 AGENTS.md（缺失则调用 harness-creator）

Plan：确定修改范围，初始化任务状态

Execute：Spawn Executor Subagent 执行代码变更

Validate：静态验证（build + lint + test）

Verify：Spawn Verifier Subagent 功能验证

Record：完成任务、记录经验、触发 AutoHarness

Present：展示结果摘要

6. 总结

Harness Skills 代表了一种"为 AI Agent 构建操作系统"的设计范式，其核心价值在于：

基础设施主动创建：不依赖项目已有文档和工具，而是主动构建 Agent 可理解的环境

强制验证机制：通过代码层面的 Guardrails 确保验证不可跳过

Agent 可操作错误：Linter 和验证输出设计为 Agent 可自主修复的格式

经验积累系统：Episodic Memory 让 Agent 从历史任务中学习

与 Spec 编程方法的关系是分层互补而非互斥：

Spec 编程适合人类主导的开发流程

Harness Skills 适合 AI Agent 独立执行的场景

两者可组合使用：harness-creator 建环境 → speckit 定义规格 → harness-executor 执行验证

参考文件索引
harness-creator references/
文件	用途
greenfield-templates.md	空项目完整模板
documentation-templates.md	文档创建模板
linter-templates.md	Linter 脚本模板
environment-detection-guide.md	环境依赖检测指南
environment-config-guide.md	环境配置创建指南
architecture-diagrams.md	架构图绘制规范
audit-templates.md	审计报告模板
eval-templates.md	评估任务模板
harness-executor references/
文件	用途
scenario-design-guide.md	功能验证场景设计指南
functional-verification-guide.md	功能验证流程架构
validation-guide.md	静态验证流程指南
state-management.md	任务状态管理规范
environment-schema.md	environment.json 数据规范
verify-schema.md	验证报告 JSON Schema
subagent-patterns.md	子 Agent 设计模式
feedback-pipeline.md	反馈处理流程








