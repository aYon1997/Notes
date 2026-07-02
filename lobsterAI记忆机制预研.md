# OpenClaw 记忆机制预研与企业级记忆扩展方案

> **版本**：2026-05-07  
> **目的**：说明 OpenClaw 记忆从写入、切片、入库、检索、召回到长期沉淀的完整生命周期，并给出项目记忆、组织记忆的扩展设计。  
> **核心结论**：OpenClaw 的记忆不是一个简单的 `MEMORY.md` 文件，也不是一个简单向量库。它更像「**Markdown 记忆材料 + SQLite 本地索引 + 记忆工具 + 后台整理机制**」的组合。

---

## 目录

- [一、先建立一个正确心智模型](#一先建立一个正确心智模型)
- [二、OpenClaw 什么时候切片](#二openclaw-什么时候切片)
- [三、SQLite 里到底存了什么](#三sqlite-里到底存了什么)
- [四、一条记忆的完整生命周期示例](#四一条记忆的完整生命周期示例)
- [五、memory_search 到底怎么工作](#五memory_search-到底怎么工作)
- [六、QMD 后端的角色](#六qmd-后端的角色)
- [七、active-memory：为什么 Agent 不一定要自己想起来搜索](#七active-memory为什么-agent-不一定要自己想起来搜索)
- [八、LobsterAI 当前做了什么](#八lobsterai-当前做了什么)
- [九、当前机制的边界](#九当前机制的边界)
- [十、企业记忆应该怎么分层](#十企业记忆应该怎么分层)
- [十一、项目记忆生命周期](#十一项目记忆生命周期)
- [十二、组织记忆生命周期](#十二组织记忆生命周期)
- [十三、是否需要改 OpenClaw 源码](#十三是否需要改-openclaw-源码)
- [十四、推荐落地路线](#十四推荐落地路线)
- [十五、最终建议](#十五最终建议)

---

## 一、先建立一个正确心智模型

OpenClaw 记忆可以拆成三件事：

1. **记忆材料在哪里**  
   人能读懂、Agent 能写入的 Markdown 文件，例如 `MEMORY.md`、`memory/YYYY-MM-DD.md`、`memory/` 目录、额外路径。

2. **记忆怎么被搜索**  
   OpenClaw 会把这些文件切成 chunk，写入本地 SQLite 索引。搜索时不是每次全量读文件，而是查 SQLite 中的全文索引、向量索引和 chunk 表。

3. **记忆怎么进入对话**  
   Agent 可以显式调用 `memory_search` / `memory_get`。某些配置下，active-memory 会在 prompt 构建前主动召回一小段相关记忆。长会话压缩、短期记忆提升和 dreaming 也会把会话内容沉淀回文件。

### 整体架构图

```text
Markdown 记忆文件
  MEMORY.md
  memory/YYYY-MM-DD.md
  memory/**
  extraPaths
        |
        | 索引同步时切片
        v
SQLite 记忆索引
  files
  chunks
  chunks_fts
  chunks_vec
  embedding_cache
        |
        | memory_search / memory_get
        v
Agent 上下文
  显式工具召回
  active-memory 主动召回
  compaction flush 写回
  short-term promotion 长期化
```

> 💡 **关键理解**：`MEMORY.md` 是记忆材料之一；SQLite 是检索缓存和索引；`memory_search` 是 Agent 接触记忆系统的工具入口；真正的记忆生命周期横跨文件、索引、工具和后台任务。

---

## 二、OpenClaw 什么时候切片

OpenClaw 不是在用户说"记住"那一刻立刻切片，也不是每次 `memory_search` 都重新切全部文件。

**切片发生在索引同步阶段。**

典型触发点有五类：

### 2.1 首次搜索前索引为空

如果进程刚启动，SQLite 里还没有任何 chunk，此时 Agent 第一次调用 `memory_search`，OpenClaw 会强制做一次同步。

```text
memory_search(query)
  -> 发现没有 indexed content
  -> sync(reason = "search", force = true)
  -> 扫描记忆文件
  -> 切片
  -> 写 SQLite
  -> 再执行搜索
```

> 这保证第一次搜索不会因为后台索引还没跑完而直接返回空。

### 2.2 会话启动后的后台同步

会话启动时，memory manager 会启动 watcher，也可能异步触发一次同步。

这个同步通常**不阻塞用户对话**。它的作用是让当前工作区的记忆文件尽快进入索引。

```text
session start
  -> MemoryIndexManager 初始化
  -> ensureWatcher()
  -> 后台 sync(reason = "session-start")
```

### 2.3 文件 watcher 发现变化

OpenClaw 会监听这些位置：

```text
MEMORY.md
memory.md
memory/
memorySearch.extraPaths
```

当文件变化后，它不会立刻重建索引，而是：

```text
文件变化
  -> watcher 标记 dirty
  -> debounce，默认约 1500ms
  -> 后续同步时只处理变化文件
```

> ⚠️ **注意**：用户或 Agent 写入 `MEMORY.md` 后，新的内容要等 watcher 和同步流程跑完，才稳定进入 SQLite 检索索引。

### 2.4 搜索时发现索引脏了

如果 `memory_search` 发生时索引被标记为 dirty，OpenClaw 可以在搜索期间触发一次后台同步。

```text
memory_search(query)
  -> 当前索引可用，但 dirty = true
  -> 先用已有索引搜索
  -> 后台刷新索引
```

- 如果索引完全为空，会**强制同步**；
- 如果索引只是过期，通常不会为了刷新阻塞本次搜索。

### 2.5 session transcript 增量索引

会话 transcript **不是默认进入记忆搜索**。只有启用 experimental session memory，并且 sources 包含 `sessions`，才会索引 session 文件。

session 索引也不是每条消息都切片。它会根据增量阈值触发，例如 transcript 字节数变化、消息数变化或强制重建。

### 2.6 切片算法做了什么

切片函数处理 Markdown 内容，默认目标参数：

| 参数 | 默认值 |
|------|--------|
| chunk tokens | `400` |
| chunk overlap | `80` |
| watcher debounce | `1500ms` |

**切片逻辑**：按行累积文本，达到预算后 flush 一个 chunk，然后保留尾部 overlap 作为下一个 chunk 的开头。它还对中文等 CJK 内容做了更保守的字符估算，避免一长段中文被错误估成很少 token。

```text
读取文件内容
  -> 按行扫描
  -> 累积到约 400 tokens
  -> 生成一个 chunk
  -> 记录 start_line / end_line
  -> 保留约 80 tokens overlap
  -> 继续生成下一个 chunk
```

> 💡 **细节**：切片保存的是**行号范围**。后续 `memory_search` 返回结果时可以告诉 Agent 片段来自哪个文件、哪几行；`memory_get` 可以再按来源读取更完整原文。

---

## 三、SQLite 里到底存了什么

OpenClaw builtin 后端使用 SQLite 作为本地记忆索引。这个 SQLite 不是业务数据库，也不是记忆事实源。它更像**「搜索用缓存」**。

默认会有这些表：

| 表 | 作用 |
|----|------|
| `meta` | 存索引版本、同步元信息 |
| `files` | 每个被索引文件的一行状态 |
| `chunks` | 每个切片的一行主记录 |
| `embedding_cache` | embedding 结果缓存 |
| `chunks_fts` | FTS5 全文索引虚表 |
| `chunks_vec` | 可选 sqlite-vec 向量索引虚表 |

### 3.1 `files`：文件级状态

`files` 表用于判断文件有没有变化。

```text
files
  path      TEXT PRIMARY KEY
  source    TEXT
  hash      TEXT
  mtime     INTEGER
  size      INTEGER
```

**示例数据**：

| path | source | hash | mtime | size |
|------|--------|------|-------|------|
| `MEMORY.md` | `memory` | `a31f...` | `1778123000000` | `1842` |
| `memory/2026-05-07.md` | `memory` | `91bd...` | `1778123600000` | `6320` |
| `.lobster-memory/projects/lobsterai/PROJECT_MEMORY.md` | `memory` | `c822...` | `1778123900000` | `4200` |

同步时，OpenClaw 会拿当前文件 hash 和 `files.hash` 比较：
- 没变 → 跳过
- 变了 → 重新切片并更新索引

### 3.2 `chunks`：切片主表

`chunks` 是最关键的表。

```text
chunks
  id          TEXT PRIMARY KEY
  path        TEXT
  source      TEXT
  start_line  INTEGER
  end_line    INTEGER
  hash        TEXT
  model       TEXT
  text        TEXT
  embedding   TEXT
  updated_at  INTEGER
```

**字段解释**：

| 字段 | 含义 |
|------|------|
| `id` | chunk 唯一 ID，基于 source、path、行号、chunk hash、model 生成 |
| `path` | 来源文件相对路径 |
| `source` | 来源类型，主要是 `memory` 或 `sessions` |
| `start_line` / `end_line` | chunk 在原文件中的行号范围 |
| `hash` | chunk 文本 hash |
| `model` | 用哪个 embedding model 索引；没有 embedding 时是 `fts-only` |
| `text` | chunk 原文 |
| `embedding` | JSON 字符串形式的向量；FTS-only 时通常是 `[]` |
| `updated_at` | 索引更新时间 |

### 3.3 `chunks_fts`：全文检索表

如果 FTS 可用，OpenClaw 会把 chunk 写入 FTS5 虚表。

```text
chunks_fts
  text
  id
  path
  source
  model
  start_line
  end_line
```

- `text` 参与全文检索；其他字段主要用于回表和返回来源。
- Tokenizer 默认可以是 `unicode61`。
- LobsterAI 当前更适合中文场景时，会倾向配置 `trigram`——对中文、项目代号、无空格术语更友好。

### 3.4 `chunks_vec`：向量检索表

如果 sqlite-vec 可用，OpenClaw 会创建 `chunks_vec`：

```text
chunks_vec
  id TEXT PRIMARY KEY
  embedding FLOAT[dims]
```

它只保存向量索引用的数据。真正的文本、路径、行号仍在 `chunks` 表里。

> 如果 sqlite-vec 不可用，OpenClaw 仍然可以写 `chunks` 和 `chunks_fts`，只是向量召回降级。

### 3.5 `embedding_cache`：embedding 缓存

embedding cache 避免同一段文本反复请求 embedding provider。

主键大致是：

```text
provider + model + provider_key + hash
```

同样文本、同样 provider、同样 model 下，可以复用向量。

---

## 四、一条记忆的完整生命周期示例

下面用一个具体例子说明更容易理解。

假设用户在 LobsterAI Cowork 里说：

> 记住：LobsterAI 项目发版前必须先通过变更单审批，Windows 包还要额外验证 PortableGit。

### 4.1 阶段一：写入记忆材料

如果这是个人或当前工作区长期记忆，Agent 可能把它写进 `MEMORY.md`：

```markdown
# Memory
- LobsterAI 项目发版前必须先通过变更单审批，Windows 包还要额外验证 PortableGit。
```

如果是更偏当日会话沉淀，尤其在 compaction flush 时，它可能被追加到：

```text
memory/2026-05-07.md
```

```markdown
## Durable Notes
- LobsterAI 项目发版前必须先通过变更单审批，Windows 包还要额外验证 PortableGit。
```

> ⚠️ **重要**：**写入 Markdown 文件不等于已经进入 SQLite 索引。** 写入只是改变了记忆材料，之后要等索引同步。

### 4.2 阶段二：watcher 标记索引变脏

文件保存后，watcher 发现 `MEMORY.md` 或 `memory/2026-05-07.md` 变化。

```text
MEMORY.md changed
  -> dirty = true
  -> 等待 debounce
  -> 后续 sync 处理
```

如果这时马上搜索，可能出现两种情况：
- SQLite 之前没有任何 chunk → 本次搜索会**强制同步**，然后能搜到；
- SQLite 已经有旧索引 → 本次搜索可能**先用旧索引返回**，同时后台刷新。

### 4.3 阶段三：同步时切片

同步开始后，OpenClaw 扫描记忆文件。

```text
listMemoryFiles()
  -> MEMORY.md
  -> memory/2026-05-07.md
  -> memory/**
  -> extraPaths
```

然后对变化文件执行：

```text
buildFileEntry()
  -> 计算文件 hash / mtime / size
  -> 读取文件文本
  -> chunkMarkdown()
  -> 生成 MemoryChunk[]
```

如果这条记忆所在文件很短，可能只有一个 chunk：

```text
chunk #1
  path: MEMORY.md
  start_line: 3
  end_line: 3
  text: "- LobsterAI 项目发版前必须先通过变更单审批，Windows 包还要额外验证 PortableGit。"
```

如果文件较长，它可能和前后几行一起进入一个 400 tokens 左右的 chunk。

### 4.4 阶段四：写入 SQLite

同步会先清理这个文件旧的索引数据，再写入新 chunk。

**`files` 表**：

| path | source | hash | mtime | size |
|------|--------|------|-------|------|
| `MEMORY.md` | `memory` | `a31f...` | `1778123000000` | `1842` |

**`chunks` 表新增一行**：

| 字段 | 示例 |
|------|------|
| id | `7c92...` |
| path | `MEMORY.md` |
| source | `memory` |
| start_line | `3` |
| end_line | `3` |
| hash | `c098...` |
| model | `text-embedding-xxx` 或 `fts-only` |
| text | `- LobsterAI 项目发版前必须先通过变更单审批，Windows 包还要额外验证 PortableGit。` |
| embedding | `[0.012, -0.034, ...]` 或 `[]` |
| updated_at | `1778123020000` |

如果 FTS 可用，还会写入 `chunks_fts`：

| text | id | path | source | start_line | end_line |
|------|----|------|--------|------------|----------|
| 同 chunk 文本 | `7c92...` | `MEMORY.md` | `memory` | `3` | `3` |

如果向量可用，还会写入 `chunks_vec`：

| id | embedding |
|----|-----------|
| `7c92...` | `FLOAT[dims]` |

### 4.5 阶段五：后续搜索召回

过几天用户问：

> LobsterAI 发版前有什么特别要注意的吗？

Agent 或 active-memory 触发：

```json
{
  "query": "LobsterAI 发版前有什么特别要注意的吗？",
  "maxResults": 6,
  "minScore": 0.35,
  "corpus": "memory"
}
```

OpenClaw 执行：

```text
memory_search
  -> FTS 搜关键词：LobsterAI / 发版 / 注意
  -> 向量搜索语义相似 chunk
  -> hybrid merge
  -> minScore 过滤
  -> 返回 top results
```

默认 hybrid 公式：

```text
final_score = vector_score * 0.7 + text_score * 0.3
```

如果 embedding provider 不可用，则降级为 FTS-only。本例里它至少可以靠 `LobsterAI`、`发版` 这类关键词召回相关 chunk；如果用户后续直接问 `PortableGit` 或 `变更单`，FTS-only 也能靠这些明确术语命中。

> 💡 **搜索顺序**：`PortableGit` 和 `变更单` 是被召回 chunk 里的内容，不应该提前出现在 query 里。Agent 可以在第一次搜索命中后，再基于结果继续追问或调用 `memory_get` 精读。

**返回结果示例**：

```json
{
  "path": "MEMORY.md",
  "source": "memory",
  "startLine": 3,
  "endLine": 3,
  "score": 0.82,
  "text": "- LobsterAI 项目发版前必须先通过变更单审批，Windows 包还要额外验证 PortableGit。"
}
```

### 4.6 阶段六：必要时读取原文

如果搜索结果只是一个片段，Agent 可以继续调用 `memory_get`：

```json
{
  "path": "MEMORY.md",
  "startLine": 1,
  "endLine": 8
}
```

这样可以读取更完整上下文，避免只根据一个切片做过度推断。

### 4.7 阶段七：短期召回统计和长期提升

`memory_search` 返回结果后，OpenClaw 会记录一部分短期召回信息。

位置类似：

```text
memory/.dreams/short-term-recall.json
memory/.dreams/phase-signals.json
```

如果某条记忆被多次不同 query 召回，并且相关分数足够高，short-term promotion 或 dreaming 可能把它提升到 `MEMORY.md` 的长期区块。

**示例**：

```markdown
## Promoted From Short-Term Memory (2026-05-07)
<!-- openclaw-memory-promotion:... -->
- LobsterAI 项目发版前必须先通过变更单审批，Windows 包还要额外验证 PortableGit。
```

> 💡 这个机制说明 OpenClaw 有"短期内容长期化"的思路。但它仍然是**个人/工作区级别的自动整理**，不是企业知识审批。

### 4.8 阶段八：修改或删除后的索引更新

如果用户后来把记忆改成：

```markdown
- LobsterAI 项目发版前必须先通过变更单审批，Windows 包要验证 PortableGit 和安装包签名。
```

下一次同步时：

```text
发现 MEMORY.md hash 变化
  -> 删除 MEMORY.md 旧 chunks / FTS rows / vector rows
  -> 重新切片
  -> 写入新 chunks
  -> 更新 files.hash
```

> ⚠️ **重要**：SQLite 中旧文本不会作为同一文件的当前 chunk 长期保留。它是**索引，不是版本历史**。
>
> 如果企业需要版本历史，不能依赖 OpenClaw SQLite，需要在 LobsterAI 或企业后端**单独设计版本表和审计表**。

---

## 五、memory_search 到底怎么工作

`memory_search` 不是简单 grep。

它的完整路径大致是：

```text
Agent 调用 memory_search
  -> 解析 query / maxResults / minScore / corpus
  -> 选择 memory backend：builtin 或 QMD
  -> builtin 下获取 MemoryIndexManager
  -> 必要时同步索引
  -> 查询 FTS 和向量索引
  -> hybrid merge
  -> 返回带来源的片段
  -> 记录短期召回
```

### 5.1 参数

常见参数：

```json
{
  "query": "要搜索的内容",
  "maxResults": 6,
  "minScore": 0.35,
  "corpus": "memory"
}
```

`corpus` 可以是：

| 值 | 含义 |
|----|------|
| `memory` | 搜 OpenClaw 记忆材料 |
| `wiki` | 搜 compiled-wiki 补充语料 |
| `all` | 同时搜索 memory 和 wiki |

### 5.2 默认搜索策略

| 参数 | 默认值 |
|------|--------|
| maxResults | `6` |
| minScore | `0.35` |
| hybrid enabled | `true` |
| vector weight | `0.7` |
| text weight | `0.3` |
| candidate multiplier | `4` |
| MMR | 默认关闭 |
| temporal decay | 默认关闭 |

搜索候选数通常是：

```text
candidates = min(200, maxResults * candidateMultiplier)
```

默认 `maxResults=6`、`candidateMultiplier=4` 时，会先取约 **24 个候选**，再融合排序。

### 5.3 为什么要同时做 FTS 和向量

| FTS 擅长精确词 | 向量擅长语义相近 |
|----------------|------------------|
| 项目代号 | "发版注意事项"匹配"上线前检查" |
| 人名 | "审批流程"匹配"变更单" |
| 系统名 | "Windows 打包风险"匹配"PortableGit 验证" |
| 文件名 | |
| 内部术语 | |
| 错误码 | |
| `PortableGit` 这种专有名词 | |

> 💡 **企业知识通常两者都需要**。只用向量会漏掉专有名词，只用关键词会漏掉表达不同但语义相同的内容。

---

## 六、QMD 后端的角色

OpenClaw 的 memory backend 可以是 builtin，也可以是 QMD。

**Builtin 和 QMD 的根本区别**：Builtin 的 SQLite 表由 OpenClaw 自己创建和写入；QMD 的 SQLite 索引由 QMD 创建和维护，OpenClaw 只负责配置 collection、触发 update/embed、调用搜索命令、把结果转成统一的 `memory_search` 返回格式。

### QMD 主流程

```text
Markdown 记忆材料
  -> QMD collection
  -> qmd update / qmd embed
  -> QMD index.sqlite
  -> qmd search / vsearch / query
  -> OpenClaw 转成 MemorySearchResult
  -> memory_get 回到原 Markdown 文件
```

> 📌 **说明**：本节的 QMD 表结构以 QMD 上游 README 和 `src/store.ts` 当前实现为口径；实际部署时仍应以目标机器 `index.sqlite` 的 `.schema` 为最终依据。

### 6.1 QMD 的记忆材料在哪里

QMD 仍然以文件作为可索引材料，但它不是像 builtin 那样直接扫描 `MEMORY.md`、`memory/`、`extraPaths` 后自己切片，而是先把这些路径注册成 **collection**。

一个 collection 可以理解成：

```text
collection
  name      collection 名称
  path      collection 根目录
  pattern   文件匹配规则
  kind      memory | custom | sessions
```

默认情况下，如果没有关闭 `includeDefaultMemory`，OpenClaw 会给 QMD 创建两个默认 collection：

| collection | path | pattern | kind |
|------------|------|---------|------|
| `memory-root-<agentId>` | 当前工作区 | `MEMORY.md` | `memory` |
| `memory-dir-<agentId>` | 当前工作区的 `memory/` | `**/*.md` | `memory` |

此外还会合并：
- `memory.qmd.paths`
- `memorySearch.extraPaths`
- `memorySearch.qmd.extraCollections`
- 可选 session export collection

**项目记忆映射示例**：

```yaml
memory:
  backend: qmd
  qmd:
    paths:
      - name: project-lobsterai
        path: .lobster-memory/projects/lobsterai
        pattern: "**/*.md"
```

**组织记忆映射示例**：

```yaml
memory:
  backend: qmd
  qmd:
    paths:
      - name: engineering-standards
        path: .lobster-memory/org/engineering
        pattern: "**/*.md"
      - name: release-process
        path: .lobster-memory/org/release
        pattern: "**/*.md"
```

### 6.2 QMD 底层 SQLite 表结构

QMD 底层也会使用 SQLite 索引。QMD 默认索引文件是：

```text
~/.cache/qmd/index.sqlite
```

在 OpenClaw 中，为了隔离不同 Agent，OpenClaw 会把 QMD 的 XDG 目录改到 agent state 下：

```text
state/
  agents/<agentId>/qmd/
    xdg-config/
      qmd/
        index.yml 等配置
    xdg-cache/
      qmd/
        index.sqlite
        models/
```

OpenClaw 启动 QMD 时会设置：

```text
XDG_CONFIG_HOME = <agentStateDir>/qmd/xdg-config
QMD_CONFIG_DIR  = <agentStateDir>/qmd/xdg-config/qmd
XDG_CACHE_HOME  = <agentStateDir>/qmd/xdg-cache
```

因此 OpenClaw 实际使用的 QMD 索引通常是：

```text
<agentStateDir>/qmd/xdg-cache/qmd/index.sqlite
```

> ⚠️ **注意**：QMD 的 schema 归 QMD 管，不归 OpenClaw 管。OpenClaw 只是设置环境变量、调用 QMD 命令，并在需要定位原文时读取 QMD 索引中的文档位置。

**QMD 主要表概览**：

| 表 | 作用 | 类比 Builtin |
|----|------|--------------|
| `store_collections` | collection 配置：名称、根路径、glob、忽略规则、上下文 | 比 builtin 的 `extraPaths` 更结构化 |
| `store_config` | QMD store 级 key-value 配置 | builtin 的 `meta` |
| `content` | 内容寻址存储，按文档 hash 保存原文 | builtin `chunks.text` 的上游原文层 |
| `documents` | 文件元数据：collection、path、title、hash、active | builtin 的 `files`，但带 collection 和 title |
| `documents_fts` | FTS5 全文索引，索引 filepath、title、body | builtin 的 `chunks_fts` |
| `content_vectors` | 文档 embedding chunk 元数据：hash、seq、pos、model | builtin 的 `chunks` 中 embedding 元信息的一部分 |
| `vectors_vec` | sqlite-vec 向量索引，保存 `hash_seq -> embedding` | builtin 的 `chunks_vec` |
| `llm_cache` | 查询扩展、rerank 等 LLM 调用缓存 | builtin 没有完全等价表 |

> 📌 上游 README 里仍能看到 `collections`、`path_contexts` 这类旧名称；当前 `store.ts` 源码里已经把 collection/config 放到 `store_collections`、`store_config`，并会清理旧表。实际部署版本可能不同，排障时应以目标机器上 `index.sqlite` 的 `.schema` 为准。

#### 6.2.1 `store_collections`

`store_collections` 是 QMD 的 collection 真正落库位置。

```text
store_collections
  name                TEXT PRIMARY KEY
  path                TEXT NOT NULL
  pattern             TEXT NOT NULL DEFAULT '**/*.md'
  ignore_patterns     TEXT
  include_by_default  INTEGER DEFAULT 1
  update_command      TEXT
  context             TEXT
```

**示例**：

| name | path | pattern | include_by_default |
|------|------|---------|--------------------|
| `project-lobsterai` | `.lobster-memory/projects/lobsterai` | `**/*.md` | `1` |
| `engineering-standards` | `.lobster-memory/org/engineering` | `**/*.md` | `1` |

这张表决定 QMD update 扫描哪些目录。

#### 6.2.2 `content`

`content` 是内容寻址表。

```text
content
  hash        TEXT PRIMARY KEY
  doc         TEXT NOT NULL
  created_at  TEXT NOT NULL
```

QMD 会对文档内容计算 SHA-256 hash。同样内容只保存一份。

| hash | doc | created_at |
|------|-----|------------|
| `7c92...` | `# Project Memory...PortableGit。` | `2026-05-07T10:00:00Z` |

#### 6.2.3 `documents`

`documents` 是文件元数据表，负责把 collection/path 映射到 content hash。

```text
documents
  id           INTEGER PRIMARY KEY AUTOINCREMENT
  collection   TEXT NOT NULL
  path         TEXT NOT NULL
  title        TEXT NOT NULL
  hash         TEXT NOT NULL
  created_at   TEXT NOT NULL
  modified_at  TEXT NOT NULL
  active       INTEGER NOT NULL DEFAULT 1
  UNIQUE(collection, path)
```

**示例**：

| id | collection | path | title | hash | active |
|----|------------|------|-------|------|--------|
| `42` | `project-lobsterai` | `PROJECT_MEMORY.md` | `Project Memory: LobsterAI` | `7c92...` | `1` |

- `active` 是**软删除标记**。文件从磁盘删除后，QMD 不一定立刻物理删除记录，而是把 `active` 置为 `0`，使它不参与搜索。

QMD 还会建索引：

```text
idx_documents_collection(collection, active)
idx_documents_hash(hash)
idx_documents_path(path, active)
```

这些索引用于 collection 过滤、docid/hash 定位、按路径读取。

#### 6.2.4 `documents_fts`

`documents_fts` 是 FTS5 虚表。

```text
documents_fts
  filepath
  title
  body
```

Tokenizer 使用：

```text
porter unicode61
```

QMD 用触发器保持 FTS 与 `documents/content` 同步：

```text
documents_ai  AFTER INSERT
documents_ad  AFTER DELETE
documents_au  AFTER UPDATE
```

当 `documents.active = 1` 时，FTS 行包含：

```text
filepath = collection || '/' || path
title    = documents.title
body     = content.doc
```

> 💡 这意味着 QMD 的关键词搜索不是搜 OpenClaw 的 chunk 表，而是搜 QMD 自己的 `documents_fts`。

#### 6.2.5 `content_vectors`

`content_vectors` 保存文档切片后的 embedding 元信息。

```text
content_vectors
  hash         TEXT NOT NULL
  seq          INTEGER NOT NULL DEFAULT 0
  pos          INTEGER NOT NULL DEFAULT 0
  model        TEXT NOT NULL
  embedded_at  TEXT NOT NULL
  PRIMARY KEY (hash, seq)
```

| 字段 | 含义 |
|------|------|
| `hash` | 文档内容 hash |
| `seq` | 第几个 embedding chunk |
| `pos` | 该 chunk 在原文中的字符位置 |
| `model` | embedding 模型 |
| `embedded_at` | embedding 时间 |

> QMD 默认把文档切成约 **900 token** 的 chunk，重叠约 **15%**。这个 chunk 主要用于向量搜索和 rerank，而不是像 OpenClaw builtin 那样直接作为 `memory_search` 的唯一返回单位。

#### 6.2.6 `vectors_vec`

`vectors_vec` 是 sqlite-vec 虚表。

```text
vectors_vec
  hash_seq   TEXT PRIMARY KEY
  embedding  float[dimensions] distance_metric=cosine
```

`hash_seq` 的格式是：

```text
<hash>_<seq>
```

| hash_seq | embedding |
|----------|-----------|
| `7c92..._0` | `float[dimensions]` |
| `7c92..._1` | `float[dimensions]` |

向量搜索时，QMD 先查 `vectors_vec`，再回到 `content_vectors`、`documents`、`content` 找文档和原文。

#### 6.2.7 `llm_cache` 与 `store_config`

`llm_cache` 缓存查询扩展、rerank 等 LLM 调用结果：

```text
llm_cache
  hash        TEXT PRIMARY KEY
  result      TEXT NOT NULL
  created_at  TEXT NOT NULL
```

`store_config` 是 QMD store 的 key-value 配置：

```text
store_config
  key    TEXT PRIMARY KEY
  value  TEXT
```

这两张表让 QMD 能避免重复扩展 query，也能记录 store 级配置。

#### 6.2.8 和 Builtin 的表级对照

| 层 | Builtin | QMD |
|----|---------|-----|
| SQLite 文件 | OpenClaw memory sqlite | QMD `index.sqlite` |
| schema 所有者 | OpenClaw | QMD |
| collection 配置 | 没有一等表，靠 path/extraPaths | `store_collections` |
| 文件状态 | `files` 表 | QMD `documents` 表 |
| 原文存储 | `chunks.text` 保存片段 | `content.doc` 保存整篇原文 |
| chunk 主表 | `chunks` | `content_vectors(hash, seq, pos)` 记录向量 chunk |
| 全文索引 | `chunks_fts` | `documents_fts` |
| 向量索引 | `chunks_vec` | `vectors_vec` |
| embedding 缓存 | `embedding_cache` | `content_vectors` + `vectors_vec`，另有 `llm_cache` |
| 搜索入口 | OpenClaw 直接 SQL / sqlite-vec | QMD 自己执行 SQL / sqlite-vec，OpenClaw 调 QMD |
| 原文定位 | `chunks.path` | `documents` 表 + collection root |

### 6.3 一条记忆如何进入 QMD 索引

仍然用前面的记忆：

```markdown
- LobsterAI 项目发版前必须先通过变更单审批，Windows 包还要额外验证 PortableGit。
```

假设它不写入 `MEMORY.md`，而是作为项目记忆物化到：

```text
.lobster-memory/projects/lobsterai/PROJECT_MEMORY.md
```

**文件内容**：

```markdown
# Project Memory: LobsterAI
## Release
- LobsterAI 项目发版前必须先通过变更单审批，Windows 包还要额外验证 PortableGit。
```

LobsterAI 配置 QMD collection：

```yaml
memory:
  backend: qmd
  qmd:
    paths:
      - name: project-lobsterai
        path: .lobster-memory/projects/lobsterai
        pattern: "**/*.md"
```

OpenClaw 初始化 QMD manager 时，会确保 collection 存在：

```text
qmd collection add .lobster-memory/projects/lobsterai project-lobsterai --pattern "**/*.md"
```

然后触发：

```text
qmd update
```

这个阶段由 QMD 扫描 collection、读取 Markdown、计算文档 hash、写入自己的 `index.sqlite`。

**`qmd update` 四类判断**：

| 文件状态 | QMD 行为 |
|----------|----------|
| 新文件 | 插入 `content`，插入 `documents(active=1)`，触发 FTS 插入 |
| 内容未变化 | hash 相同，基本跳过 |
| 内容变化 | 插入新的 `content` hash，更新 `documents.hash/title/modified_at`，触发 FTS 更新 |
| 文件被删除 | 将 `documents.active` 置为 `0`，触发 FTS 删除 |

> 这和 builtin 的 `files.hash` 对比很像，但 QMD 多了一层 **content-addressable storage**：原文先进入 `content`，文件元数据再通过 hash 指向它。

**`content` 表**：

| hash | doc | created_at |
|------|-----|------------|
| `7c92...` | `# Project Memory: LobsterAI ... PortableGit。` | `2026-05-07T10:00:00Z` |

**`documents` 表**：

| collection | path | hash | active |
|------------|------|------|--------|
| `project-lobsterai` | `PROJECT_MEMORY.md` | `7c92...` | `1` |

因为有 FTS trigger，`documents_fts` 也会被同步：

| rowid | filepath | title | body |
|-------|----------|-------|------|
| `42` | `project-lobsterai/PROJECT_MEMORY.md` | `Project Memory: LobsterAI` | `# Project Memory...PortableGit。` |

如果后续运行：

```text
qmd embed
```

QMD 会把文档内容切成 embedding chunk，并写入 `content_vectors` 与 `vectors_vec`。

**`content_vectors` 示例**：

| hash | seq | pos | model | embedded_at |
|------|-----|-----|-------|-------------|
| `7c92...` | `0` | `0` | `embeddinggemma` | `2026-05-07T10:02:00Z` |

**`vectors_vec` 示例**：

| hash_seq | embedding |
|----------|-----------|
| `7c92..._0` | `float[dimensions]` |

> OpenClaw 不直接写这些表，只能通过 QMD 命令和 `qmd status` 判断向量是否可用，例如看到 `Vectors: 123`。

**QMD 入库过程总结**：

```text
Markdown 文件
  -> collection 绑定
  -> qmd update
     -> content 写原文
     -> documents 写文件状态
     -> documents_fts 同步全文索引
  -> qmd embed
     -> content_vectors 写 chunk 元信息
     -> vectors_vec 写向量
  -> OpenClaw 只记录 manager 状态，不直接写 QMD 表
```

### 6.4 QMD 什么时候入库和更新

QMD 不会在用户写文件的瞬间同步入库。它和 builtin 一样，也有同步时机。

| 触发点 | 发生什么 |
|--------|----------|
| boot update | QMD manager 初始化后默认执行 `qmd update` |
| watcher update | collection 下文件 add/change/unlink 后，标记 dirty 并防抖同步 |
| interval update | 默认约每 5 分钟执行一次 `qmd update` |
| search dirty sync | 如果搜索前发现 dirty，先执行 sync |
| manual sync | 外部显式调用 sync |
| embed interval | 默认约每 60 分钟执行一次 `qmd embed` |

**流程示意**：

```text
PROJECT_MEMORY.md changed
  -> watcher 标记 dirty
  -> debounce
  -> sync(reason = "watch")
  -> qmd update
  -> 如果需要，qmd embed
```

**默认关键参数**：

| 参数 | 默认值 |
|------|--------|
| searchMode | `search` |
| maxResults | `6` |
| maxSnippetChars | `700` |
| maxInjectedChars | `4000` |
| search timeout | `4000ms` |
| update interval | `5m` |
| update debounce | `15000ms` |
| embed interval | `60m` |
| update/embed timeout | `120000ms` |

### 6.5 QMD 如何搜索

当用户问：

> LobsterAI 发版前有什么特别要注意的吗？

OpenClaw 里的 `memory_search` 参数仍然是统一格式：

```json
{
  "query": "LobsterAI 发版前有什么特别要注意的吗？",
  "maxResults": 6,
  "minScore": 0.35,
  "corpus": "memory"
}
```

如果 backend 是 QMD，OpenClaw 不会查 builtin 的 `chunks_fts` 或 `chunks_vec`，而是根据 `searchMode` 调 QMD。

**三种搜索模式**：

| 模式 | 命令 | 查什么 | 类比 Builtin |
|------|------|--------|--------------|
| `search` | `qmd search "..." --json -n 6 -c project-lobsterai` | QMD 的词法/BM25 索引 | 类似查 `chunks_fts` |
| `vsearch` | `qmd vsearch "..." --json -n 6 -c project-lobsterai` | QMD 的向量索引 | 类似查 `chunks_vec` |
| `query` | `qmd query "..." --json -n 6 -c project-lobsterai` | QMD 的混合查询，可能包含 lex、vec、hyde/rerank | 类似 hybrid search，但由 QMD 决定融合方式 |

#### `qmd search`：全文/BM25 路径

```text
query
  -> documents_fts MATCH
  -> BM25/rank 排序
  -> rowid 回到 documents.id
  -> documents.hash 回到 content.doc
  -> 生成 snippet、score、docid
```

主要依赖：

```text
documents_fts(filepath, title, body)
documents(id, collection, path, title, hash, active)
content(hash, doc)
```

#### `qmd vsearch`：向量路径

```text
query
  -> 用 embedding 模型生成 query vector
  -> vectors_vec 做近邻搜索
  -> hash_seq 拆回 hash + seq
  -> content_vectors 找 chunk 位置 pos/model
  -> documents 根据 hash 找文档
  -> content 根据 hash 取原文并生成 snippet
```

主要依赖：

```text
vectors_vec(hash_seq, embedding)
content_vectors(hash, seq, pos, model, embedded_at)
documents(collection, path, hash, active)
content(hash, doc)
```

#### `qmd query`：混合路径

```text
query
  -> 可能先做 query expansion
  -> 生成 lex / vec / hyde 子查询
  -> 分别走 FTS 和 vector
  -> 合并候选，通常会做 rerank
  -> 缓存 LLM expansion/rerank 结果
```

主要额外依赖：

```text
llm_cache(hash, result, created_at)
```

> 💡 **和 builtin 最大的差异**：builtin 的 hybrid merge 在 OpenClaw 里做；QMD 的 hybrid、query expansion、rerank 在 QMD 内部做，OpenClaw 只拿最终 JSON 结果。

如果有多个 collection，OpenClaw 会逐个 collection 搜索，再把结果按 `docid` 或文件路径去重，保留分数更高的结果。

### 6.6 QMD 搜索结果长什么样

QMD CLI 输出 JSON。OpenClaw 期望解析出这些字段：

```json
[
  {
    "docid": "7c92...",
    "collection": "project-lobsterai",
    "file": "PROJECT_MEMORY.md",
    "score": 0.81,
    "snippet": "@@ -5,1\n- LobsterAI 项目发版前必须先通过变更单审批，Windows 包还要额外验证 PortableGit。",
    "start_line": 5,
    "end_line": 5
  }
]
```

| 字段 | 含义 |
|------|------|
| `docid` | QMD 文档 ID，通常来自内容 hash 的短前缀；OpenClaw 会用它回查 `documents.hash` |
| `collection` | 命中的 collection |
| `file` | collection 内的文件路径 |
| `score` | QMD 返回的相关性分数 |
| `snippet` | 命中的片段 |
| `start_line` / `end_line` | 命中片段行号，可能来自 QMD 或 snippet header |

OpenClaw 解析后，还会做几件事：

```text
解析 JSON
  -> 用 docid 在 QMD index.sqlite 的 documents 表里找 collection/path
  -> 如果 docid 找不到，尝试用 collection/file hint 找 path
  -> 截断 snippet 到 maxSnippetChars
  -> minScore 过滤
  -> 限制总注入字符 maxInjectedChars
  -> 转成 MemorySearchResult
```

最终返回给 Agent 的结果就变成统一格式：

```json
{
  "path": "qmd/project-lobsterai/PROJECT_MEMORY.md",
  "source": "memory",
  "startLine": 5,
  "endLine": 5,
  "score": 0.81,
  "text": "- LobsterAI 项目发版前必须先通过变更单审批，Windows 包还要额外验证 PortableGit。"
}
```

**`path` 的两种情况**：

| 情况 | path 形式 |
|------|-----------|
| 文件在 workspace 内 | workspace 相对路径，如 `memory/2026-05-07.md` |
| 文件在 workspace 外部 collection | 虚拟路径，如 `qmd/project-lobsterai/PROJECT_MEMORY.md` |

> 虚拟路径不是实际磁盘根路径，而是 OpenClaw 为了后续 `memory_get` 能安全回到 collection 而构造的路径。

### 6.7 QMD 的 `memory_get` 怎么读原文

QMD 的搜索结果来自 QMD 索引，但 `memory_get` 精读时仍然回到 Markdown 文件。

如果 path 是：

```text
qmd/project-lobsterai/PROJECT_MEMORY.md
```

OpenClaw 会解析：

```text
collection = project-lobsterai
relativePath = PROJECT_MEMORY.md
root = .lobster-memory/projects/lobsterai
absPath = root/PROJECT_MEMORY.md
```

然后做**安全检查**：
- collection 必须存在；
- 解析出来的文件必须仍在 collection 根目录内；
- workspace 相对路径必须仍在 workspace 内；
- 只允许读取 Markdown；
- 支持按行读取。

**QMD 下的 `memory_get` 流程**：

```text
memory_get(qmd/project-lobsterai/PROJECT_MEMORY.md)
  -> 解析 collection
  -> 找 collection root
  -> 防止路径逃逸
  -> 读取 Markdown 原文
  -> 返回指定行范围
```

**和 builtin 的对比**：

| 行为 | Builtin | QMD |
|------|---------|-----|
| 搜索结果来自 | OpenClaw `chunks` 表 | QMD search JSON |
| 定位原文 | `chunks.path` | `documents` 表 + collection root |
| 精读原文 | workspace 文件 | workspace 文件或 collection 文件 |

### 6.8 用同一条记忆走一遍 QMD 生命周期

完整串起来：

```text
1. LobsterAI 生成项目记忆文件
   .lobster-memory/projects/lobsterai/PROJECT_MEMORY.md

2. OpenClaw 注册 QMD collection
   qmd collection add .lobster-memory/projects/lobsterai project-lobsterai --pattern "**/*.md"

3. QMD 入库
   qmd update
   -> QMD 扫描 PROJECT_MEMORY.md
   -> 写 content(hash, doc)
   -> 写 documents(collection, path, title, hash, active)
   -> 触发 documents_fts 更新

4. QMD 生成向量
   qmd embed
   -> 写 content_vectors(hash, seq, pos, model)
   -> 写 vectors_vec(hash_seq, embedding)

5. 用户提问
   "LobsterAI 发版前有什么特别要注意的吗？"

6. OpenClaw 调 QMD 搜索
   qmd search "...?" --json -n 6 -c project-lobsterai

7. QMD 返回
   docid + collection + file + snippet + score

8. OpenClaw 回查文档位置
   documents.hash/docid -> collection/path

9. OpenClaw 转成 memory_search 结果
   qmd/project-lobsterai/PROJECT_MEMORY.md#L5

10. Agent 如需精读
    memory_get(qmd/project-lobsterai/PROJECT_MEMORY.md)
    -> 读取原 Markdown
```

### 6.9 QMD 的 fallback 和补充能力

OpenClaw 配置 QMD 后，仍会保留 fallback 思路。

**创建阶段**：

```text
backend = qmd
  -> 检查 workspace
  -> 检查 qmd binary
  -> 创建 QmdMemoryManager
  -> 失败则 fallback 到 builtin
```

**运行阶段**：

```text
QMD search/read 失败
  -> 标记 QMD primary failed
  -> 关闭 QMD manager
  -> fallback 到 builtin MemoryIndexManager
```

QMD 还可以通过 mcporter/MCP 调用，而不是直接 CLI。此时 `query` 模式会被拆成 typed searches：

| searchMode | MCP searches |
|------------|--------------|
| `search` | `lex` |
| `vsearch` | `vec` |
| `query` | `lex + vec + hyde` |

这些是调用方式差异，不改变主生命周期。

### 6.10 QMD 对企业记忆的价值和边界

QMD 比 builtin 更适合承载项目和组织记忆，核心原因是 **collection**。

可以把知识域拆成：

```text
personal-memory
project-lobsterai
engineering-standards
release-process
security-policy
```

> ⚠️ **但 QMD collection 仍然不是企业权限模型**。它只能表达"索引哪些资料、按什么集合搜索"，不能表达：
> - 谁有权限读；
> - 谁能审批；
> - 哪条记忆已生效；
> - 哪条内容过期；
> - 哪些内容不能注入 prompt；
> - 哪次召回需要审计。

**企业级设计定位**：

```text
LobsterAI / 企业后端:
  决定用户可访问哪些 project/org memory
  决定生成哪些 QMD collection 或调用哪些 scoped search
  负责 ACL、审批、版本、审计

QMD:
  负责多 collection 的索引、全文搜索、向量搜索和结果召回

OpenClaw:
  负责调用 QMD，把结果统一成 memory_search / memory_get 能消费的形式
```

---

## 七、active-memory：为什么 Agent 不一定要自己想起来搜索

只靠 prompt 告诉 Agent"需要时搜索记忆"是不稳定的。模型可能忘记搜索，也可能搜索太晚。

OpenClaw 的 active-memory 提供了一个**主动路径**：

```text
before_prompt_build
  -> 从当前消息或最近上下文构造 recall query
  -> 启动子 Agent
  -> 子 Agent 只能用 memory_search / memory_get
  -> 总结少量相关记忆
  -> 注入主 Agent prompt
```

它解决的是"**自动回忆**"的体验问题。

> ⚠️ **企业场景风险**：
> - 如果 scope 不受控，可能召回无关项目资料；
> - 如果 ACL 不受控，可能召回无权限组织知识；
> - 如果 classification 不受控，可能把敏感内容直接注入 prompt；
> - 如果 citation 不强制，回答无法追溯。
>
> 所以项目记忆、组织记忆要进入 active-memory，必须先做 **scope-aware policy**。

---

## 八、LobsterAI 当前做了什么

LobsterAI 当前对 OpenClaw 记忆做了上层接入，但范围有限。

### 8.1 设置页记忆管理

当前设置页主要管理 `MEMORY.md` 中的顶层 bullet。

**能力包括**：
- 读取 `MEMORY.md`；
- 解析顶层 `- xxx` 条目；
- 新增、编辑、删除；
- 搜索这些 bullet；
- 从旧 SQLite `user_memories` 懒迁移到 `MEMORY.md`。

**这不是 OpenClaw 全量记忆管理**。它看不到或不能完整管理：
- `memory/YYYY-MM-DD.md`；
- `memory/.dreams/*`；
- `extraPaths`；
- session memory；
- QMD collections；
- active-memory 注入结果；
- SQLite 中的 chunk 状态。

### 8.2 配置同步

LobsterAI 会同步 OpenClaw 工作区配置和 `AGENTS.md` 记忆策略。

它会告诉 Agent：
- 用户明确要求 remember 时，要先写入再确认；
- 可写入 `MEMORY.md` 或 `memory/YYYY-MM-DD.md`；
- 回答和工作时需要参考长期记忆；
- 如果开启 embedding memory，会同步 `memorySearch` provider、model、FTS tokenizer、hybrid 等配置。

> 💡 **定位**：LobsterAI 当前更像 OpenClaw 记忆的**产品化外壳**，不是替代 OpenClaw 记忆机制。

---

## 九、当前机制的边界

OpenClaw 这套机制适合个人和工作区级 Agent 记忆，但不能直接当企业组织记忆系统。

| 边界 | 说明 |
|------|------|
| SQLite 是索引，不是事实源 | 修改、删除、版本、审批不能靠它 |
| 原生 scope 很弱 | 主要只有 `memory` / `sessions`，没有 personal / project / organization |
| 文件路径不是权限模型 | `extraPaths` 能扩展索引范围，但不能表达员工 ACL |
| Agent 可写文件 | 适合个人记忆，不适合组织制度 |
| active-memory 需要约束 | 否则会主动召回过宽内容 |
| 组织知识需要 citation | 不能只给模型一段无来源摘要 |
| GUI 视图不完整 | 当前只覆盖 `MEMORY.md` bullet |

> ❌ **不能这样设计**：
> ```text
> 组织知识全部丢到 memory/org/
> 然后让 OpenClaw 自动搜索
> ```
> 这能做 demo，但不适合公司内部长期使用。

---

## 十、企业记忆应该怎么分层

建议把记忆分成三类。

| 类型 | 事实源 | 客户端角色 | 写入方式 | 读取方式 |
|------|--------|------------|----------|----------|
| **个人记忆** | 可以是本地，也可以同步到用户云端 | 本地编辑、缓存、同步 | 用户确认，个人助手生成候选 | 默认可读，受用户控制 |
| **项目记忆** | 企业后端 | 只读缓存、项目上下文同步、候选提交 | 项目成员或 Agent 提交候选，后端确认后生效 | 当前项目会话按权限读取 |
| **组织记忆** | 企业后端 | 发起查询，不应全量持有 | 知识负责人维护，审批发布 | 按用户、部门、密级、场景受控检索 |

> 🔑 **关键原则**：**项目记忆和组织记忆应该坐在企业服务器后端，客户端只做同步、缓存、检索代理和修改请求。**

这三类不要放在同一个 `MEMORY.md` 里。更准确的推荐架构是：

### 推荐架构图

```text
企业后端 Memory Service，事实源
  personal_memory
  project_memory
  organization_memory
  memory_candidate
  memory_acl
  memory_version
  memory_audit_log
        |
        | 授权查询 / 增量同步 / 候选提交 / 审计记录
        v
LobsterAI 客户端
  个人记忆本地编辑
  项目记忆只读缓存
  组织记忆按需查询
  QMD collection 物化视图
  scoped_memory_search 工具代理
        |
        | 只给 OpenClaw 暴露当前会话允许看到的内容
        v
OpenClaw Runtime
  MEMORY.md
  memory/YYYY-MM-DD.md
  只读 PROJECT_MEMORY.md 缓存
  只读 QMD collections
  scoped_memory_search
```

**设计原则**：
- 个人记忆可以继续兼容 OpenClaw 文件体系；
- 项目记忆和组织记忆以企业后端为事实源；
- 客户端上的 Markdown/QMD collection 只是物化缓存，不是真源；
- Agent 不能直接写项目/组织 active memory，只能提交候选或修改请求；
- 后端负责 ACL、审批、版本、审计、密级和过期策略；
- OpenClaw SQLite 和 QMD index 只是本地检索索引；
- 组织记忆默认不应全量同步到客户端，更推荐后端检索后返回受控结果。

---

## 十一、项目记忆生命周期

项目记忆应该这样落地：后端是事实源，客户端只同步当前项目允许看到的只读视图。

### 11.1 写入

用户或 Agent 在项目会话中发现一条值得长期保留的信息：

> 当前项目 Windows 打包依赖 PortableGit，发版前必须验证安装包签名。

**不要直接让 Agent 写到项目共享文件**。客户端应向企业后端提交候选：

```text
project_memory_candidate
  project_id: lobsterai
  content: 当前项目 Windows 打包依赖 PortableGit，发版前必须验证安装包签名。
  evidence: 当前会话链接 / PR / 文档
  proposed_by: agent
  status: pending
```

### 11.2 确认

项目成员在 LobsterAI UI 中确认、编辑或拒绝，但确认动作落到后端。

确认后进入结构化项目记忆：

```text
project_memory
  id
  project_id
  content
  source_uri
  owner
  status = active
  version = 1
  updated_at
```

### 11.3 客户端同步

客户端按当前登录用户、项目、仓库和会话模式请求同步：

```text
GET /api/memory/project-sync?projectId=lobsterai&sinceVersion=12
Authorization: Bearer <token>
```

后端返回：

```text
project_id
version
items[]
deleted_item_ids[]
expires_at
```

> 客户端只缓存当前用户有权读取的项目记忆。缓存必须带版本号、TTL 和项目绑定信息。

### 11.4 物化为 OpenClaw 可用材料

客户端可以把后端返回的项目记忆物化成只读 Markdown：

```text
.lobster-memory/projects/lobsterai/PROJECT_MEMORY.md
```

**内容示例**：

```markdown
# Project Memory: LobsterAI
## Release
- Windows 打包依赖 PortableGit，发版前必须验证安装包签名。
  Source: PR-123 / release-checklist
  Owner: platform-team
```

也可以生成 QMD collection：

```text
collection: project-lobsterai
path: .lobster-memory/projects/lobsterai
pattern: **/*.md
```

> 💡 这一步只是**检索物化**，不是项目记忆的真实存储。

### 11.5 本地索引

LobsterAI 将该目录加入 OpenClaw `memorySearch.extraPaths`。

OpenClaw 下次同步时：

```text
扫描 PROJECT_MEMORY.md
  -> chunkMarkdown()
  -> 写 files
  -> 写 chunks
  -> 写 chunks_fts
  -> 可选写 chunks_vec
```

如果使用 QMD，则由 QMD 索引该只读项目 collection。

### 11.6 召回

项目 Cowork 中用户问：

> Windows 包发版前要检查什么？

OpenClaw 召回 `PROJECT_MEMORY.md` 中的 release chunk。Agent 回答时带上来源。

### 11.7 变更

如果项目规则变化，不直接改 OpenClaw SQLite，也不直接改本地物化文件。应提交到后端：

```text
POST /api/memory/project-candidates
  project_id
  content
  evidence
  operation: create | update | archive
```

后端确认后生成新版本，客户端再增量同步并重新物化。OpenClaw 只是重新索引最新物化结果。

---

## 十二、组织记忆生命周期

组织记忆必须更严格：后端是唯一事实源，客户端默认不做全量同步，只做按需检索或小范围只读缓存。

### 12.1 来源

组织记忆通常来自：
- 公司制度；
- 部门规范；
- 飞书或 Confluence 文档；
- 发布流程；
- 安全合规要求；
- 运维 runbook；
- HR / 行政 FAQ。

### 12.2 服务端入库

组织知识进入企业记忆服务：

```text
organization_memory
  scope: engineering
  title: 生产发版流程
  content: 所有生产发版必须先创建变更单并通过审批...
  source_uri: https://docs.example.com/release-process
  owner: release-management-team
  classification: internal
  status: active
  version: 3
  effective_at: 2026-05-01
```

### 12.3 权限

同时建立 ACL：

```text
memory_acl
  memory_id: org-release-process
  subject_type: department
  subject_id: engineering
  permission: read
```

### 12.4 客户端不应全量同步组织记忆

组织记忆不建议像项目记忆那样完整物化到客户端。原因：

- 组织知识规模大，容易污染 prompt；
- 组织知识通常有部门、角色、密级限制；
- 本地副本过期后风险高；
- 本地文件难以做审计和撤回；
- 离线缓存可能造成敏感信息扩散。

**更推荐的方式**：

```text
客户端只发送 query + 当前会话上下文
  -> 后端做 ACL / classification / status / version 过滤
  -> 后端检索
  -> 返回少量可引用结果
  -> 客户端交给 OpenClaw 使用
```

### 12.5 检索

用户提问时，LobsterAI 先根据登录用户和当前会话算出允许范围。

```text
user = 张三
department = engineering
project = lobsterai

allowed scopes:
  personal: 张三
  project: lobsterai
  organization: engineering internal docs
```

**推荐产品化接口**：

```text
POST /api/memory/search
  query
  scopes:
    personal: true
    projectIds: ["lobsterai"]
    organizationScopes: ["engineering"]
  context:
    workspace
    repository
    sessionType
```

后端返回：

```text
results[]
  content
  source_uri
  scope_type
  scope_id
  classification
  version
  effective_at
  citation_required
```

客户端再把结果作为 `scoped_memory_search` 的 tool result 或受控 context block 提供给 OpenClaw。

> 💡 MVP 可以小范围物化低风险组织记忆为只读 Markdown/QMD collection，但必须带 TTL、版本号、密级限制和清理机制。

### 12.6 回答

组织记忆参与回答时必须带来源：

> 根据《生产发版流程》v3，生产发版前必须先创建变更单并通过审批。
> 来源：https://docs.example.com/release-process

### 12.7 更新和过期

组织记忆变化时：

```text
制度文档更新
  -> ingestion pipeline 重新解析
  -> 生成新 version
  -> 旧 version archived
  -> 更新索引或 collection
  -> 后续搜索只返回 active 且有权限版本
```

> ⚠️ 这部分不能依赖 OpenClaw 的本地 SQLite 完成。SQLite 只知道当前被索引 chunk，不知道组织制度的审批状态和版本历史。

---

## 十三、是否需要改 OpenClaw 源码

短期不一定需要。

### 13.1 不改源码的可行方案

可以先做：

- ✅ 个人记忆继续使用 `MEMORY.md`；
- ✅ 后端增加项目记忆结构化存储；
- ✅ 客户端同步当前项目只读记忆并物化 Markdown；
- ✅ 配置 `memorySearch.extraPaths` 或 QMD collection 作为本地检索缓存；
- ✅ 组织记忆先走后端 `scoped_memory_search`，不做全量同步；
- ✅ LobsterAI 自己在 prompt 前注入后端返回的受控组织记忆摘要；
- ✅ 项目/组织记忆修改都走候选提交和后端审批。

> 🎯 **这个阶段的目标是验证价值，不是把 OpenClaw 变成企业知识平台。**

### 13.2 需要扩展的点

正式产品化后，建议扩展：

- `scoped_memory_search`；
- `scoped_memory_get`；
- active-memory scope policy；
- 搜索结果 metadata：scope、classification、sourceUri、version；
- 组织记忆强制 citation；
- 记忆召回审计；
- promotion 写入 candidate，而不是直接写 active memory。

> 这些能力可以在 LobsterAI 层实现，也可以部分贡献给 OpenClaw。但**公司组织架构、权限、审批流不应该塞进 OpenClaw 核心**。

---

## 十四、推荐落地路线

### Phase 1：把个人记忆讲清楚、管清楚

- 设置页区分 `MEMORY.md` 和每日记忆；
- 展示记忆来源和更新时间；
- 删除和编辑产生审计；
- 明确提示：SQLite 是索引，不是事实源。

### Phase 2：项目记忆 MVP

- 后端增加 project memory 和 candidate 表；
- 客户端增加项目识别；
- Agent 或用户提交 project memory candidate 到后端；
- 项目成员在服务端确认后 active；
- 客户端增量同步当前项目只读记忆；
- 生成只读 `PROJECT_MEMORY.md` 或 QMD collection；
- 在项目 Cowork 中召回。

### Phase 3：组织记忆只读试点

- 选择低风险知识域，如发版流程、研发规范；
- 后端建立组织记忆结构化表；
- 接入 ACL、密级、版本和审计；
- 客户端不全量同步，默认调用后端 `scoped_memory_search`；
- 仅对低风险资料允许短 TTL 只读缓存或 QMD collection；
- 回答强制 citation；
- 禁止 Agent 写组织 active memory。

### Phase 4：scope-aware active-memory

- 会话启动时计算 scope policy；
- active-memory 只召回允许范围；
- 组织记忆召回带 classification；
- confidential 内容默认不注入 prompt；
- 所有组织召回写审计。

### Phase 5：平台级 scoped memory

- 实现 `scoped_memory_search`；
- 后端统一 hybrid search、ACL、审计、版本；
- OpenClaw 只消费工具结果；
- `MEMORY.md` 保留为个人和本地工作区记忆。

---

## 十五、最终建议

OpenClaw 的记忆机制可以作为个人助手和项目 Cowork 的底座，但不要把它误用成组织知识库。

### 更稳妥的定位

```text
OpenClaw:
  负责 Agent runtime、文件记忆、索引、召回工具、active-memory、短期提升。

LobsterAI / 企业后端:
  负责项目/组织记忆的事实源、权限、审批、版本、审计和服务端检索。

LobsterAI 客户端:
  负责个人记忆编辑、项目记忆同步、组织记忆查询代理和本地只读缓存。
```

### 分层建议

| 场景 | 方案 |
|------|------|
| **只做个人记忆** | 改 LobsterAI 的 `MEMORY.md` 管理和 OpenClaw 配置就够了 |
| **做项目记忆** | 可以先不改 OpenClaw，用后端结构化存储 + 客户端只读同步 + Markdown/QMD 物化 + `extraPaths` 验证 |
| **做组织记忆** | 必须引入企业级 memory service，并把 ACL、citation、版本、审计和检索尽量放在服务端 |

> 📌 **一句话总结**：OpenClaw SQLite 和 QMD index 里的内容都是**客户端检索视图**，不是企业记忆本身；项目记忆和组织记忆的真源必须在企业后端，由客户端受控同步或发起修改请求，再把允许的结果交给 OpenClaw 召回。
