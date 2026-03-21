---
name: commit-message
description: 根据代码变更生成规范的 commit message。当用户需要提交代码、生成提交信息、写 commit 时，应该使用此 Skill。支持 Conventional Commits 规范，自动识别变更类型。
---

# Commit Message 生成器

根据用户提供的代码变更或描述，生成符合 Conventional Commits 规范的提交信息。

## 规范标准

使用 Conventional Commits 格式：
```
<type>(<scope>): <subject>

<body>

<footer>
```

## Type 类型

- **feat**: 新功能
- **fix**: Bug 修复  
- **docs**: 文档变更
- **style**: 代码格式（不影响功能）
- **refactor**: 重构（既不是新功能也不是修 bug）
- **perf**: 性能优化
- **test**: 测试相关
- **chore**: 构建过程或辅助工具变更
- **ci**: CI 配置变更
- **revert**: 回滚提交

## 生成规则

1. **Subject**
   - 使用祈使句，现在时态
   - 不超过 50 个字符
   - 首字母小写
   - 结尾不加句号

2. **Body**（可选）
   - 解释 **为什么** 做这个变更
   - 说明 **如何** 解决问题
   - 每行不超过 72 字符
   - 与 subject 之间空一行

3. **Footer**（可选）
   - 关闭 Issue：`Closes #123`
   - 破坏性变更：`BREAKING CHANGE: 描述`

## 使用流程

当用户请求生成 commit message 时：

1. **分析变更**
   - 询问或分析变更内容
   - 判断变更类型（feat/fix/docs 等）
   - 识别影响范围（scope）

2. **生成提交信息**
   - 根据规范生成符合格式的 commit message
   - 提供完整版本（含 body）和简洁版本
   - 必要时询问用户确认

3. **输出格式**
   - 用代码块展示生成的 commit message
   - 提示用户可以直接使用：`git commit -m "message"`

## 示例

### 示例 1：新功能
**输入**：添加了用户登录功能，包括 JWT 认证

**输出**：
```
feat(auth): add JWT user authentication

Implement login endpoint with JWT token generation
Add authentication middleware for protected routes
Include token refresh mechanism

Closes #123
```

### 示例 2：Bug 修复
**输入**：修复了登录超时未处理的问题

**输出**：
```
fix(auth): handle login timeout gracefully

Add timeout handling for authentication requests
Show user-friendly error message when timeout occurs
Set 30-second timeout limit
```

### 示例 3：简洁版本
**输入**：更新了 README 文档

**输出**：
```
docs: update installation guide
```

## 注意事项

- 如果用户只提供了简单的变更描述，生成简洁版本即可
- 如果用户提供了详细的变更信息，生成完整版本（含 body）
- 当有多个变更时，优先识别主要变更类型
- 如果无法确定 type，可以询问用户确认
- 需要提供中英各一版的 commit message，方便不同语言环境的开发者使用