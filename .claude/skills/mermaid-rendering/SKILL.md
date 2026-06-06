---
name: mermaid-rendering
description: Use when 输出 Mermaid 图、需要终端可复制的 ASCII/Unicode 图，或需要从同一 Mermaid 源同时导出 SVG 与文本图时。
---

# 用 beautiful-mermaid 渲染 Mermaid

在最终回答包含 Mermaid 图时，优先使用 `beautiful-mermaid` 从同一份 Mermaid 源**实际渲染**出文本图与 SVG。

它已经同时提供：

- `renderMermaidASCII()`：终端或回答里的 ASCII/Unicode 文本图
- `renderMermaidSVG()`：SVG 字符串
- `renderMermaidSVGAsync()`：异步 SVG 包装
- `THEMES` / `fromShikiTheme()`：现成主题与 Shiki 主题映射

不要再默认拆成 `mermaid-ascii + mmdc` 双工具链；只有在 `beautiful-mermaid` 当前环境不可用，或用户明确要求别的工具时，才降级。

## 何时使用

当你准备输出以下任意内容时，使用本 skill：

- `mermaid`
- flowchart / graph TD / graph LR
- sequenceDiagram
- stateDiagram
- classDiagram
- erDiagram
- xychart
- 任何需要架构图、流程图、调用链图、状态图的 Mermaid 输出
- 任何“既要终端文本图，又要 SVG”的 Mermaid 渲染需求

如果最终不会输出 Mermaid，也不需要 Mermaid 衍生图，则不要使用本 skill。

## 输出契约

最终回答包含 Mermaid 时，默认输出顺序必须是：

1. fenced `mermaid` 代码块
2. fenced `text` 代码块，对应 **实际渲染** 的文本图

如果用户明确要“ASCII”，文本图必须来自：

```typescript
renderMermaidASCII(source, { useAscii: true, colorMode: 'none' })
```

注意：`renderMermaidASCII()` 默认是 **Unicode 线框图**，不是纯 ASCII。用户说“ASCII”时，不能省略 `useAscii: true`。

如果用户还要 SVG：

- 用 `renderMermaidSVG()` 生成真实 SVG
- 在回答中给出导出方式、示例代码或生成文件路径
- 不要手写、伪造或臆测 SVG 内容

## 首选 API

| 需求 | API |
| --- | --- |
| 纯 ASCII 文本图 | `renderMermaidASCII(source, { useAscii: true, colorMode: 'none' })` |
| Unicode 文本图 | `renderMermaidASCII(source, { useAscii: false, colorMode: 'none' })` |
| SVG 字符串 | `renderMermaidSVG(source, THEMES['github-light'])` |
| 异步 SVG | `await renderMermaidSVGAsync(source, opts)` |
| 沿用编辑器主题 | `renderMermaidSVG(source, fromShikiTheme(theme))` |

默认优先：

- 文本图：`colorMode: 'none'`，保证回答可复制、无 ANSI 污染
- SVG：`THEMES['github-light']` 作为保守默认主题；除非用户明确要深色或项目已有主题映射

## 安装方式

优先复用当前项目已有依赖；如果当前仓库未安装，再补装：

```bash
npm install beautiful-mermaid
# 或
bun add beautiful-mermaid
# 或
pnpm add beautiful-mermaid
```

## 最小示例

Mermaid 源文件：`diagram.mmd`

```text
graph TD
    A[Client] --> B[API]
    B --> C[(DB)]
```

实际渲染 ASCII 与 SVG：

```typescript
import { readFileSync, writeFileSync } from 'node:fs'
import { renderMermaidASCII, renderMermaidSVG, THEMES } from 'beautiful-mermaid'

const source = readFileSync('diagram.mmd', 'utf8')

const ascii = renderMermaidASCII(source, {
  useAscii: true,
  colorMode: 'none',
})

const svg = renderMermaidSVG(source, THEMES['github-light'])

console.log(ascii)
writeFileSync('diagram.svg', svg)
```

如果只想快速得到 SVG 字符串：

```typescript
import { renderMermaidSVG } from 'beautiful-mermaid'

const svg = renderMermaidSVG(`
graph TD
    A[Start] --> B{Decision}
    B -->|Yes| C[Action]
    B -->|No| D[End]
`)
```

## 执行要求

1. Mermaid 源码必须作为单一事实来源。
2. 先写 Mermaid 源，再调用 `beautiful-mermaid` 实际渲染。
3. 用户要 ASCII 时，必须显式传 `useAscii: true`。
4. 回答里要放可复制文本图时，必须传 `colorMode: 'none'`。
5. 需要 SVG 时，优先用 `renderMermaidSVG()`；不要额外引入第二套默认渲染链。
6. Mermaid 内容一旦修改，文本图与 SVG 都必须重新渲染。
7. 不要手写或伪造 ASCII、Unicode 或 SVG 渲染结果。
8. 如果本地无法渲染，必须明确说明原因，不能假装已生成。

## 常见误区

| 误区 | 正确做法 |
| --- | --- |
| “先手写一个 ASCII 差不多就行” | 不允许。必须调用 `renderMermaidASCII()` 实际渲染。 |
| “默认输出就是 ASCII” | 错。默认是 Unicode；要 ASCII 必须 `useAscii: true`。 |
| “SVG 还是继续用另一个 CLI 更顺手” | 默认不这样做。优先单用 `beautiful-mermaid`，减少双工具链漂移。 |
| “回答里的文本图带 ANSI 颜色更好看” | 默认不用。回答里优先 `colorMode: 'none'`，保证纯文本可复制。 |

## Red Flags

- 想手写 ASCII 图
- 想继续把 ASCII 和 SVG 分给两套默认工具
- 忘了 `useAscii: true` 却声称输出的是 ASCII
- 文本图里混入 ANSI 颜色码
- Mermaid 已改动，但没重新渲染衍生图

出现这些情况，停止输出，重新渲染。