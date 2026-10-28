# 源码阅读指南：从小白到看懂 MiniAgent

MiniAgent 的代码量不大，但“运行时 + 插件 + 异步 + 会话恢复”叠在一起，第一次读容易迷路。这份指南给出一条按数据流展开的阅读顺序，并解释读之前需要补的概念。

目标不是背下每个函数，而是建立一张地图：

- 一个任务从 CLI 进入到模型返回，中间经过哪些层；
- 每层负责什么、不负责什么；
- 出问题时，应该在哪个边界找原因。

## 适合谁

- 会基础 Python，但没有系统用过 `asyncio`；
- 调用过 LLM API，但不清楚 tool calling 的完整消息格式；
- 想改 MiniAgent，或者想自己写一个 mini harness。

不需要编译原理、分布式系统或深度学习数学。

## 先看全景

```text
CLI / Python API
      │
      ▼
Harness（应用运行时）
  ├─ 插件表 / services / events
  ├─ Registry / Policy / Session
  └─ busy 守卫、依赖激活、失败回滚
      │
      ├──────────────┬──────────────┐
      ▼              ▼              ▼
 Agent Loop      工具边界         模型边界
 agent.py        registry.py      llm.py
      │              │              │
      │              ├─ Skill 白名单
      │              ├─ JSON Schema
      │              ├─ Policy 授权
      │              └─ executor
      │                             │
      └──────────► Session / context_view ◄──┘
```

一次任务的路径大致是：

1. CLI 解析参数、加载配置、构造 `Harness`；
2. `Harness.arun()` 打开 Session，启动尚未激活的插件；
3. 从 `services` 取出 `agent_loop`，进入“模型—工具”循环；
4. 每轮先用 `context_view()` 裁剪消息，再调用 `LLMBackend.complete()`；
5. 模型返回 `tool_calls`，逐个经过 Skill 白名单、JSON Schema、Policy；
6. 执行结果写回消息和 Session，进入下一轮；
7. 没有工具调用时返回最终文本；
8. 无论成功、失败还是取消，`finally` 都会关闭 Session、清理本轮状态。

---

## 一、先把概念对齐

| 词 | 在本项目里指什么 | 不是什么 | 代码位置 |
|---|---|---|---|
| **Harness / 运行时** | 持有实例状态、管理插件生死、调度任务、做并发守卫的宿主 | 不是模型，也不是 Agent 循环本身 | [harness.py](../miniagent/harness.py) |
| **Plugin** | 可注册、可启动、可清理的**生命周期单元** | 不是 pip 包，也不必须是独立进程 | [harness.py](../miniagent/harness.py) |
| **Scope / `ctx`** | 插件 setup 时拿到的“登记簿 + 清理栈” | 不是 LLM 的上下文窗口 | [harness.py](../miniagent/harness.py) |
| **Service** | 注册在 `app.services` 里的**进程内命名对象** | 不是微服务，不是网络端口，不是工具 | [harness.py](../miniagent/harness.py) |
| **Tool** | 暴露给模型、由模型决定调用的函数 | 不是插件；工具通常由插件注册 | [registry.py](../miniagent/tools/registry.py) |
| **Agent Loop** | “请求模型 → 执行工具 → 再请求模型”的决策循环 | 不是 `asyncio` 的事件循环 | [agent.py](../miniagent/agent.py) |
| **Session** | 追加写的 JSONL 事件日志 + 恢复协议 | 不只是聊天记录数组 | [session.py](../miniagent/session.py) |
| **Event** | 运行过程中的广播点，供观察者订阅 | 不是消息队列，不控制授权 | [harness.py](../miniagent/harness.py) |
| **Policy** | 工具执行前的授权检查 | 不是容器沙箱 | [policy.py](../miniagent/policy.py) |
| **Skill** | prompt + 工具白名单 + temperature + 迭代预算 | 不是插件，不参与资源生命周期 | [skills.py](../miniagent/skills.py) |

### 1. “运行时”到底是什么

“运行时”在不同语境有三层，读代码时要分清：

| 层 | 是什么 | 在本项目中的体现 |
|---|---|---|
| **语言运行时** | Python 解释器 + `asyncio` 事件循环 | `asyncio.run(cli_run(args))` |
| **应用运行时** | `Harness` 对象：状态、生命周期、调度、守卫 | [harness.py](../miniagent/harness.py) |
| **宿主进程** | `miniagent` 这个 CLI 进程 | [cli.py](../miniagent/cli.py) |

通常说“MiniAgent 的运行时”，指的是第二层 `Harness`。它负责六件事：

1. **持有状态**：`messages`、`services`、`registry`、`policy`、`session`、插件表。
2. **管理生命周期**：`use()` 注册 → `start()` 激活 → `unuse()` / `aclose()` 逆序清理。
3. **解析依赖**：按 `requires` / `inject` 做拓扑排序，检测环。
4. **调度任务**：`arun()` 打开会话、启动插件、取出 `agent_loop` 服务并执行。
5. **并发守卫**：`busy` 期间禁止改插件、换 Skill、重入任务。
6. **提供观察点**：`emit()` 发事件，CLI 靠它显示工具进度。

简化后的核心路径：

```python
async def arun(self, query):
    self.busy = True
    await self._start()                 # 激活插件
    loop = self.services["agent_loop"]  # 取决策循环
    return await loop(self, query)      # 跑任务
```

一句话概括：

> **Harness 管“生死和秩序”，Agent Loop 管“下一步做什么”。**

Harness 自己不决定调用哪个工具，它只是把模型、工具、会话、权限、插件粘在一起，并保证顺序和收尾正确。

### 2. “服务”到底是什么

服务就是**一个被命名的、进程内的能力对象**，存放在 `app.services` 字典里：

```python
ctx.service("llm", backend)      # 注册：名字 -> 对象
backend = ctx.get("llm")        # 取用
```

它不是 HTTP 服务，也不是独立进程。它解决的是“插件之间如何共享能力，而不互相 import 具体类”：

- `llm_plugin` 注册 `"llm"` 服务 → [llm.py](../miniagent/llm.py)
- `agent-loop` 插件注册 `"agent_loop"` 服务 → [agent.py](../miniagent/agent.py)
- MCP 插件注册 `"mcp:<name>"` 服务 → [mcp_plugin.py](../miniagent/extensions/mcp_plugin.py)

服务有三个关键约束：

1. **名字唯一**：重复注册直接抛 `ValueError`。
2. **生命周期归注册者**：`ctx.service()` 会往 `AsyncExitStack` 压一个 `services.pop(name)`，插件卸载时自动移除。
3. **依赖要声明**：`ctx.get("llm")` 只是查找，不算声明依赖；要让生命周期知道“我必须等 llm 先启动”，得写 `inject=("llm",)`。

为什么不用全局变量或直接 import？因为要支持**替换和测试**：

- CLI 的 `/model` 切换会卸载旧 `llm` 插件、注册新 backend，服务名不变，消费者无感知。
- 测试可以传一个 `FakeBackend` 进 `Harness(backend)`，完全不碰网络。

这属于一种**作用域内的服务定位 + 依赖声明**：`ctx.get()` 是服务定位，`inject` 是给生命周期看的依赖声明。

### 3. Plugin、Scope、Context 三者的关系

```python
def my_plugin(ctx):          # ctx 是 Scope，不是对话上下文
    ctx.service("x", 1)      # 注册服务
    ctx.tool(my_tool)        # 注册工具
    ctx.on("tool.end", cb)   # 注册事件监听
    ctx.cleanup(close_db)    # 注册清理回调
```

- **Plugin** 是注册单元：`Plugin(name, setup, requires, inject)`。
- **Scope** 是这次 setup 的“所有权边界”：它记录这个插件注册了什么，卸载时逆序撤销。
- **Context（`ctx`）** 就是这个 Scope 的别名。它和 LLM 的 context window 完全不是一回事。

### 4. 三个“同名不同物”

| 说法 | 含义 A | 含义 B | 含义 C |
|---|---|---|---|
| **context** | 插件 `ctx`（Scope） | LLM 上下文（messages / token 预算） | `contextvars.ContextVar`，见 [code_tools.py](../miniagent/tools/code_tools.py) |
| **loop** | `agent_loop`：模型—工具迭代 | `asyncio` event loop：协程调度 | reactive 生命周期的“重新协调”，不是后台轮询 |
| **runtime** | Python 解释器 / 事件循环 | `Harness` 应用运行时 | `miniagent` CLI 进程 |

读代码时看到 `ctx` 就想“登记簿”，看到 `context_view` 就想“发给模型的裁剪视图”，看到 `ContextVar` 就想“当前异步任务的隐式变量”，就不会混。

---

## 二、需要补哪些基础

### 必修 1：Python 语言机制

不需要精通元编程，但要能顺畅读懂这些用法：

- **函数是一等对象 / 闭包 / lambda**：插件大量用 `lambda c: ...`。
- **`dataclass`**：`Plugin`、`Tool`、`Skill` 都是 dataclass。
- **`*args` / `**kwargs`**：`tool.executor(**arguments)` 把模型给的 JSON 参数直接展开。
- **异常与异常组**：`BaseExceptionGroup` 用于“setup 和 cleanup 同时失败”。
- **`inspect`**：判断返回值是否 awaitable。
- **`with` / `__enter__` / `__exit__`**：理解资源释放的 LIFO 语义。

够用标准：能解释 `Plugin` 为什么可以只写一个函数，以及 `executor(**arguments)` 在参数不匹配时会发生什么。

### 必修 2：`async` / `await` 与 `asyncio`

这是 MiniAgent 的核心运行机制所在，也是最大的门槛。至少要掌握：

- 协程（coroutine）、Task、event loop 的区别；
- `await` 让出控制权意味着什么；
- `asyncio.run()`、`create_task()`、`gather()`、`wait_for()`、`shield()`；
- `CancelledError` 与 `finally` 的执行时机；
- **异步上下文管理器** `async with`；
- **`AsyncExitStack`** 的 LIFO 回调机制；
- 取消一个任务时，为什么清理逻辑仍然要执行。

对应到项目：

| 机制 | 项目中的使用 |
|---|---|
| `asyncio.run` | CLI 启动一个事件循环跑完整个 REPL |
| `create_task` | MCP 连接由独立 owner task 持有 |
| `wait_for` | Shell 超时、MCP 启动超时 |
| `shield` | 超时后仍保证终止子进程 |
| `gather` | 同时排空 stdout / stderr 并等待进程结束 |
| `CancelledError` | 工具取消后标记 uncertain、补齐工具结果 |
| `AsyncExitStack` | 插件清理、服务 / 工具 / 事件注销 |

够用标准：能说清“一个 `arun()` 从开始到 `finally` 关闭 Session，中间哪些地方可能被取消，取消后哪些代码仍会执行”。

### 必修 3：资源生命周期与依赖注入

需要理解几个工程概念：

- **所有权（ownership）**：谁注册，谁负责清理。
- **作用域（scope）**：一次插件启动内的注册边界。
- **拓扑排序**：`requires` / `inject` 决定启动顺序，卸载则逆序。
- **回滚（rollback）**：启动到一半失败，已启动的插件要撤销。
- **状态机**：插件在 `PENDING` / `ACTIVE` 之间切换；Scope 关闭后不可再注册，运行中禁止变更。
- **依赖注入 vs 服务定位**：MiniAgent 是“按名查找 + 声明依赖”，不是构造函数注入。

对应代码：`Harness._start()` 是启动流程，`Harness.unuse()` 是逆序卸载。

够用标准：能解释为什么 `unuse("base")` 在严格模式下会被拒绝，而在 `reactive=True` 下会连同消费者一起清理。

### 必修 4：LLM tool calling 协议

不需要会训练模型，但要能手写一轮工具调用的消息结构：

```json
[
  {"role": "system", "content": "..."},
  {"role": "user", "content": "..."},
  {"role": "assistant", "content": null, "tool_calls": [
    {"id": "call_1", "type": "function",
     "function": {"name": "read", "arguments": "{\"path\":\"a.py\"}"}}
  ]},
  {"role": "tool", "tool_call_id": "call_1", "content": "文件内容..."}
]
```

要理解：

- 工具定义是 **JSON Schema**（`type` / `properties` / `required`）；
- `arguments` 是**字符串**，不是对象，必须 `json.loads`；
- `tool_call_id` 必须与 assistant 消息里的 id 一一对应；
- `finish_reason` 可能是 `stop`、`tool_calls`、`length`，只有前两个是完整结果；
- 流式响应里工具调用是**按 index 分片**到达的，需要拼接；
- `reasoning_content` / `extra_content` 是部分 Provider 的扩展字段，不能丢。

对应代码：`agent_loop` 校验 call ID，`LLMBackend.complete()` 处理流式聚合。

够用标准：给一个 assistant 消息和工具结果，能判断下一轮请求应该长什么样；能解释为什么“先记录 assistant 消息，再执行工具”。

### 选修：按需补

| 知识 | 用在哪里 | 学到什么程度 |
|---|---|---|
| HTTP / SSE | Provider 请求、流式响应 | 知道分块传输和 `finish_reason` 即可 |
| 文件锁与持久化 | Session 的 JSONL、OS 写锁 | 理解 append-only、`fsync`、尾部半行截断 |
| 进程管理 | Shell 工具 | 理解超时、进程组、Windows `taskkill /T` |
| 测试 | `tests/` 与 `FakeBackend` | 会写 fixture、`tmp_path`、`monkeypatch` |
| 打包 | `pyproject.toml`、extras | 知道 console script 和可选依赖怎么装 |

---

## 三、源码阅读路线

不要从 `harness.py` 第一行硬啃。按“数据流”读会顺很多：

| 顺序 | 读什么 | 带着什么问题 | 预计 |
|---|---|---|---|
| 1 | [README.md](../README.md) + [lean-harness.md](lean-harness.md) | 它到底解决什么问题？边界在哪？ | 30 min |
| 2 | [agent.py](../miniagent/agent.py) | 一轮对话如何变成工具调用？为什么先记录 assistant 消息？ | 1 h |
| 3 | [registry.py](../miniagent/tools/registry.py) + [policy.py](../miniagent/policy.py) | 三道闸门分别防什么？`on_start` 为什么在 executor 之前？ | 1 h |
| 4 | [harness.py](../miniagent/harness.py) | Scope 如何保证清理？strict / reactive 差在哪？busy 守卫防什么？ | 2 h |
| 5 | [session.py](../miniagent/session.py) | 为什么 append-only？uncertain 为什么能阻止继续？压缩为什么不删原事件？ | 1.5 h |
| 6 | [llm.py](../miniagent/llm.py) | 流式工具调用怎么拼？为什么校验 `finish_reason`？ | 1 h |
| 7 | [plugins.py](../miniagent/plugins.py) + [mcp_plugin.py](../miniagent/extensions/mcp_plugin.py) | Shell 如何做到超时 / 取消 / 有界输出？MCP 为什么需要 owner task？ | 1 h |
| 8 | [cli_runtime.py](../miniagent/cli_runtime.py) | 一次 CLI 任务如何装配 Harness？事件如何变成终端输出？ | 1 h |

### 读 `agent.py`：抓住主循环

重点看这四件事：

1. 为什么先把 assistant 消息 `record` 下来，再执行工具？
2. 为什么要校验 tool call ID 的唯一性，并拒绝缺失或复用？
3. 工具抛异常时，`started_effect` 如何决定是否标记 uncertain？
4. 达到 `max_steps` 时为什么不是“返回已有内容”，而是直接报错？

### 读 `registry.py` + `policy.py`：抓住执行边界

执行顺序是固定的：

```text
Skill 白名单 → JSON Schema → Policy 授权 → on_start → executor
```

重点看：

- `Registry.get()` 和 `Registry.schemas()` 都在做 Skill 白名单过滤；
- `Policy.authorize()` 对 `read` / `write` 注入解析后的 `path`；
- `readonly` 在哪一步拒绝写操作；
- `confirm` 模式下 `approve` 回调返回什么才算同意；
- 非 TTY 时为什么默认拒绝。

### 读 `harness.py`：抓住四条线

把 300 多行拆成四条线，会容易很多：

| 线 | 关键方法 | 要回答的问题 |
|---|---|---|
| **注册线** | `use()` / `Plugin` | 插件什么时候只是登记，什么时候真正启动？ |
| **启动线** | `_start()` / `Scope` | 依赖如何排序？失败如何回滚？ |
| **清理线** | `unuse()` / `aclose()` / `_close_scopes()` | 为什么必须先关消费者？清理失败如何继续？ |
| **守卫线** | `_idle()` / `busy` | 为什么运行中不能改插件或换 Skill？ |

再重点理解 `AsyncExitStack`：每个 `ctx.service()`、`ctx.tool()`、`ctx.on()`、`ctx.cleanup()` 都往同一个栈里压一个撤销动作，Scope 关闭时 LIFO 执行。这就是“谁注册，谁负责清理”的实现。

### 读 `session.py`：抓住四个问题

1. 为什么是 append-only，而不是每次覆盖整个 JSONL？
2. 为什么要在 OS 写锁下截断“不完整的尾行”？
3. 崩溃恢复时，如何把“有 tool_call 但没有结果”的调用配对？
4. 上下文压缩为什么只改模型视图，不删原始事件？

### 读 `llm.py`：抓住协议边界

- 非流式：检查 `finish_reason`，把 SDK 对象转成普通 dict。
- 流式：按 `index` 拼接 tool call 的 `id`、`name`、`arguments`；最后再校验完整性。
- 为什么 `reasoning_content` / `extra_content` 要合并而不是覆盖。
- 客户端在 `aclose()` 时如何关闭。

### 读 `plugins.py` + `mcp_plugin.py`：抓住真实世界

- Shell 为什么要同时排空 stdout / stderr；
- 超时和取消时，如何保证子进程被终止；
- 输出为什么要“头 + 尾 + 截断标记”；
- MCP 为什么需要一个独立 owner task 持有 `async with Client(...)`。

### 读 `cli_runtime.py`：抓住装配

- 一个 `asyncio.run()` 如何服务整个 REPL；
- `display` 插件如何把 `tool.start` / `tool.end` 事件变成终端输出；
- `/model` 切换时，为什么先卸载旧 `llm` 插件、再注册新 backend；
- 非 TTY 和 `NO_COLOR` 下输出为什么不同。

---

## 四、最小实验（不需要 API Key）

以下实验都在仓库根目录运行，Python 3.11+。

### 实验 1：用假 Backend 跑通 Agent Loop

```python
import asyncio
from miniagent import Harness, Plugin, Policy
from miniagent.tools.registry import Tool


class FakeBackend:
    async def complete(self, messages, tools, stream):
        if messages[-1]["role"] == "user":
            return {"role": "assistant", "content": None, "tool_calls": [{
                "id": "c1", "type": "function",
                "function": {"name": "add_one", "arguments": '{"n": 1}'}}]}
        return {"role": "assistant", "content": messages[-1]["content"]}


async def main():
    app = Harness(FakeBackend(), policy=Policy(mode="trusted"))
    app.use(Plugin("math", lambda ctx: ctx.tool(Tool(
        "add_one", "add one",
        {"type": "object", "properties": {"n": {"type": "integer"}},
         "required": ["n"], "additionalProperties": False},
        lambda n, path: n + 1, access="read"))))
    print("answer:", await app.arun("go"))
    await app.aclose()


asyncio.run(main())
```

预期输出 `answer: 2`。这个实验对应 [test_harness.py](../tests/test_harness.py) 里的假 Backend 测试，能让你看清：模型返回什么、工具怎么执行、结果怎么回填。

### 实验 2：观察插件状态与清理顺序

```python
import asyncio
from miniagent import Harness, Plugin


async def main():
    seen = []
    app = Harness(reactive=True)
    app.use(Plugin("consumer", lambda ctx: seen.append(ctx.get("value")),
                   inject=("value",)))
    await app.start()
    print("after start:", app.states)

    app.use(Plugin("provider", lambda ctx: ctx.service("value", 1)))
    await app.start()
    print("after provider:", app.states, seen)

    await app.unuse("provider")
    print("after unuse:", app.states, seen)
    await app.aclose()


asyncio.run(main())
```

你会看到 `consumer` 先处于 `PENDING`，provider 出现后变为 `ACTIVE`，provider 卸载后又回到 `PENDING`。`agent-loop` 也会出现在状态里，因为 Harness 默认注册了它。

### 实验 3：制造一次 uncertain

在实验 1 的基础上做三处修改：

1. 把工具的 `access` 改为 `"write"`；
2. 让 executor 在开始后抛异常（例如 `raise RuntimeError("failed after start")`）；
3. 让 `FakeBackend` 第一轮请求这个工具。

然后：

```python
try:
    await app.arun("go")
except RuntimeError as error:
    print(error)          # Tool result is uncertain...
print(app.uncertain)      # {'c1'}
app.resolve("c1", "checked manually")
print(app.uncertain)      # set()
```

这个实验能直观解释“为什么未知副作用不能自动重放”。

### 实验 4：观察上下文压缩

```python
from miniagent.session import compact

msgs = [{"role": "system", "content": "s"}]
for i in range(100):
    msgs += [{"role": "user", "content": "u" * 100},
             {"role": "assistant", "content": "a" * 100}]

view = compact(msgs, 8000)
print("original:", len(msgs), "view:", len(view))
print(view[1]["content"][:40])
```

`view` 会明显短于原始消息，并且第 2 条是“较早对话已省略”的提示。原始 `msgs` 不会被修改。注意 `compact()` 要求预算大于输出预留（默认 1024），预算太小时会直接报错。

---

## 五、八个自测问题

如果能答出这八个，基本就“看懂”了：

1. 为什么 `agent_loop` 要先 `record` assistant 消息，再执行工具？
2. 为什么工具异常要区分“已开始执行”和“未开始执行”？
3. `ctx.get("llm")` 和 `inject=("llm",)` 有什么区别？
4. 为什么严格模式下不能直接 `unuse` 一个还有消费者的服务提供者？
5. 为什么流式响应必须等到 `finish_reason` 才能返回？
6. 为什么会话压缩只改模型视图，不删原始事件？
7. 为什么非 TTY 下 `confirm` 模式默认拒绝？
8. 为什么 MCP 连接要由一个独立的 owner task 持有？

---

## 六、常见误解

| 误解 | 实际情况 |
|---|---|
| “权限确认等于沙箱隔离” | Policy 只约束托管工具；插件、Shell、MCP 都是可信代码，不是容器。 |
| “Service 就是微服务” | 它是进程内对象，按名字放在 `app.services`，没有网络边界。 |
| “Session 就是聊天记录” | 它是带版本和序号的事件日志，还负责恢复和 uncertain 协议。 |
| “Plugin 就是一个 pip 包” | 它是运行时里的生命周期单元，可以只由几十行 Python 构成。 |
| “Context 就是对话上下文” | 插件里的 `ctx` 是 Scope；对话上下文由 `context_view()` 处理。 |
| “Agent Loop 就是 asyncio 的 loop” | 前者是模型—工具迭代，后者是协程调度器。 |

---

## 七、一句话心法

> **Harness 管生死，Loop 管决策，Registry 管执行，Policy 管许可，Session 管记忆，Service 管共享，Event 管观察。**

先抓住这七个角色的职责边界，再去抠 `AsyncExitStack`、取消语义、流式拼接这些细节，就不会在 `harness.py` 里迷路。

## 继续阅读

- [原生 Harness 使用指南](lean-harness.md)
- [插件生命周期与可替换 Loop](plugin-lifecycle.md)
- [模型连接与凭据](model-connections.md)
- [旧版迁移](legacy-migration.md)
- [源码统计脚本](../scripts/source_stats.py)
