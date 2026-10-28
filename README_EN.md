# MicaCoder

MicaCoder is a lightweight Python coding agent for local repositories. It is in early development: the current focus is a readable, extensible agent runtime, with a fuller workflow for understanding code, editing files, and verifying changes planned next.

## Current capabilities

- **Code operations:** Read, search, create, and edit workspace files, and run Shell commands.
- **Model and tool loop:** Connect a model, execute requested tool calls, and stream output.
- **Execution policy:** Choose `readonly`, `confirm`, or `trusted`; `confirm` is the default.
- **Sessions and extensions:** Session persistence and recovery, Skills, plugins, and optional MCP integration.

These capabilities describe the current code, which is not a stable release. Shell commands, plugins, and MCP tools run on the host; confirmation prompts do not provide sandbox isolation.

## Quick start

Python 3.11 or newer is required. A virtual environment is recommended:

```bash
git clone https://github.com/HuffyKitty/MicaCoder.git
cd MicaCoder
python -m pip install -e ".[credentials]"
miniagent --connect
miniagent
```

After connecting a model, enter a task in the interactive CLI. For a single read-only task:

```bash
miniagent --policy readonly --prompt "Read this project and find its CLI entry point and tests"
```

**Naming:** The repository is MicaCoder, while the current Python package and CLI are still named `miniagent`. Renaming them and providing a migration path are future work.

## Python API

```python
from miniagent import Harness, Policy
from miniagent.plugins import coding_tools

app = Harness(model="provider:model-id", policy=Policy(mode="readonly"))
app.use(coding_tools())
print(app.run("Summarize the structure of this repository"))
```

Replace `provider:model-id` with your model and configure its credential first. See [model connections](docs/model-connections.md), [plugins and API](docs/lean-harness.md), and the [source reading guide](docs/reading-guide.md).

## Possible next steps

1. **Unify project naming:** Migrate the package, CLI, configuration, and docs to MicaCoder, with guidance for existing commands.
2. **Complete the coding workflow:** Connect task planning, code discovery, patch review, test verification, and result summaries.
3. **Improve change safety:** Strengthen sensitive-file protection, command authorization, recovery, and rollback while documenting host-execution boundaries.
4. **Improve usability:** Simplify first-time model setup, error messages, session recovery, and terminal output.
5. **Build reproducible evaluations:** Use representative tasks on real repositories to measure success, time, and cost before expanding scope.

These are possible directions, not completed features.

## Development and license

Run the offline tests:

```bash
python -m pytest tests -q -m "not live"
```

Licensed under the [Apache License 2.0](LICENSE).
