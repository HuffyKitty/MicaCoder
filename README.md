# MicaCoder

MicaCoder 是一个面向本地代码仓库的轻量 Python Coding Agent。项目目前处于早期开发阶段：先保留清晰、可扩展的 Agent 运行内核，再逐步完善理解代码、修改文件和验证结果的完整编程流程。

## 当前能力

- **代码操作**：读取、搜索、创建和编辑工作区文件，可通过 Shell 执行命令。
- **模型与工具循环**：连接模型，按模型的工具调用请求执行任务；支持流式输出。
- **执行控制**：提供 `readonly`、`confirm` 和 `trusted` 三种策略，默认使用 `confirm`。
- **会话与扩展**：支持会话记录与恢复、Skills、插件，以及可选的 MCP 集成。

这些能力以当前代码为准，尚未作为稳定产品发布。Shell、插件和 MCP 工具在本机运行；执行确认不提供沙箱隔离。

## 快速开始

需要 Python 3.11 或更新版本。建议在虚拟环境中运行：

```bash
git clone https://github.com/HuffyKitty/MicaCoder.git
cd MicaCoder
python -m pip install -e ".[credentials]"
miniagent --connect
miniagent
```

首次连接模型后，可以在交互界面输入任务。也可以执行一次只读任务：

```bash
miniagent --policy readonly --prompt "阅读当前项目，找出命令行入口和相关测试"
```

**命名说明**：仓库名称已改为 MicaCoder；当前 Python 包名和命令名仍为 `miniagent`。统一包名、命令名及配置迁移属于后续改进工作。

## Python API

```python
from miniagent import Harness, Policy
from miniagent.plugins import coding_tools

app = Harness(model="provider:model-id", policy=Policy(mode="readonly"))
app.use(coding_tools())
print(app.run("概述这个仓库的代码结构"))
```

将 `provider:model-id` 替换为实际模型，并先配置对应凭据。更多内容见[模型连接](docs/model-connections.md)、[插件与 API](docs/lean-harness.md)和[源码阅读指南](docs/reading-guide.md)。

## 后续方向

1. **统一项目身份**：将 Python 包、CLI、配置与文档逐步迁移到 MicaCoder 命名，并提供旧命令的迁移办法。
2. **完善编程闭环**：让任务规划、代码定位、补丁审查、测试验证和结果总结形成清晰的工作流程。
3. **加强变更安全**：改进敏感文件保护、命令授权、失败恢复与变更回滚，并明确本机执行的边界。
4. **提升使用体验**：简化首次模型连接、错误提示、会话恢复和终端输出。
5. **建立可复现评估**：用代表性的真实仓库任务衡量成功率、耗时和成本，再决定进一步扩展哪些能力。

方向会根据实际使用反馈调整，不代表这些能力已经完成。

## 开发与许可

运行离线测试：

```bash
python -m pytest tests -q -m "not live"
```

项目采用 [Apache License 2.0](LICENSE)。
