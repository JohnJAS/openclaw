---
name: mini-agent
description: 委托编码任务给 Mini-Agent。适用于：构建新功能、代码重构、文件操作、执行 shell 命令等。需要 Mini-Agent 服务器运行在 ws://localhost:8765。使用场景：用户明确要求使用 mini-agent，或需要执行复杂编码任务时。
metadata:
  {
    "openclaw": {
      "emoji": "🤖",
      "requires": { "bins": ["mini-agent-exec"] }
    }
  }
---

# Mini-Agent Skill

委托编码任务给 Mini-Agent 执行。Mini-Agent 是一个具备文件操作、Bash 执行、MCP 工具和 Skills 能力的智能代理。

## 前提条件

Mini-Agent WebSocket 服务器必须正在运行：

```bash
mini-agent-server
```

默认监听 `ws://localhost:8765`。

## 使用方式

```bash
# 基本用法 - 执行单个任务
mini-agent-exec "Your task description here"

# 指定工作目录
mini-agent-exec --workspace /path/to/project "Your task"

# 流式输出（实时显示进度）
mini-agent-exec --stream "Write a Python script"

# 详细模式（显示思考和工具调用）
mini-agent-exec --verbose "Refactor the codebase"
```

## 参数说明

| 参数 | 说明 |
|------|------|
| `--workspace, -w` | 工作目录路径 |
| `--stream, -s` | 流式输出，实时显示进度 |
| `--verbose, -v` | 详细模式，显示思考和工具调用 |
| `--timeout, -t` | 超时时间（秒，默认 300） |
| `--url` | WebSocket 服务器地址（默认 ws://localhost:8765） |

## 典型使用场景

### 1. 文件操作

```bash
mini-agent-exec "Create a config.yaml file with database settings"
mini-agent-exec --workspace ./myproject "Add a README.md with project documentation"
```

### 2. 代码重构

```bash
mini-agent-exec --workspace ./src "Refactor the auth module to use async/await"
mini-agent-exec "Convert the JavaScript code to TypeScript"
```

### 3. Shell 命令执行

```bash
mini-agent-exec "List all Python files and count their lines"
mini-agent-exec "Set up a new Python project with venv and requirements.txt"
```

### 4. 复杂任务

```bash
mini-agent-exec --stream --verbose "Build a REST API for user management with FastAPI"
```

## 注意事项

1. **服务器必须运行**: 确保在另一个终端中启动了 `mini-agent-server`
2. **工作目录**: 使用 `--workspace` 指定正确的工作目录
3. **超时**: 长任务可能需要增加 `--timeout` 参数
4. **错误处理**: 如果连接失败，检查服务器是否运行在正确的端口

## 与其他 Skills 的关系

- 对于简单的文件编辑，优先使用内置的 edit/write 工具
- 对于需要 Mini-Agent 特定能力（如 MCP 工具、特定 Skills）的任务，使用此 skill
- 可以与 `coding-agent` skill 配合使用，根据任务特点选择合适的代理