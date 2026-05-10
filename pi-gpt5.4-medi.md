# Hermes-Agent 记忆系统源码解读

## 总体结构

结合源码，`hermes-agent` 的“记忆系统”并不是单一模块，而是三层并存：

1. **内建持久记忆**：`MEMORY.md` + `USER.md`
2. **会话历史召回**：`session_search` + SQLite FTS5
3. **外部记忆提供者插件**：Honcho / Mem0 / Hindsight / Supermemory 等

设计目标是：

- 让关键事实常驻上下文
- 让海量历史按需检索
- 保持 prompt 稳定，尽量不破坏 prompt caching

---

## 一、内建持久记忆：`MEMORY.md` / `USER.md`

### 1. 配置入口

在 `hermes_cli/config.py` 中：

```python
"memory": {
    "memory_enabled": True,
    "user_profile_enabled": True,
    "memory_char_limit": 2200,
    "user_char_limit": 1375,
    "provider": "",
}
```

默认开启两类内建记忆：

- `MEMORY.md`：环境、项目、经验、约定
- `USER.md`：用户偏好、身份、沟通风格

### 2. 存储位置

在 `tools/memory_tool.py`：

```python
def get_memory_dir() -> Path:
    return get_hermes_home() / "memories"
```

所以记忆目录实际是：

- `get_hermes_home()/memories/MEMORY.md`
- `get_hermes_home()/memories/USER.md`

这意味着它天然支持 **profile 隔离**，不会写死到 `~/.hermes`。

---

## 二、内建记忆的数据结构与文件格式

核心类是 `tools/memory_tool.py` 里的 `MemoryStore`。

### 1. 维护两份状态

```python
self.memory_entries: List[str] = []
self.user_entries: List[str] = []
self._system_prompt_snapshot = {"memory": "", "user": ""}
```

它同时维护：

- `memory_entries / user_entries`：**实时状态**
- `_system_prompt_snapshot`：**启动时冻结的 system prompt 快照**

### 2. 文件格式

条目分隔符是：

```python
ENTRY_DELIMITER = "\n§\n"
```

也就是说虽然叫 `.md`，本质上是简单文本条目列表，不是复杂 Markdown 结构。

### 3. 容量控制

不是按 token，而是按字符数：

- `MEMORY.md` 默认 2200 chars
- `USER.md` 默认 1375 chars

这样做是为了模型无关、成本稳定。

---

## 三、内建记忆如何注入 prompt

### 1. Agent 初始化时加载

在 `run_agent.py` 初始化阶段：

```python
self._memory_store = MemoryStore(...)
self._memory_store.load_from_disk()
```

`load_from_disk()` 会：

1. 读取 `MEMORY.md` / `USER.md`
2. 去重
3. 渲染成块文本
4. 存入 `_system_prompt_snapshot`

### 2. 构建 system prompt 时注入

`run_agent.py` 中：

```python
if self._memory_store:
    if self._memory_enabled:
        mem_block = self._memory_store.format_for_system_prompt("memory")
    if self._user_profile_enabled:
        user_block = self._memory_store.format_for_system_prompt("user")
```

而 `format_for_system_prompt()` 返回的是冻结快照，不是实时状态。

### 3. 冻结快照模式

这是 Hermes 的关键设计：

- 会话启动时把记忆读进 system prompt
- 会话中途即使调用 `memory` 工具写入新记忆，也**只会落盘**
- **当前 session 的 system prompt 不会更新**
- 新记忆要到 **下一个 session** 才会进入 prompt

这样做的目的是：

> 保持 system prompt 前缀稳定，保护 prompt caching / prefix cache。

---

## 四、模型如何写记忆：`memory` 工具

### 1. 实际支持的动作

`tools/memory_tool.py -> memory_tool()` 实际支持：

- `add`
- `replace`
- `remove`

虽然顶部注释提到 `read`，但真实 dispatcher 和 schema 都没有 `read`。

### 2. 为什么没有 `read`

因为 Hermes 认为内建记忆已经自动进入 system prompt，模型天然“看得见”，所以工具只负责修改。

### 3. replace/remove 的定位方式

用的是子串匹配：

```python
matches = [(i, e) for i, e in enumerate(entries) if old_text in e]
```

所以模型不需要维护 entry id，只要给一个足够唯一的片段即可。

如果匹配到多个不同条目，会报错要求更具体。

---

## 五、安全设计：防 prompt injection / secrets exfiltration

在 `tools/memory_tool.py` 的 `_scan_memory_content()` 中，写入前会扫描威胁模式。

会拦截的内容包括：

- prompt injection（如 `ignore previous instructions`）
- 身份劫持（如 `you are now ...`）
- secrets exfiltration（如 `curl ... $API_KEY`、`cat ~/.env`）
- SSH / 持久化后门（如 `authorized_keys`）
- 不可见 Unicode（零宽字符等）

原因很简单：这些内容最终会注入到 system prompt，所以必须做防护。

---

## 六、并发写安全：锁与原子替换

### 1. 写前先加锁、再重读

每次 `add/replace/remove` 都会：

```python
with self._file_lock(self._path_for(target)):
    self._reload_target(target)
```

作用：

- 多 session 并发修改时不互相覆盖
- 始终基于磁盘最新状态修改

### 2. 原子写入

```python
fd, tmp_path = tempfile.mkstemp(...)
...
os.replace(tmp_path, str(path))
```

好处：

- 读者不会读到截断中的半成品
- 要么看到旧文件，要么看到新文件

### 3. 跨平台锁

- Unix 用 `fcntl`
- Windows 用 `msvcrt`

---

## 七、自动补记机制：不是只靠模型“自觉记住”

Hermes 还有两套自动保存机制，位于 `run_agent.py`。

### 1. 周期性后台 review

相关状态：

```python
self._memory_nudge_interval = 10
self._turns_since_memory = 0
```

每经过一定轮数，会触发后台 review。流程是：

1. 当前回复先正常完成
2. Hermes fork 一个后台 `AIAgent`
3. 给它一段 review prompt：检查这段对话里有没有值得保存的用户偏好、个性、习惯、要求
4. 如果有，就调用 `memory` 工具写入

这样可以让“记忆整理”不干扰主对话。

### 2. flush_memories：压缩/退出前最后记一笔

`run_agent.py -> flush_memories()` 会在这些时机触发：

- context compression 前
- reset 前
- CLI 退出前
- gateway session 结束前

做法是：

1. 暂时往消息里插入一条系统提示
2. 只开放 `memory` 工具
3. 额外跑一次模型调用
4. 如果模型决定保存记忆，就直接执行
5. 最后把这次 flush 的痕迹从消息历史中移除

本质上是：

> 在上下文即将丢失前，再给模型一次整理长期记忆的机会。

---

## 八、第二层记忆：`session_search` 其实是“会话历史召回”

这不是 `MEMORY.md` 的一部分，而是另一条线。

相关文件：

- `tools/session_search_tool.py`
- `hermes_state.py`

### 1. 所有会话进入 SQLite

在 `hermes_state.py` 中：

```python
DEFAULT_DB_PATH = get_hermes_home() / "state.db"
```

数据库包括：

- `sessions`
- `messages`
- `messages_fts`

其中 `messages_fts` 是 FTS5 全文搜索表。

### 2. 搜索流程

`session_search` 的流程大致是：

1. 用 FTS5 搜索消息
2. 以 session 分组
3. 取最相关的几个 session
4. 截出对话片段
5. 交给辅助模型总结
6. 返回摘要而不是整段 transcript

所以它更像：

> “按需回忆过去聊过什么”

而不是“始终加载在 prompt 里的长期记忆”。

### 3. 与 built-in memory 的分工

- `MEMORY.md` / `USER.md`：小而精、稳定、必须常驻
- `session_search`：海量历史、细节、以前具体做过什么

---

## 九、第三层：外部记忆 provider 插件

相关文件：

- `agent/memory_provider.py`
- `agent/memory_manager.py`
- `plugins/memory/__init__.py`
- `plugins/memory/*`

### 1. 启用方式

在配置里设置：

```yaml
memory:
  provider: honcho
```

然后 `run_agent.py` 会调用：

```python
from plugins.memory import load_memory_provider
```

加载对应 provider。

### 2. 插件发现机制

`plugins/memory/__init__.py` 会扫描两类目录：

1. 内置：`plugins/memory/<name>/`
2. 用户安装：`$HERMES_HOME/plugins/<name>/`

### 3. provider 抽象接口

`agent/memory_provider.py` 定义了统一生命周期：

- `initialize()`
- `system_prompt_block()`
- `prefetch()`
- `queue_prefetch()`
- `sync_turn()`
- `get_tool_schemas()`
- `handle_tool_call()`
- `shutdown()`

这说明外部记忆系统不仅是“加工具”，而是完整的 recall + sync + lifecycle 体系。

---

## 十、外部 provider 如何融入主对话

### 1. 静态说明进入 system prompt

在 `run_agent.py`：

```python
_ext_mem_block = self._memory_manager.build_system_prompt()
```

这部分通常是 provider 的说明文字，比如：

- 当前启用了 Honcho
- 当前是 context / tools / hybrid 模式

### 2. 动态召回不进 system prompt，而是注入当前 user message

每轮开始前：

```python
_ext_prefetch_cache = self._memory_manager.prefetch_all(_query)
```

真正发请求时，会包装成 fenced block 注入本轮用户消息：

```python
_fenced = build_memory_context_block(_ext_prefetch_cache)
api_msg["content"] = _base + "\n\n" + _fenced
```

格式大概是：

```text
<memory-context>
[System note: The following is recalled memory context, NOT new user input...]
...
</memory-context>
```

这也是为了：

- 静态内容走 system prompt，利于缓存
- 动态 recall 不污染固定前缀

### 3. 每轮结束后同步给 provider

```python
self._memory_manager.sync_all(original_user_message, final_response)
self._memory_manager.queue_prefetch_all(original_user_message)
```

即：

- 已完成的对话写回 provider
- 并为下一轮 recall 做预取

---

## 十一、源码层面几个值得注意的细节

### 1. MemoryManager 注释与实际 wiring 有些偏差

`agent/memory_manager.py` 的注释提到 builtin provider 总是注册在 manager 里，但从 `run_agent.py` 实际逻辑看：

- 内建记忆走 `self._memory_store`
- 外部 provider 才走 `self._memory_manager`

所以当前实现并不是“所有记忆都统一抽象成 provider”。

### 2. built-in 写入只桥接 add/replace 给外部 provider

在 `run_agent.py`：

```python
if self._memory_manager and function_args.get("action") in ("add", "replace"):
    self._memory_manager.on_memory_write(...)
```

也就是说：

- `add` / `replace` 会通知外部 provider
- `remove` 不会

这意味着 built-in 和 external provider 的状态未必严格一致。

### 3. flush_memories 里也没有完全走 bridge

`flush_memories()` 直接调用内建 `_memory_tool(...)`，并没有统一经过 manager 路由，所以 flush 期间保存的 built-in memory 不一定同步到外部 provider。

### 4. 注释 / 文档有历史残留

例如顶部注释还提到 `read` action，但真实实现没有。说明这块经历过演进，注释和实现并非处处完全同步。

---

## 十二、整体调用链梳理

### 启动时

1. 读取 config
2. 初始化 `MemoryStore`
3. 从磁盘加载 `MEMORY.md` / `USER.md`
4. 生成冻结 snapshot
5. 如有配置，再初始化一个外部 memory provider

### 每轮开始前

1. `on_turn_start()` 通知 provider
2. provider `prefetch()` 回忆相关上下文
3. 回忆结果以 `<memory-context>` 形式注入本轮 user message

### 对话中

1. 模型可以调用 `memory` 工具
2. `MemoryStore` 修改 live entries
3. 立即落盘
4. 当前 session 的 system prompt 不更新

### 每轮结束后

1. provider `sync_turn()` 同步新对话
2. `queue_prefetch()` 为下一轮预取
3. 如达到阈值，后台 review 可能补写 memory

### 压缩 / 退出 / reset 前

1. `flush_memories()` 触发一次额外 memory-only 调用
2. 保存可能丢失的重要事实
3. 再压缩或结束会话

---

## 十三、评价

### 优点

- **prompt cache 友好**：冻结快照设计非常关键
- **记忆分层明确**：稳定事实、历史会话、外部语义记忆各司其职
- **持久化实现扎实**：锁 + 原子写入
- **扩展性好**：memory provider 生命周期设计完整
- **自动保存机制完善**：后台 review + flush 双保险

### 局限

- 内建 memory 仍偏朴素，没有语义检索能力
- built-in memory 与 external provider 没有完全统一
- `remove` 不桥接 external provider，可能导致状态漂移
- 少量注释 / 文档与实现存在历史偏差

---

## 总结

Hermes 的记忆系统本质上是一个 **分层长期记忆架构**：

- 用 `MEMORY.md` / `USER.md` 保存小而稳定的关键事实
- 用 `state.db + session_search` 保存和召回完整会话历史
- 用 memory provider 插件扩展更强的语义记忆能力

其中最核心的实现思想是：

> **记忆可以实时写盘，但 system prompt 中的记忆快照在一个 session 内保持冻结。**

这既保住了跨会话记忆能力，也兼顾了 prompt caching 的性能收益。
---

## 十四、从 `run_agent.py` 视角再看一遍主调用链

如果按 `AIAgent` 的真实执行路径来看，记忆系统大致沿着这条链路运转。

### 1. 初始化阶段

`AIAgent.__init__()` 先处理内建记忆，再处理外部 provider。

#### 内建记忆初始化

```python
self._memory_store = None
self._memory_enabled = False
self._user_profile_enabled = False
...
mem_config = _agent_cfg.get("memory", {})
self._memory_enabled = mem_config.get("memory_enabled", False)
self._user_profile_enabled = mem_config.get("user_profile_enabled", False)

if self._memory_enabled or self._user_profile_enabled:
    from tools.memory_tool import MemoryStore
    self._memory_store = MemoryStore(...)
    self._memory_store.load_from_disk()
```

这一步完成：

- 读取 memory 配置
- 创建 `MemoryStore`
- 从磁盘装载 `MEMORY.md` / `USER.md`
- 生成 frozen snapshot

#### 外部 provider 初始化

```python
_mem_provider_name = mem_config.get("provider", "")
...
from agent.memory_manager import MemoryManager
from plugins.memory import load_memory_provider
self._memory_manager = _MemoryManager()
_mp = _load_mem(_mem_provider_name)
if _mp and _mp.is_available():
    self._memory_manager.add_provider(_mp)
...
self._memory_manager.initialize_all(...)
```

这一步完成：

- 读取 `memory.provider`
- 动态加载插件
- 初始化 provider 的 session / 连接 / 缓存 / 后台线程

所以初始化阶段实际上是“双轨”：

- `_memory_store` 负责 built-in memory
- `_memory_manager` 负责 external provider

---

### 2. 构造 system prompt

真正组 system prompt 时，Hermes 会按顺序拼接：

1. 常规 system prompt
2. built-in memory snapshot
3. provider 的静态说明块

源码形态大致是：

```python
if self._memory_store:
    if self._memory_enabled:
        mem_block = self._memory_store.format_for_system_prompt("memory")
    if self._user_profile_enabled:
        user_block = self._memory_store.format_for_system_prompt("user")

if self._memory_manager:
    _ext_mem_block = self._memory_manager.build_system_prompt()
```

这里的边界很清楚：

- built-in memory 的内容是真的进入 system prompt 主体
- external provider 的 `system_prompt_block()` 更多是状态说明 / 使用说明
- 动态 recall 不在这里注入

---

### 3. 每轮开始前：通知 provider + 预取 recall

在 `run_conversation()` 开头，Hermes 会先通知 provider：

```python
self._memory_manager.on_turn_start(self._user_turn_count, _turn_msg)
```

这给 provider 一个 per-turn hook，可以做：

- cadence 计数
- 本轮策略判断
- 内部缓存刷新

接着做 recall 预取：

```python
_ext_prefetch_cache = self._memory_manager.prefetch_all(_query) or ""
```

也就是：

- 以当前用户消息为 query
- 从 provider 拿回本轮可能相关的记忆上下文

---

### 4. 真正发 API 请求时：把 recall 注入“当前轮 user message”

这一步不是修改消息历史本身，而是在构造 API message 时临时注入：

```python
if idx == current_turn_user_idx and msg.get("role") == "user":
    _injections = []
    if _ext_prefetch_cache:
        _fenced = build_memory_context_block(_ext_prefetch_cache)
        _injections.append(_fenced)
    ...
    api_msg["content"] = _base + "\n\n" + "\n\n".join(_injections)
```

这个设计的意义是：

- `messages` 中保存的原始用户消息不变
- 数据库存档的 transcript 不被动态 recall 污染
- 只有实际发给模型时，才临时注入 memory-context

这是 Hermes 控制“上下文污染”和“持久化洁净度”的关键实现。

---

### 5. 工具调用阶段：built-in memory 与 provider tools 分流

Hermes 对 memory 相关工具做了显式分流。

#### built-in memory 工具

```python
elif function_name == "memory":
    result = _memory_tool(..., store=self._memory_store)
```

#### provider 自定义工具

```python
elif self._memory_manager and self._memory_manager.has_tool(function_name):
    return self._memory_manager.handle_tool_call(function_name, function_args)
```

所以语义上：

- `memory` 是 Hermes 自带的内建记忆工具
- `honcho_*` / `supermemory_*` 之类是 provider 的专属工具

这也是为什么外部 provider 不会直接覆盖内建 `memory` 工具。

---

### 6. 每轮结束后：sync + queue_prefetch

一轮完成后，Hermes 会把结果同步给 provider：

```python
self._memory_manager.sync_all(original_user_message, final_response)
self._memory_manager.queue_prefetch_all(original_user_message)
```

这两步分别对应：

- `sync_all()`：把刚发生的 turn 写回 provider
- `queue_prefetch_all()`：为下一轮准备 recall 缓存

也就是说，provider 侧的思路不是“每轮现搜现算”，而是尽量提前做下一轮 recall 预热。

---

## 十五、以 Honcho 为例：provider 是怎么接入 Hermes 的

在所有 memory provider 里，Honcho 的实现最完整，也最能体现 Hermes 的设计意图。

相关文件包括：

- `plugins/memory/honcho/__init__.py`
- `plugins/memory/honcho/session.py`
- `plugins/memory/honcho/client.py`

### 1. Honcho 支持三种 recall 模式

看 `system_prompt_block()` 的实现，有三种模式：

```python
if self._recall_mode == "context":
    ...
elif self._recall_mode == "tools":
    ...
else:  # hybrid
```

对应含义：

#### context 模式

- 自动把相关记忆注入对话上下文
- 不暴露记忆工具给模型

#### tools 模式

- 不自动注入记忆
- 需要模型主动调用 `honcho_profile` / `honcho_search` / `honcho_context` 等工具

#### hybrid 模式

- 自动注入 + 工具调用 两者并存

这说明 Hermes 的 provider 抽象并不限制 recall 方式，而是允许插件自由选择：

- 完全自动
- 完全工具化
- 混合策略

---

### 2. Honcho 的 system prompt block 很克制

Honcho 的 `system_prompt_block()` 返回的主要是说明文字，例如：

- 当前启用 Honcho
- 当前是 context / tools / hybrid 哪种模式
- 工具怎么用

而不是把大段动态记忆直接塞进 system prompt。

这与 Hermes 整体设计完全一致：

> 静态信息放 system prompt，动态 recall 放 API-call-time 注入。

---

### 3. Honcho 真正的回忆逻辑在 `prefetch()`

`prefetch()` 的注释里写得很清楚：

```python
Return base context (representation + card) plus dialectic supplement.
```

也就是说它回忆的内容至少包含：

- base context
- representation
- AI identity card
- dialectic supplement

并且它支持更细的策略控制，例如：

- `context_cadence`
- `dialectic_cadence`
- `injection_frequency`

例如：

```python
if self._injection_frequency == "first-turn" and self._turn_count > 1:
    return ""
```

这意味着它甚至可以配置成：

- 只在第一轮自动注入
- 后续轮次不再持续注入

从工程上看，这是一种很现实的 token 成本控制手段。

---

### 4. 为什么 Hermes 要主动清理 `<memory-context>`

在 `run_agent.py` 中有专门的清理逻辑：

```python
user_message = sanitize_context(user_message)
persist_user_message = sanitize_context(persist_user_message)
```

它针对的就是 Honcho 这类 provider 注入的 fenced context。

原因是：

- recall 内容是临时注入到当前 user message 的 API 版本里
- 某些保存路径可能把它一起落进 transcript
- 下一轮读取历史时，这段内容可能被误当成“用户原话”回流

所以 Hermes 会在后续 turn 开始时显式剥掉：

- `<memory-context> ... </memory-context>`
- 配套的 `[System note: ...]`

避免 recalled context 变成污染用户输入的“假历史”。

---

### 5. Honcho 与 built-in memory 的本质差异

built-in memory 更像一个“受约束的小笔记本”：

- 一条一条纯文本 entry
- 简单的 add / replace / remove
- 容量极小
- 直接进 system prompt

而 Honcho 更像一个“外部记忆服务”：

- 支持 session 级同步
- 支持用户画像和上下文建模
- 支持 recall mode 切换
- 支持多种 memory tools
- 有自己的 session manager / client / flush 逻辑

所以两者虽然都叫 memory，本质上其实是两种不同层级的能力。

---

## 十六、为什么 Hermes 要做成三层记忆，而不是一锅炖

从源码实现看，这不是偶然，而是一个明确的架构取舍。

### 1. built-in memory 负责“小而稳”

`MEMORY.md` / `USER.md` 只适合承载：

- 高频会用到
- 长期稳定
- 值得每轮都带着
- 文本足够短

比如：

- 用户偏好简洁回答
- 项目在某个目录
- 机器环境有某个特殊约束

如果把所有历史都塞进这里，会立刻出现：

- prompt 变大
- snapshot 管理变复杂
- caching 价值下降

所以 built-in 层必须“克制”。

### 2. session_search 负责“大而全但按需取”

SQLite + FTS5 存的是全量会话，它不追求每轮都带着，而是：

- 需要时搜
- 搜到后 summarize
- 把摘要带回主模型

这让 Hermes 获得近似“无限历史”的能力，但不会每轮都付出上下文成本。

### 3. provider 负责“更智能的外部长期记忆”

外部 provider 这一层解决的是：

- 语义 recall
- 用户画像
- 长周期行为建模
- 外部索引 / 图谱 / 数据库存储

也就是说，它补的是内建记忆和 session_search 之间的空白层。

---

## 十七、我认为最重要的工程亮点

如果只挑一个最值得夸的实现点，那就是：

> **冻结的 built-in system-prompt snapshot + 动态 provider recall 注入 user message**

这套分拆非常工程化。

### 为什么好

因为它同时满足了三件事：

1. 让关键事实始终在上下文里
2. 让动态 recall 能随着 query 变化
3. 尽量不破坏 system prompt 的 prefix cache

很多 agent 框架会把所有记忆统一塞进 prompt，结果是：

- 每轮 prompt 都在变
- cache 几乎废掉
- token 成本失控

Hermes 在这里明显是“为真实运行成本优化过”的。

---

## 十八、如果按架构审查标准看，当前还有哪些可改进点

### 1. built-in memory 仍未完全 provider 化

`agent/memory_manager.py` 的注释里已经隐含了理想状态：

- built-in 也是 provider
- external 也是 provider
- 都走统一 manager

但当前真实实现还不是这样，而是：

- built-in 走 `_memory_store`
- external 走 `_memory_manager`

因此 `run_agent.py` 里保留了不少桥接逻辑。

### 2. `remove` 没有桥接到 external provider

这是最明显的状态一致性问题之一：

- add / replace 会通知 provider
- remove 不通知

如果 external provider 也在镜像 built-in memory，这里就可能漂移。

### 3. flush 路径和主工具调用路径不统一

`flush_memories()` 直接执行 `_memory_tool(...)`，不是走统一的 tool dispatch。这样虽然简单，但会带来：

- bridge 漏洞
- provider side effect 不一致
- 行为在正常 turn / flush turn 之间不完全相同

### 4. 文档、注释、实现有少量漂移

尤其是：

- 顶部注释还提 `read`
- 某些文档示例和当前 schema 引导不完全一致

这类问题不影响主流程，但会影响后续维护判断。

---

## 十九、再压缩成一句架构话术

如果要用一句话描述 Hermes 的记忆系统，可以说：

> Hermes 采用“常驻小记忆 + 全量会话检索 + 可插拔外部语义记忆”的三层架构，并通过冻结 snapshot 与动态 recall 分离来兼顾长期记忆能力和 prompt caching 性能。

---

## 二十、如果把 Hermes 的记忆系统拆成“职责边界图”

可以把它抽象成 4 个层次：

### A. 存储层
- `MEMORY.md`
- `USER.md`
- `state.db`
- 外部 provider 自己的存储（Honcho / Mem0 / etc.）

### B. 记忆操作层
- `tools/memory_tool.py`：内建记忆的增删改
- `tools/session_search_tool.py`：历史会话检索
- `plugins/memory/*`：外部 provider 工具与同步

### C. 编排层
- `run_agent.py`
- `agent/memory_manager.py`

这一层负责：
- 初始化
- prompt 注入
- 工具分发
- turn 生命周期
- flush / shutdown

### D. prompt 融合层
- system prompt 中的 frozen built-in memory
- 当前 user turn 中的 `<memory-context>` 动态 recall

这个分层看起来简单，但实际上很重要，因为 Hermes 不是“谁能存数据谁就叫 memory”，而是严格区分：

- **存储**
- **检索**
- **编排**
- **注入**

---

## 二十一、内建 memory 的真实定位：不是知识库，而是“稳定前置信息”

如果只看 `MemoryStore`，可能会觉得它很简陋：
没有 embedding，没有向量检索，没有冲突推理。

但从源码角度，它本来就不是为了做这些。

它真正承担的是：

> 把最小规模、最高价值、最稳定的长期事实，变成每轮都能直接看到的固定前置信息。

所以内建 memory 更像：

- shell 的环境变量
- 用户画像摘要
- 项目约定便签
- agent 的长期操作注意事项

而不是：
- 完整知识库
- 长篇工作日志
- 历史任务数据库

这也是为什么它容量被硬性卡得很小。

---

## 二十二、为什么 `session_search` 其实是 Hermes 的“真正长时记忆底盘”

很多用户第一眼会觉得：

- `MEMORY.md` = 长期记忆
- `session_search` = 一个附加搜索工具

但从源码看，更准确的说法应该是：

> **Hermes 的大容量长期记忆，实际上主要由 `state.db + session_search` 承担。**

原因很直接：

### 1. 容量上
- built-in memory 只有几千字符
- session DB 是全量会话，理论上无限增长

### 2. 语义上
- built-in memory 适合稳定事实
- session_search 适合找“我们之前到底怎么做的”

### 3. 工程上
`session_search` 不是简单 grep，而是：

- SQLite FTS5 检索
- 会话去重与 lineage 处理
- 截断匹配窗口
- 辅助模型总结

所以它实际上已经是一套“conversation recall pipeline”。

如果说 built-in memory 是“前置缓存”，
那 session_search 更像“冷数据层 + 检索层”。

---

## 二十三、Hermes 为什么要有 `flush_memories()`，而不是完全依赖后台 review

这两套机制表面上都在“自动存记忆”，但触发点完全不同。

### 后台 review
- 是**周期性**
- 发生在正常 turn 之后
- 目标是“顺手整理一下有没有值得存的东西”

### flush_memories
- 是**上下文即将丢失时**
- 发生在 compression / reset / exit 前
- 目标是“抢救那些还没来得及进长期记忆的重要事实”

所以 flush 本质上更像：

> 一个上下文回收前的 salvage pass（打捞过程）

这在 agent 系统里其实很实用，因为很多重要事实只在最近几轮出现，一旦压缩或重置，可能就再也不在 prompt 里了。

---

## 二十四、从一致性角度看，当前系统存在三种“状态”

这是 Hermes 记忆实现里最容易忽略的点。

### 状态 1：磁盘/后端真实状态
例如：
- `MEMORY.md`
- `USER.md`
- provider backend
- `state.db`

这是持久化真相。

### 状态 2：进程内 live state
例如：
- `self.memory_entries`
- `self.user_entries`
- provider 的缓存 / session manager / prefetch cache

这是运行时真相。

### 状态 3：当前 session 可见状态
例如：
- frozen system prompt snapshot
- 本轮注入的 `<memory-context>`

这是“模型眼中的真相”。

这三者**并不总相等**。

最典型的例子就是 built-in memory：

- 写盘后，磁盘状态已更新
- live state 也更新了
- 但当前 session 的 system prompt snapshot 不更新

所以如果从 agent 认知论角度说：

> Hermes 明确允许“存储真相”和“当前可见真相”短暂不一致。

这是有意识的工程折中，不是 bug。

---

## 二十五、这个“不一致”为什么不是设计缺陷

很多系统设计里，一提到“状态不一致”就会觉得危险。
但在 Hermes 这里，这个不一致是**可控且有价值**的。

### 它换来了什么？

#### 1. 更强的缓存稳定性
system prompt 不变，prefix cache 命中更稳。

#### 2. 更简单的多轮语义
当前 session 不会因为中途写了 memory 而突然“重写过去的自我认知”。

#### 3. 更低的 token 波动
不会因为每次写 memory 都膨胀 prompt。

所以这里本质上是在拿：

- **即时可见一致性**

换取：

- **性能**
- **稳定性**
- **成本可控性**

这是合理的。

---

## 二十六、如果要继续演进，我会怎么重构这套系统

如果按“保守演进、尽量不打坏现有行为”的思路，我会分 3 步。

### 第一步：把 built-in memory 真的封成 `BuiltinMemoryProvider`

目标不是重写逻辑，而是把接口统一。

让它实现：
- `system_prompt_block()`
- `handle_tool_call()`
- `on_memory_write()`
- `on_session_end()`

这样 `run_agent.py` 里很多 if/else 能减少。

### 第二步：统一 tool 路由与 flush 路由
让：

- 正常 turn 的 memory tool 调用
- flush 阶段的 memory-only 调用

都走同一条 dispatch pipeline。

这样可以避免：
- bridge 漏同步
- side-effect 不一致
- 某些 provider hook 只在正常路径触发

### 第三步：定义“memory mirror policy”
明确规定 built-in 与 external provider 之间的关系：

到底是：
1. built-in 为主，provider 镜像
2. provider 为主，built-in 仅 prompt snapshot
3. 二者独立并行，不保证一致

目前源码是介于 1 和 3 之间，有点模糊。

---

## 二十七、如果按产品角度总结，Hermes 其实在解决两个不同问题

### 问题 1：用户不想重复解释自己
这由：
- `USER.md`
- `MEMORY.md`

解决。

比如：
- 偏好
- 环境
- 项目结构
- 使用习惯

### 问题 2：用户不想重复讲以前做过的事
这由：
- `state.db`
- `session_search`
- 外部 provider recall

解决。

前者更像“persona continuity”，
后者更像“historical recall”。

Hermes 把这两个问题分开处理，是对的。

---

## 二十八、最后给一个更精炼的源码判断

如果只基于源码，不看宣传文案，我会这样评价 Hermes 记忆系统：

### 它最强的地方
不是“记忆能力最花哨”，而是：

- 分层清楚
- 成本意识强
- prompt caching 意识强
- 多轮会话污染控制做得细

### 它最值得补的地方
不是“多加一个 provider”，而是：

- built-in / provider 抽象统一
- 一致性策略明确
- flush 路径统一化

---

## 三十、`session_search` 这条链，为什么比看起来更重要

很多人会把 `session_search` 理解成：

> “一个搜索旧聊天记录的小工具”

但源码层面它其实是 **Hermes 的长期历史回忆基础设施**。

相关核心文件：

- `hermes_state.py`
- `tools/session_search_tool.py`

---

### 1. 数据底座：`state.db`

数据库路径：

```python
DEFAULT_DB_PATH = get_hermes_home() / "state.db"
```

这意味着 session history 同样是 profile 隔离的，不同 profile 不共享历史。

SQLite schema 里关键表有：

- `sessions`
- `messages`
- `messages_fts`

其中：

#### `sessions`
存会话级元信息：

- `id`
- `source`
- `model`
- `system_prompt`
- `parent_session_id`
- `started_at`
- `ended_at`
- `message_count`
- `tool_call_count`
- 标题和成本信息等

#### `messages`
存每条消息：

- `role`
- `content`
- `tool_calls`
- `tool_name`
- `reasoning`
- `timestamp`

#### `messages_fts`
这是 FTS5 虚表：

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
    content,
    content=messages,
    content_rowid=id
);
```

它通过 trigger 自动和 `messages` 同步。

所以本质上：

> 每一条历史消息都被自动纳入全文检索索引。

---

### 2. `search_messages()` 不是简单 SELECT，而是完整 FTS pipeline

`hermes_state.py` 中：

```python
def search_messages(query, source_filter=None, exclude_sources=None, role_filter=None, ...)
```

流程是：

#### 第一步：清理 query
```python
query = self._sanitize_fts5_query(query)
```

避免非法 FTS5 语法直接炸掉查询。

#### 第二步：按 FTS5 做 rank 搜索
SQL 里：

```sql
SELECT
    m.id,
    m.session_id,
    m.role,
    snippet(messages_fts, 0, '>>>', '<<<', '...', 40) AS snippet,
    ...
FROM messages_fts
JOIN messages m ON m.id = messages_fts.rowid
JOIN sessions s ON s.id = m.session_id
WHERE messages_fts MATCH ?
ORDER BY rank
LIMIT ? OFFSET ?
```

这里做了几件很实用的事：

- 用 `MATCH` 走 FTS5
- 用 `snippet()` 直接生成匹配片段
- 关联 session metadata
- 按 `rank` 排序

#### 第三步：补上下文窗口
每个 match 再补前后 1 条消息：

```python
SELECT role, content FROM messages
WHERE session_id = ? AND id >= ? - 1 AND id <= ? + 1
```

所以 search result 不只是一个孤立 hit，而是带一点局部语境。

#### 第四步：去掉 full content
```python
match.pop("content", None)
```

避免结果太大。

---

### 3. `session_search_tool.py` 做了第二层语义加工

真正暴露给模型的不是 `search_messages()` 原始结果，而是 `session_search()`。

这层额外做了几件非常关键的事。

#### 1）按 session 聚合，而不是按 message 返回
FTS 命中的是 message，但 agent 真正需要回忆的是“那次会话发生了什么”。

所以它会：

- 先拿 raw matches
- 再按 `session_id` 聚合

#### 2）处理 session lineage，避免把当前会话/子会话搜回来

Hermes 有：

- compression continuation session
- delegation child session

所以会话可能形成 parent-child 链。

工具里专门有：

```python
def _resolve_to_parent(session_id: str) -> str:
```

它会一路向上追 `parent_session_id`，找到 root parent。

这样做的意义是：

- 避免当前会话被自己搜索到
- 避免 compression 之后的新旧 session 被当成两个不相干会话
- 避免 delegation child session 干扰主会话 recall

这说明 Hermes 的历史记忆不是“平铺日志”，而是考虑了会话谱系。

#### 3）不是原文返回，而是“截段 + 总结”

每个匹配 session 会：

```python
messages = db.get_messages_as_conversation(session_id)
conversation_text = _format_conversation(messages)
conversation_text = _truncate_around_matches(conversation_text, query)
```

然后送给辅助模型总结。

其中 `_truncate_around_matches()` 也不是傻截断，而是优先保留：

1. query 完整短语命中
2. 多关键词近邻共现
3. 单关键词命中

然后选覆盖最多匹配点的窗口。

这其实已经是一个小型 retrieval-ranking-truncation pipeline 了。

---

### 4. 这条链本质上在做什么

如果抽象一下，`session_search` 做的是：

> 全量历史消息库  
> → FTS 候选召回  
> → session-level 聚合  
> → lineage 去重  
> → query-centered truncation  
> → LLM 摘要压缩  
> → 返回给主模型

所以它不只是搜索，而是：

> **历史会话记忆压缩回放器**

---

## 三十一、为什么 `session_search` 比 built-in memory 更像“真正的长期记忆”

从源码职责上看：

### built-in memory
更像：
- 常驻前置事实
- 稳定 persona/context
- 小而固定

### session_search
更像：
- 可无限扩展的历史回忆仓
- 需要时再提取
- 针对任务动态取回相关片段

换句话说：

- `MEMORY.md` / `USER.md` 解决的是“我该一直记住什么”
- `session_search` 解决的是“我之前到底做过什么”

后者其实更接近人类意义上的“回忆过往经历”。

---

## 三十二、Hermes 为什么没有把 session_search 的结果直接持久化成 memory

这是个很值得注意的设计点。

表面看，好像系统完全可以：

- 搜到一个旧会话
- 总结出几个结论
- 自动写进 `MEMORY.md`

但 Hermes 没这么做，原因从架构上也能推出来：

### 1. 会污染 built-in memory
历史会话内容天然很多、很杂，如果自动沉淀进 small memory，很容易把那层变成垃圾堆。

### 2. 会引入“摘要的摘要”
旧会话已经是回放摘要了，再写进 memory，就变成二次压缩，信息保真度更差。

### 3. 会打破“稳定事实 vs 动态历史”的边界
Hermes 现在这条边界其实很清晰：

- built-in：稳定事实
- session_search：动态查历史

这是好事。

---

## 三十三、再深入一层：Honcho 为什么不是“另一个 memory_tool”

Honcho 的实现明显不只是提供几个工具 schema，它其实带了自己的一整套会话模型。

从源码看，`HonchoMemoryProvider` 至少维护这些状态：

```python
self._manager
self._config
self._session_key
self._prefetch_result
self._base_context_cache
self._turn_count
self._last_context_turn
self._last_dialectic_turn
```

这说明它内部不是 stateless tool，而是有：

- 配置
- session identity
- 缓存
- cadence 控制
- recall pipeline

---

### 1. Honcho 初始化时会解析自己的运行策略

在 `initialize()` 里会读出：

- `recall_mode`
- `injectionFrequency`
- `contextCadence`
- `dialecticCadence`
- `dialecticDepth`
- `reasoningLevelCap`

也就是说 Honcho 不是简单“开/关”，而是支持一套 recall 策略矩阵。

### 2. Honcho 的 session key 不是简单复用 Hermes session_id

它会调用：

```python
cfg.resolve_session_name(
    session_title=session_title,
    session_id=session_id,
    gateway_session_key=gateway_session_key,
)
```

说明它在 provider 侧还会重新解析“这次会话在 Honcho 里叫什么”。

这很重要，因为 provider 的 session 语义未必和 Hermes 一样：

- Hermes 可能按 CLI/gateway turn session 组织
- Honcho 可能按用户、标题、聊天上下文来组织

所以它不是机械映射，而是有一层 name resolution。

### 3. Honcho 甚至会尝试迁移 memory files

初始化时有：

```python
self._manager.migrate_memory_files(self._session_key, mem_dir)
```

也就是说：

- 当会话是新建的
- 且 session strategy 允许时
- 它会尝试把 `MEMORY.md` / `USER.md` 迁进 Honcho 侧

这进一步说明 Hermes 把 provider 视为**长期记忆后端**，而不是纯外部辅助。

---

## 三十四、Honcho 的 recall 是“两层结构”

`prefetch()` 注释说得很明白：

1. Base context
2. Dialectic supplement

### 第一层：base context
来自：

```python
self._manager.get_prefetch_context(self._session_key)
```

它会缓存到：

```python
self._base_context_cache
```

特点是：

- 偏稳定
- 不需要每轮都重新拉
- 由 `context_cadence` 控制刷新频率

这层更像“长期画像 / 背景卡片”。

### 第二层：dialectic supplement
来自：

```python
self._run_dialectic_depth(query)
```

特点是：

- 和本轮 query 更相关
- 更像 provider 后端对“当前问题”的推理性回答
- 由 `dialectic_cadence` 控制

这层更像“基于长期记忆对当前问题做的针对性补充”。

### 这两层分开有什么意义？

这样可以把 recall 分成：

- **稳定层**：不用频繁更新
- **推理层**：更贵，但更相关

这是一个很合理的成本分层。

---

## 三十五、Honcho 的 `queue_prefetch()` 说明 Hermes 在记忆系统上很有“异步化思维”

`queue_prefetch()` 并不是当前轮要用时再同步查询，而是提前启动后台线程：

- context refresh 线程
- dialectic prefetch 线程

源码里：

```python
self._manager.prefetch_context(self._session_key, query)
```

以及：

```python
self._prefetch_thread = threading.Thread(...)
self._prefetch_thread.start()
```

这说明 provider recall 的目标不是“绝对最新”，而是“下一轮大概率够用且不阻塞”。

所以 Hermes 在这里更偏向：

> **低延迟 recall**，而不是**强同步 recall**

这是面向交互体验的设计。

---

## 三十六、Honcho 的 `sync_turn()` 也说明 provider 是“异步写后端”的

```python
for chunk in self._chunk_message(user_content, msg_limit):
    session.add_message("user", chunk)
for chunk in self._chunk_message(assistant_content, msg_limit):
    session.add_message("assistant", chunk)
self._manager._flush_session(session)
```

而且它跑在线程里。

几个细节值得注意：

### 1）消息过长会分块
因为 Honcho API 有 message size 限制。

### 2）不是纯 fire-and-forget
如果上一条 `_sync_thread` 还活着，它会：

```python
self._sync_thread.join(timeout=5.0)
```

说明它仍然控制并发，不让同步线程无限堆积。

### 3）session end 时会强制 flush
```python
self._manager.flush_all()
```

这和 Hermes 的 `flush_memories()` 思想类似，都是“结束前兜底”。

---

## 三十七、一个容易忽略的细节：Honcho 只镜像 built-in 的 `user:add`

`on_memory_write()` 里：

```python
if action != "add" or target != "user" or not content:
    return
```

也就是说，Honcho 只在这种情况下镜像：

- action = `add`
- target = `user`

然后调用：

```python
self._manager.create_conclusion(self._session_key, content)
```

这意味着它并不打算“完整镜像 built-in memory”，而是有选择地吸收 built-in user profile 事实。

这其实能解释前面那个“为什么 remove 不桥接”的问题：

> 对某些 provider 来说，built-in memory 并不是要 1:1 镜像的主数据源。

换句话说，built-in 与 provider 在 Hermes 当前架构下，更像：

- **并行的两种记忆机制**
- 有少量桥接
- 但不是强一致双写系统

---

## 三十八、所以 Hermes 当前最准确的记忆架构描述应该是：

不是：

> 一个统一记忆系统，多个后端

而更像：

> 一个内建小记忆层  
> + 一个历史会话检索层  
> + 一个外部记忆增强层  
> 三者在 `run_agent.py` 被编排到一起工作

这三者会互相配合，但并没有完全收敛成单一抽象。

这也是为什么我前面说：

- `MemoryManager` 的注释代表了一种“理想统一态”
- 但当前代码现实还是“双轨+桥接”

---

## 三十九、如果把这套设计映射到人类认知，会比较像什么

可以类比成：

### 1. `USER.md` / `MEMORY.md`
你的便签纸 / 常识卡片  
“这个人喜欢简洁回答”“那台服务器用 2222 端口”

### 2. `state.db + session_search`
你的完整工作日志 + 可全文检索档案柜  
“上个月我们到底怎么修那个 bug 的？”

### 3. Honcho / 其他 provider
你的外部助理记忆系统 / 知识图谱 / 长期画像引擎  
“结合你长期行为模式，当前问题最相关的背景是什么？”

这个比喻和源码职责是比较对齐的。

---

## 四十、给一个更偏架构师视角的结论

从工程成熟度来看，Hermes 记忆系统最成熟的不是“算法”，而是**边界管理**：

- 哪些东西常驻
- 哪些东西按需召回
- 哪些东西进 system prompt
- 哪些东西只进当前 user turn
- 哪些东西落 SQLite
- 哪些东西交给 provider
- 哪些东西要在 session 结束前强制 flush

这些边界划分得相当清楚。

而它当前最不成熟的地方，不是 recall 本身，而是**统一抽象还没完全收口**。
