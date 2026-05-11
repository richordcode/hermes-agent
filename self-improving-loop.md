# 自进循环（Self-Improving Loop）

Hermes 实现了一套**三层自进系统**，能把每次对话都转化为持久的学习成果。Agent 不只是回答问题 —— 它会将成功的技术捕捉为可复用技能，提取用户偏好存入记忆，并定期整理技能库，保持其精简且易于查找。

三层按不同时间尺度运作：

| 层 | 触发条件 | 时间尺度 | 作用 |
|------|---------|------------|--------------|
| **后台审查** | 每 N 轮用户对话 | 每轮对话结束数秒后 | 从对话中提取记忆 + 技能更新 |
| **技能系统** | Agent 工具调用 | 对话过程中 | 以工序记忆的形式创建/修补技能 |
| **策展器（Curator）** | 每周（可配置） | 数天到数周 | 整理、修剪和维护技能库 |

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HERMES 自进循环                                    │
│                                                                     │
│  ┌──────────┐     ┌──────────────┐     ┌──────────────────┐        │
│  │  用户     │────▶│  AIAgent     │────▶│  后台审查分支     │        │
│  │  消息     │     │  主循环       │     │   (每 N 轮)      │        │
│  └──────────┘     └──────────────┘     │                    │        │
│                                         │  提取:             │        │
│                                         │  • 偏好 → MEMORY.md│        │
│                                         │  • 技法 →          │        │
│                                         │    skill_manage    │        │
│                                         └───────┬────────────┘        │
│                                                 │                    │
│                                          skill_manage()              │
│                                                 │                    │
│                                                 ▼                    │
│                              ┌──────────────────────────┐           │
│                              │  ~/.hermes/skills/       │           │
│                              │  • SKILL.md              │           │
│                              │  • references/           │           │
│                              │  • templates/            │           │
│                              │  • scripts/              │           │
│                              │                          │           │
│                              │  ~/.hermes/skills/       │           │
│                              │    .usage.json (附属文件) │           │
│                              └──────────┬───────────────┘           │
│                                         │                           │
│                                  每 7 天                            │
│                                         │                           │
│                                         ▼                           │
│                              ┌──────────────────────────┐           │
│                              │  策展器 (Curator)         │           │
│                              │  • 自动状态迁移           │           │
│                              │  • LLM 伞式合并           │           │
│                              │  • 归档过期技能 →         │           │
│                              │    .archive/              │           │
│                              │  • Cron 任务引用重写      │           │
│                              │  • 运行前 tar.gz 快照     │           │
│                              └──────────────────────────┘           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 第一层：后台审查（会话内学习）

**源码：** `run_agent.py` → `_spawn_background_review()`

每 `nudge_interval` 轮用户输入（默认 10 轮）之后，Agent 会 fork 一个自身的后台副本，用于审查对话。此过程以守护线程形式运行 —— 用户永远无需等待。

### 触发逻辑

```python
# run_agent.py — 记忆触发器（按对话轮次）
if (self._memory_nudge_interval > 0
        and self._turns_since_memory >= self._memory_nudge_interval):
    _should_review_memory = True

# run_agent.py — 技能触发器（按迭代次数）
if (self._skill_nudge_interval > 0
        and self._iters_since_skill >= self._skill_nudge_interval
        and "skill_manage" in self.valid_tool_names):
    _should_review_skills = True

# run_agent.py — 任一触发则启动
if _should_review_memory or _should_review_skills:
    self._spawn_background_review(
        messages_snapshot=list(messages),
        review_memory=_should_review_memory,
        review_skills=_should_review_skills,
    )
```

审查在**向用户返回响应之后**才开始，因此不会在活跃任务期间与模型争抢注意力。

### Fork 的配置

```python
# run_agent.py
review_agent = AIAgent(
    model=self.model,            # 继承主会话的模型
    max_iterations=16,           # 小预算 — 这是审查，不是任务
    quiet_mode=True,             # 无旋转动画，无进度输出
    enabled_toolsets=["memory", "skills"],  # 仅这两个工具集
    ...
)
review_agent._memory_write_origin = "background_review"
review_agent._memory_nudge_interval = 0    # 不再触发递归审查
review_agent._skill_nudge_interval = 0
review_agent.suppress_status_output = True
```

Fork 继承父级 Agent 的 provider、模型、API 凭证和记忆存储。它直接写入共享的记忆 / 技能存储 —— 变更立即对下一轮主会话可用。

### 审查提示词

根据触发条件选择三种提示词变体：

| 提示词常量 | 何时使用 | 关注点 |
|-----------------|-----------|-------|
| `_COMBINED_REVIEW_PROMPT` | 记忆 + 技能触发器同时触发 | 全面审查用户身份 + 技术 |
| `_MEMORY_REVIEW_PROMPT` | 仅记忆触发器 | 提取用户偏好、人格、期望 |
| `_SKILL_REVIEW_PROMPT` | 仅技能触发器 | 识别可复用技术 → 创建/修补技能 |

**对审查 Agent 的关键指令：**

- **记忆：** 保存关于用户的事实和持久偏好。"记忆用于记录用户是谁，以及当前的情形和运作状态。"
- **技能：** 捕捉"如何完成这类任务"。四层优先级：(1) 修补当前已加载技能、(2) 修补已有伞式技能、(3) 在已有伞式技能下新增支撑文件、(4) 创建新的类级别伞式技能。
- **禁止捕捉：** 环境相关故障、对工具的负面断言、特定会话的临时错误、一次性任务叙事。

### 用户可见输出

```python
# run_agent.py
self._safe_print("  💾 自进式审查：创建技能 'python-packaging' · "
                 "更新记忆偏好")
```

摘要只有一行，列出具体执行的操作。审查失败时记入 WARNING 级别日志，绝不暴露给用户。

---

## 第二层：技能系统（工序记忆）

**源码：** `tools/skill_manager_tool.py` + `tools/skills_tool.py`

技能是 Hermes 的**工序记忆** —— 针对特定任务类型的、可执行的精简文档。遵循 Anthropic 的 Claude Skills 格式，采用渐进式展开：先是元数据（名称 + 描述），按需展示完整说明，必要时引用关联文件。

### 技能存储

```
~/.hermes/skills/
├── python-packaging/
│   ├── SKILL.md              # YAML frontmatter + Markdown 正文
│   ├── references/
│   │   └── pyproject.toml-examples.md
│   ├── templates/
│   │   └── setup.cfg
│   ├── scripts/
│   │   └── verify-build.sh
│   └── assets/
│       └── logo.png
├── github-pr-review/
│   └── SKILL.md
└── .usage.json               # 遥测附属文件
```

### 工具接口：`skill_manage`

六个操作，全部通过单工具调用：

```python
# tools/skill_manager_tool.py
def skill_manage(action, name, ...):
    if action == "create":
        # 完整 SKILL.md 写入 + 目录脚手架
    elif action == "patch":
        # 精准查找并替换（使用模糊匹配引擎）
    elif action == "edit":
        # 整个 SKILL.md 重写（仅重大整体改写时用）
    elif action == "delete":
        # 删除技能目录（合并时必须传 absorbed_into）
    elif action == "write_file":
        # 向 references/、templates/、scripts/ 或 assets/ 添加文件
    elif action == "remove_file":
        # 删除支撑文件
```

### 来源追踪

并非所有技能都能由策展器管理。系统区分四种来源：

```python
# tools/skill_usage.py
def is_agent_created(skill_name):
    """既不是内置技能，也不是中心安装技能。"""
    off_limits = _read_bundled_manifest_names() | _read_hub_installed_names()
    return skill_name not in off_limits

def _is_curator_managed_record(record):
    """仅限后台审查分支显式标记的技能。"""
    return record.get("created_by") == "agent"
```

`created_by: "agent"` 标记**仅在** `skill_manage(action="create")` 在后台审查分支内调用时设置 —— 绝不来自用户发起的工具调用。这使策展器不会触碰用户手动创建的技能。

### 使用遥测

每次技能交互都会更新 `~/.hermes/skills/.usage.json` 中的计数器：

```python
# tools/skill_usage.py
def bump_view(skill_name):    # 调用了 skill_view()
    rec["view_count"] += 1
    rec["last_viewed_at"] = now()

def bump_use(skill_name):     # 技能被加载到提示词中
    rec["use_count"] += 1
    rec["last_used_at"] = now()

def bump_patch(skill_name):   # skill_manage(patch/edit/write_file)
    rec["patch_count"] += 1
    rec["last_patched_at"] = now()
```

这些时间戳是策展器自动状态迁移逻辑的**唯一输入**。一个上周查看过但从未使用或修补过的技能，没有近期的"活跃记录"——策展器最终会将其标记为 stale。

---

## 第三层：策展器（后台维护）

**源码：** `agent/curator.py`（约 1782 行）+ `agent/curator_backup.py` + `tools/skill_usage.py`

策展器是**质量控制层**。它防止技能库退化成一份扁平的长列表，里面装满了数百个狭窄、近重复的"一个会话一个技能"条目。

:::info 设计理念
技能集合的目标是一份**按类编排的说明和经验知识库**。收录几百个狭窄技能，其中每个仅记录了某个会话的特定 bug，是失败的 —— 不是特色。策展器负责构建伞式技能。
:::

### 如何触发

策展器**不是 cron 守护进程**。它附着在已有基础设施上：

| 入口 | 代码位置 | 行为 |
|-------------|---------------|----------|
| **Gateway** | `gateway/run.py` | 每 `CURATOR_EVERY` 次 cron tick 检查 |
| **CLI** | `cli.py` | 会话启动时检查 |
| **手动** | `hermes curator run` | 立即执行，绕过所有关卡 |

所有路径经过同一关卡：

```python
# agent/curator.py
def should_run_now(now=None):
    if not is_enabled():           # config.yaml 中 curator.enabled
        return False
    if is_paused():                # hermes curator pause
        return False
    if last_run_at is None:        # 首次观测 → 播种 + 推迟
        seed_state()
        return False
    return (now - last_run_at) >= interval_hours  # 默认：7 天
```

**首次运行行为：** 全新安装时，策展器将 `last_run_at` 播种为"当前"，并将首次实际执行推迟一个完整周期。这给用户留出时间审查技能库、钉住重要技能，或彻底退出。

### 两阶段执行

每次执行包含两个独立阶段：

#### 第一阶段：自动迁移（确定性，无需 LLM）

```python
# agent/curator.py: apply_automatic_transitions()
for row in agent_created_report():
    if row.pinned:                        # 已钉住 → 完全跳过
        continue

    anchor = row.last_activity_at or row.created_at
    if anchor <= archive_cutoff:          # 默认：90 天
        archive_skill(name)               # → .archive/
    elif anchor <= stale_cutoff:          # 默认：30 天
        set_state(name, STATE_STALE)
    elif anchor > stale_cutoff and row.state == STATE_STALE:
        set_state(name, STATE_ACTIVE)     # 重新激活
```

**生命周期状态机：**

```
active ──(30 天无活跃)──▶ stale ──(90 天无活跃)──▶ archived
  ▲                                │
  └──────(检测到新活跃)────────────┘

pinned: 完全免于所有自动状态迁移
```

#### 第二阶段：LLM 审查（伞式合并）

策展器以与后台审查相同的模式 fork 一个 `AIAgent` 实例：

```python
# agent/curator.py: _run_llm_review()
review_agent = AIAgent(
    model=_model_name,
    max_iterations=9999,         # 大技能库容忍度较高
    quiet_mode=True,
    platform="curator",
    skip_context_files=True,
    skip_memory=True,            # 策展期间不做记忆操作
)
review_agent._memory_nudge_interval = 0
review_agent._skill_nudge_interval = 0
```

**审查提示词**（`CURATOR_REVIEW_PROMPT`）要求 fork 完成：

1. **扫描候选技能** — 所有标记为 Agent 创建且附带使用统计的技能
2. **识别前缀聚类** — 共享领域关键词的技能（例如 `hermes-config-*`、`gateway-*`、`ollama-*`）
3. **采用三种策略合并：**

| 策略 | 何时使用 | 操作 |
|----------|------|--------|
| 并入已有伞式技能 | 聚簇中某个技能范围已足够广 | 修补伞式技能，将兄弟技能独特洞察作为注明段落加入，然后 `skill_manage(delete, absorbed_into=<umbrella>)` |
| 创建新伞式技能 | 聚簇中没有足够宽的现有成员 | `skill_manage(create)` 生成类级别覆盖，将窄兄弟技能归档 |
| 降级为支撑文件 | 兄弟技能中有窄但有价值的会话细节 | 移入伞式技能的 `references/`、`templates/` 或 `scripts/`，然后归档兄弟技能 |

**提示词强制执行的硬不变性：**

- 绝不触碰内置或中心安装技能（已在前面过滤）
- 绝不删除 —— 最大破坏性操作是归档（可恢复）
- 绝不触碰已钉住技能
- 使用计数器**不是**跳过合并的理由 —— 根据内容重叠程度判断

### 通过 `absorbed_into` 分类

策展器归档技能时必须声明意图。`skill_manage(action="delete")` 上的 `absorbed_into` 参数是有权威的信号：

```python
# tools/skill_manager_tool.py: _delete_skill()
def _delete_skill(name, absorbed_into=None):
    if absorbed_into is None:
        # 遗留情况 — 接受，但记录为"模糊"（ambiguous）
    elif absorbed_into == "":
        # 显式：真正修剪，无转发目标
    else:
        # 显式：内容已吸入伞式技能 — 文件必须存在于磁盘上
        target = _find_skill(absorbed_into)
        if not target:
            return error("伞式技能不存在")
```

策展器的分类调节器区分两种结果：

- **合并** — 内容仍以不同名称存在。被吸收兄弟技能的目录移入 `.archive/` 作为安全保留，但知识在伞式技能中持续存在。
- **修剪** — 因过时 / 无关而归档，内容未在别处留存。

这种区分驱动下游行为，如 cron 任务技能引用重写。

### Cron 任务引用重写

当某个 cron 任务引用了技能 `X`，而策展器将 `X` 并入伞式技能 `Y` 时，cron 任务的 `skills` 数组会原地更新：

```python
# agent/curator.py: _write_run_report() → cron.jobs.rewrite_skill_refs()
cron_rewrites = _rewrite_cron_refs(
    consolidated={"旧技能": "伞式技能"},
    pruned=["确实过时技能"],
)
```

这确保计划任务在合并过程中保持运行，无需用户干预。

### 备份与回滚

每次有变更的执行之前，策展器会进行快照：

```python
# agent/curator.py: run_curator_review()
from agent import curator_backup
snap = curator_backup.snapshot_skills(reason="pre-curator-run")
```

**备份内容**（tar.gz，位于 `~/.hermes/skills/.curator_backups/<时间戳>/`）：
- 全部 `SKILL.md` 文件 + 目录
- `.usage.json`（使用遥测）
- `.archive/`（先前归档的技能）
- `.curator_state`（上次运行指针）
- `.bundled_manifest`（保护标记）
- `cron/jobs.json`（回滚时恢复重写前的状态）

**排除内容：**
- `.curator_backups/` 自身（会无限递归）
- `.hub/`（由技能中心管理）

回滚命令：`hermes curator rollback <快照>`。连回滚操作本身都支持撤销 —— 恢复前会先对当前状态做快照。

### 每次运行的报告

每次策展器运行生成报告，位于 `~/.hermes/logs/curator/{YYYYMMDD-HHMMSS}/`：

| 文件 | 内容 |
|------|---------|
| `run.json` | 机器可读：完整计数、工具调用日志、分类结果、cron 重写 |
| `REPORT.md` | 人类可读：自动迁移摘要、合并方案与说明、修剪、新技能、状态变更、还原指南 |
| `cron_rewrites.json` | 仅当 cron 任务被触及时生成 |

用户可通过 `hermes curator status` 查看简要摘要，或阅读完整报告了解细节。

---

## 配置

三层均可独立配置，写入 `~/.hermes/config.yaml`：

```yaml
# ── 后台审查 ──
skills:
  creation_nudge_interval: 10     # 技能审查间隔（每 N 轮对话），0 = 关闭
  guard_agent_created: false      # 对 agent 创建的技能进行安全扫描

memory:
  nudge_interval: 10              # 记忆审查间隔（每 N 轮对话），0 = 关闭

# ── 策展器 ──
curator:
  enabled: true                   # 总开关
  interval_hours: 168             # 两次运行间隔 7 天
  min_idle_hours: 2               # Agent 必须空闲至少这么久
  stale_after_days: 30            # 无活跃超此时长标记 stale
  archive_after_days: 90          # 仍无活跃超此时长则归档
  backup:
    keep: 5                       # 保留的快照数量
```

---

## 设计原则

以下原则贯穿所有三层：

### 1. 非阻塞

后台审查和策展器均以守护线程方式运行。用户活跃对话永远无需等待自进任务。如果审查 fork 崩溃，错误会被 DEBUG/WARNING 级别的日志记录并静默丢弃。

### 2. 默认安全

三层中最大的破坏性操作是**归档**（将技能移入 `.archive/`）。真正的删除永远不会自动化。归档可通过 `hermes curator restore <名称>` 恢复，或手动将目录移回。

### 3. 来源隔离

策展器的操作范围严格受来源限制：
- **内置技能**（随仓库一同发布） → 在 `.bundled_manifest` 中追踪，永不修改
- **中心安装技能**（来自 [agentskills.io](https://agentskills.io)） → 在 `.hub/lock.json` 中追踪，永不修改
- **用户创建技能**（手动 `skill_manage`） → 未标记 `agent_created`，永不被自动策展
- **Agent 创建技能**（来自后台审查 fork） → `created_by: "agent"`，完全参与策展

### 4. 快照保护

每次策展器运行前都会进行 tar.gz 快照。回滚始终可行，包括对回滚本身的再回滚。

### 5. 提示缓存友好

后台审查 fork 和策展器 fork 各自运行在独立的 `AIAgent` 实例中，携带各自的提示缓存。它们绝不对主会话的对话历史、消息或系统提示做任何修改 —— 从而保持 Anthropic 前缀缓存在多轮对话中有效。

### 6. 防止负面反馈固化

所有三层审查提示词都明确禁止捕捉：
- 环境相关故障（缺少二进制、未安装软件包）
- 对工具的负面断言（"某个工具坏了"）
- 特定会话的临时错误
- 一次性任务叙事

这些内容一旦固化，Agent 会在之后很长时间内反复引用这些早已修复的问题来自我设限。

---

## 关联文档

- [策展器用户指南](/docs/user-guide/features/curator.md) — 面向最终用户的文档
- [技能](/docs/user-guide/features/skills.md) — 从用户视角看技能系统
- [Agent 循环](/docs/developer-guide/agent-loop.md) — 承载审查 fork 的主对话循环
- [创建技能](/docs/developer-guide/creating-skills.md) — 创作内置技能
- [记忆 Provider 插件](/docs/developer-guide/memory-provider-plugin.md) — 可插拔记忆后端
