---
name: mini-agent
description: 创意工厂 - 将你的想法变为现实。Mini-Agent 是一个智能创意实现引擎，能够理解你的创意描述并自动生成代码、文件、脚本等实现成果。适用于：应用开发、原型验证、自动化脚本、创意原型、工具构建等场景。使用前确保 Mini-Agent 服务器已启动 (mini-agent-server)。
metadata:
  {
    "openclaw": {
      "emoji": "🎨",
      "requires": { "bins": ["mini-agent-exec"] }
    }
  }
---

# Mini-Agent 创意工厂 🏭

> 把你的创意交给 AI，让它变成现实。

Mini-Agent 是你的创意实现引擎。你只需要描述想要什么，它会自动处理所有实现细节——设计、编码、文件创建、测试，一切都在一次对话中完成。

## 快速开始

### 启动服务器并执行

```bash
# 确保服务器已启动
mini-agent-server

# 实现你的创意
mini-agent-exec "你的创意描述"
```

## 创意类型

### 🖥️ 应用开发

```bash
# Web 应用
mini-agent-exec "创建一个待办事项 Web 应用，支持添加、删除、标记完成，数据保存到本地存储"

# API 服务
mini-agent-exec "构建一个 REST API，包含用户注册、登录、JWT 认证，使用 FastAPI 和 SQLite"

# CLI 工具
mini-agent-exec "开发一个命令行工具，可以批量重命名文件，支持正则匹配和预览"
```

### 🎨 创意原型

```bash
# 游戏原型
mini-agent-exec "用 Pygame 做一个简单的贪吃蛇游戏，有分数和游戏结束界面"

# 可视化
mini-agent-exec "创建一个数据可视化仪表盘，展示实时 CPU 和内存使用情况"

# 交互演示
mini-agent-exec "做一个交互式数学函数绘图器，用户可以输入函数表达式"
```

### ⚙️ 自动化脚本

```bash
# 文件处理
mini-agent-exec "写一个脚本，扫描目录下所有图片，按拍摄日期整理到子文件夹"

# 数据处理
mini-agent-exec "创建一个 CSV 数据清洗脚本，处理缺失值、去重、格式标准化"

# 网络爬虫
mini-agent-exec "开发一个爬虫，抓取指定网站的新闻标题和摘要，保存为 JSON"
```

### 🛠️ 工具构建

```bash
# 开发工具
mini-agent-exec "创建一个代码生成器，根据数据库 schema 生成 CRUD 操作代码"

# 分析工具
mini-agent-exec "做一个代码复杂度分析工具，输出每个文件的圈复杂度报告"

# 配置工具
mini-agent-exec "创建一个项目初始化工具，支持选择框架、配置 ESLint/Prettier、生成基础结构"
```

## 高级用法

### 指定工作目录

```bash
mini-agent-exec --workspace /path/to/your/project "你的创意"
```

### 流式查看进度

```bash
mini-agent-exec --stream "构建一个完整的博客系统"
```

### 详细模式（查看思考过程）

```bash
mini-agent-exec --verbose --stream "你的复杂创意"
```

### 超时设置

```bash
# 大型项目可能需要更长超时（默认 300 秒）
mini-agent-exec --timeout 600 "创建一个微服务架构的项目"
```

## 创意描述技巧

### ✅ 好的创意描述

```
✓ 明确的目标
  "创建一个个人财务追踪应用，支持收入支出记录、分类统计、月度报告"

✓ 具体的技术栈
  "用 React + Tailwind CSS 构建一个图片画廊，支持拖拽排序和灯箱预览"

✓ 清晰的功能点
  "做一个番茄钟应用，包含 25 分钟工作时段、5 分钟休息、统计今日完成数量"
```

### ❌ 不好的创意描述

```
✗ 太模糊
  "做一个好玩的东西" → Mini-Agent 不知道你想要什么

✗ 范围太大
  "创建一个完整的电商平台" → 一次对话难以完成，建议拆分

✗ 缺少上下文
  "帮我完善它" → 需要说明具体要完善什么
```

## 创意工厂能力

Mini-Agent 创意工厂具备以下能力：

| 能力 | 描述 |
|------|------|
| 🔨 **代码生成** | 生成高质量、可运行的代码 |
| 📁 **文件操作** | 创建、读取、编辑文件和目录 |
| 🖥️ **Shell 执行** | 运行命令、安装依赖、启动服务 |
| 🔧 **MCP 工具** | 连接外部工具和服务 |
| 📚 **Skills 扩展** | 加载专业技能模块 |
| 💾 **会话记忆** | 多轮对话保持上下文 |

## 最佳实践

### 1. 从小开始

```bash
# 先做核心功能
mini-agent-exec "创建一个最简单的待办列表，只有添加和显示功能"

# 再迭代增强
mini-agent-exec --workspace ./todo-app "给待办列表添加删除和编辑功能"
```

### 2. 指定约束

```bash
mini-agent-exec "创建一个 Flask API，要求：
1. 使用 Blueprint 组织路由
2. 所有响应使用 JSON 格式
3. 包含错误处理
4. 添加基本的日志记录"
```

### 3. 提供示例

```bash
mini-agent-exec "创建一个 CLI 工具，类似下面这样的用法：
$ mytool process input.csv --output result.json
处理 CSV 文件并输出统计结果"
```

### 4. 分步实现

```bash
# Step 1: 项目结构
mini-agent-exec "初始化一个 Python 项目结构，包含 src、tests、docs 目录"

# Step 2: 核心模块
mini-agent-exec --workspace ./project "在 src/ 目录实现核心数据处理模块"

# Step 3: 测试
mini-agent-exec --workspace ./project "为数据处理模块编写单元测试"
```

## 故障排除

| 问题 | 解决方案 |
|------|----------|
| `Connection refused` | 确保已运行 `mini-agent-server` |
| `Timeout` | 增加 `--timeout` 参数 |
| 结果不符合预期 | 使用 `--verbose` 查看思考过程，优化描述 |
| 权限问题 | 确保工作目录有读写权限 |

## 开始创造

现在就试试：

```bash
mini-agent-exec "创建一个随机名言生成器，每次运行显示一句励志名言"
```

让创意变成现实，只需要一句话。🏭✨
