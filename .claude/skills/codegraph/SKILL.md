---
name: codegraph
description: Code intelligence and knowledge graph for any codebase. Use when searching symbols, tracing call chains, analyzing impact of changes, building task context, or finding affected tests. Requires codegraph CLI (npm i -g @codegraph/cli).
---

# CodeGraph

CodeGraph builds a persistent knowledge graph of a codebase — symbols, call edges, file structure — and exposes it via CLI or MCP server.

## When to Activate

- 需要搜索符号定义、调用者、被调用者
- 分析修改某个函数/类的影响范围
- 为任务自动构建上下文（替代手动 grep）
- 查找受变更影响的测试文件
- 作为 MCP server 给 AI 助手提供代码智能

## Prerequisites

```bash
# 检查是否已安装
which codegraph

# 未安装时
npm install -g @codegraph/cli
```

## Project Lifecycle

### 初始化（首次使用）

```bash
codegraph init [path]          # 在项目目录初始化 .codegraph/
codegraph index [path]         # 全量索引所有文件
codegraph status [path]        # 查看索引状态和统计
```

### 日常同步

```bash
codegraph sync [path]          # 增量同步自上次索引以来的变更
```

### 清理

```bash
codegraph uninit [path]        # 删除 .codegraph/ 目录
codegraph unlock [path]        # 清除卡住的锁文件
```

## Core Commands

### query — 搜索符号

```bash
codegraph query <search>
codegraph query <search> -l 20          # 最多返回 20 条
codegraph query <search> -k function    # 只搜函数
codegraph query <search> -k class       # 只搜类
codegraph query <search> -j             # JSON 输出
codegraph query <search> -p /path/to/project
```

### context — 为任务构建上下文

```bash
codegraph context "implement pagination for user list"
codegraph context "fix auth token refresh" -n 30 -c 5
codegraph context "add rate limiting" --no-code    # 只要结构，不要代码块
codegraph context <task> -f json                   # JSON 格式
```

选项：
- `-n` / `--max-nodes`：最多包含节点数（默认 50）
- `-c` / `--max-code`：最多包含代码块数（默认 10）

### callers — 谁调用了这个符号

```bash
codegraph callers <symbol>
codegraph callers "UserService.authenticate" -l 30
codegraph callers <symbol> -j
```

### callees — 这个符号调用了谁

```bash
codegraph callees <symbol>
codegraph callees "processPayment" -l 20
```

### impact — 修改某符号的影响范围

```bash
codegraph impact <symbol>
codegraph impact "DatabaseConnection.query" -d 3   # 遍历深度 3（默认 2）
codegraph impact <symbol> -j
```

### affected — 找到受变更文件影响的测试

```bash
codegraph affected src/auth/token.ts src/auth/session.ts
codegraph affected --stdin < changed_files.txt
codegraph affected -d 3 -f "**/*.test.ts"          # 自定义测试文件 glob
codegraph affected -q                               # 只输出文件路径
```

### files — 查看项目文件结构

```bash
codegraph files
codegraph files -p /path/to/project
```

## MCP Server Mode

将 codegraph 作为 MCP server 挂载给 AI 助手：

```bash
# 自动安装到 Claude Code / Cursor / Codex CLI 等
codegraph install

# 手动配置（~/.claude.json）
```

```json
"codegraph": {
  "command": "codegraph",
  "args": ["serve"],
  "env": {}
}
```

```bash
codegraph uninstall    # 从所有 agent 移除
```

## Workflow Patterns

### 理解陌生代码库

```bash
codegraph init && codegraph index
codegraph status                          # 确认索引完成
codegraph context "how does auth work"    # 获取任务相关上下文
codegraph query "authenticate" -k function
codegraph callers "authenticate"
```

### 安全重构前的影响分析

```bash
codegraph sync                            # 确保索引最新
codegraph impact "OldService.method" -d 3
codegraph callers "OldService.method"
# 根据输出决定重构范围
```

### CI 中精准运行受影响测试

```bash
git diff --name-only HEAD~1 | codegraph affected --stdin -q | xargs pytest
# 或
git diff --name-only HEAD~1 | codegraph affected --stdin -q | xargs npx jest
```

### 为 AI 任务自动注入上下文

```bash
# 在 prompt 前先获取上下文
CONTEXT=$(codegraph context "add retry logic to HTTP client" -c 8)
# 将 $CONTEXT 拼入 prompt
```

## Tips

- `.codegraph/` 应加入 `.gitignore`，索引是本地产物
- 大型 monorepo 用 `-p` 指定子目录，避免全量索引
- `sync` 比 `index` 快得多，日常用 `sync`，CI 首次用 `index`
- `impact -d 1` 只看直接调用者，`-d 3` 看三层传播
- `context` 输出可直接粘贴进 prompt，节省手动 grep 时间
