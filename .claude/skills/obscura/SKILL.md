---
name: obscura
description: >
  使用 obscura 无头浏览器进行网页抓取、截图、PDF 导出、JS 执行、
  批量爬取、CDP/Puppeteer/Playwright 对接和 MCP 浏览器服务。当用户要求抓取/爬取网页、
  获取页面内容或渲染后的 HTML、对网页截图、导出 PDF、执行页面 JS、提取链接/cookie、
  批量抓 URL、绕过基础反爬（stealth）、给 MCP 客户端提供浏览器能力时使用本 skill。
  也适用于用户提到 obscura、无头浏览器、headless browser、替代 headless Chrome 的场景。
---

# Obscura 无头浏览器使用指南

Obscura 是 Rust 编写的无头浏览器引擎（自带渲染，无需 Chrome），内置 V8 跑真实 JS，
支持 Chrome DevTools Protocol（CDP），可作为 Puppeteer/Playwright 的 drop-in 替代。
开始前先 `which obscura` 确认已安装；未安装则见文末「安装」一节（安装前须征得用户同意）。

## 选哪个子命令

| 需求 | 命令 |
|---|---|
| 抓一个页面：读内容 / 截图 / 执行 JS / 提取 | `obscura fetch` |
| 批量并行抓多个 URL 并渲染 | `obscura scrape` |
| 供 Puppeteer/Playwright/CDP 客户端连接 | `obscura serve` |
| 给 MCP 客户端（Claude Desktop/Code 等）提供浏览器工具 | `obscura mcp` |

## obscura fetch —— 单页抓取

```bash
# 读页面正文（agent 读网页优先用 markdown，比 html 省 token）
obscura fetch https://example.com --dump markdown

# 其他 dump 格式
--dump html        # 渲染后 HTML（默认）
--dump text        # 纯文本
--dump links       # 所有 <a href>，每行一个 URL
--dump assets      # 每行一个 JSON，列出页面引用的全部子资源（script/img/iframe 及 fetch()/XHR）
--dump original    # 原始 HTTP 响应体（不经浏览器/JS 层，二进制安全；抓图片/JSON/JS/CSS 用这个）
--dump cookies     # cookie 罐里全部 cookie 的 JSON 数组（含 document.cookie 看不到的 HttpOnly）

# CSS 选择器缩小范围
obscura fetch https://example.com --selector "div.main" --dump text

# 执行 JS，结果以 JSON 打印
obscura fetch https://example.com -e "document.title"

# 截图 PNG（默认 1280x720 视口，可与 --eval 组合：先跑 JS 再截图，常用于滚动）
obscura fetch https://example.com -s page.png
obscura fetch https://example.com -e "window.scrollTo(0,1200)" -s scrolled.png

# 等待与超时
--wait 3                 # 显式固定延迟秒数；不传则自适应等待页面静默（5s 上限）
--timeout 30             # 导航超时（默认 30s）
--wait-until load        # domcontentloaded | load | networkidle2 | networkidle0（默认 load）

# 其他
-o out.md                # 写文件而非 stdout
-q                       # 静默日志
--user-agent "..."       # 覆盖 UA（默认模拟近期 Chrome）

# 批量模式：文件/stdin 读 URL（每行一个，# 开头跳过）
# 注意：--file 批量是原始抓取（--dump original），要渲染的批量输出请用 scrape
obscura fetch --file urls.txt --concurrency 5
cat urls.txt | obscura fetch --file -
```

## obscura scrape —— 批量并行抓取（带渲染）

```bash
# 对每个 URL 执行 JS 表达式，JSON 输出（默认并发 10，单 URL 超时 60s）
obscura scrape https://a.com https://b.com -e "document.title"
obscura scrape -e "document.querySelectorAll('.price').length" --concurrency 20 \
  https://shop1.example.com https://shop2.example.com

# URL 从 stdin 读
cat urls.txt | obscura scrape - --eval "document.title"
```

依赖同目录的 `obscura-worker`。

## obscura serve —— CDP 服务器（Puppeteer/Playwright 对接）

```bash
obscura serve --port 9222     # 默认 ws://127.0.0.1:9222
```

客户端连接（注意包名/方法，别用错）：

```js
// Puppeteer：用 puppeteer-core（不是 puppeteer，后者会下载 Chrome）
const browser = await puppeteer.connect({ browserWSEndpoint: 'ws://127.0.0.1:9222' });

// Playwright：用 connectOverCDP（不是 connect，协议不同）
const browser = await chromium.connectOverCDP('ws://127.0.0.1:9222');
const page = await (browser.contexts()[0] || await browser.newContext()).newPage();
```

支持：goto/reload/goBack、evaluate、click/type/fill、waitForSelector/waitForFunction、
cookies、setRequestInterception、exposeFunction、screenshot（含 fullPage）、pdf（raster 输出）、
raw CDP `Page.startScreencast`（活动驱动帧，需对每帧 `Page.screencastFrameAck`）。
`waitUntil` 默认 domcontentloaded。

其他 serve 选项：`--workers N`（多 worker 进程，建议每核一个）、`--max-connections`（默认 128）、
`--allow-file-access`（允许 file:// 导航，默认关）、`--font-dir`、`-q`。

## obscura mcp —— 作为 MCP 服务器

```bash
obscura mcp                      # stdio（默认），配 Claude Desktop/Claude Code
obscura mcp --http --port 3000   # HTTP 传输（无内置鉴权！暴露到 loopback 外必须加防护）
```

HTTP 传输安全：绑定非 127.0.0.1 时务必设 `OBSCURA_MCP_ALLOWED_ORIGINS`（逗号分隔 Origin 白名单，
未列出的浏览器来源请求 403）并置于反向代理/网络隔离之后。

MCP 工具保持一个活动会话：先 `browser_navigate`，再 snapshot/extract/click/evaluate 等。
工具含导航、snapshot/markdown/links/extract、interactive elements/表单检测、点击/填表/按键/滚动、
等待、JS 执行、网络请求与 console 诊断、cookie/storage、多标签页，渲染构建另有
`browser_screenshot`（返回 image/png）与 `browser_pdf`。

## Stealth / 代理 / 反指纹

```bash
obscura fetch https://example.com --stealth          # 全局 flag，fetch/serve/scrape/mcp 通用
obscura fetch https://example.com --proxy http://user:pass@proxy:8080
obscura fetch https://example.com --proxy socks5://proxy:1080
```

- `--stealth`：浏览器一致的 TLS 指纹（ClientHello/ALPN/密码套件）+ 追踪器/指纹端点拦截
- 能对付：查 TLS 指纹或 UA 的基础 bot 检测；对付不了：Cloudflare 交互挑战、Datadome/Akamai
  主动挑战、CAPTCHA、IP 限速（后者用代理解决）
- 身份一致性：默认单一稳定浏览器 profile；`OBSCURA_PROFILE=2` 固定、`OBSCURA_ROTATE_PROFILE=1`
  每 context 随机（注意：固定了代理区域/TLS 时不要开轮换）
- `OBSCURA_TIMEZONE=America/New_York`（默认 Europe/Berlin，应与出口 IP 区域一致）
- `OBSCURA_GEOLOCATION="40.7128,-74.0060"`（应与 timezone/代理一致）

## 会话持久化（cookies + localStorage）

```bash
# 同一 --storage-dir 的后续调用带着前次的登录态
obscura fetch https://example.com --storage-dir ./obscura-data
obscura serve --storage-dir ./session-1     # 登录一次，之后所有运行复用
obscura serve --port 9222 --storage-dir ./identity-a   # 多身份隔离
```

目录内有 `cookies.json` 和 `localStorage/<origin>.json`，格式稳定可 `jq` 查看。
清理状态：`rm -rf ./obscura-data`。写入时机：进程干净退出、每次导航完成、CDP setCookie/deleteCookies。

## 关键环境变量

| 变量 | 作用 | 默认 |
|---|---|---|
| `OBSCURA_NAV_TIMEOUT_MS` | 单次导航硬上限 | 30000 |
| `OBSCURA_SCRIPT_DEADLINE_MS` | 页面脚本执行阶段预算（重 SPA 需调大） | 30000 |
| `OBSCURA_CDP_COMMAND_TIMEOUT_MS` | CDP 单命令死线（0 禁用） | 60000 |
| `OBSCURA_FETCH_TIMEOUT_MS` | 页内 fetch()/XHR/模块加载超时 | 30000 |
| `OBSCURA_NAV_CHAIN_LIMIT` | 导航链长度（SSO 多跳需调大） | 10 |
| `OBSCURA_PROXY` | worker 未给 `--proxy` 时的默认代理 | — |
| `OBSCURA_ALLOW_PRIVATE_NETWORK` | 见下方 SSRF 说明 | 关 |

V8 参数用 `--v8-flags "--max-old-space-size=4096"`（追加在默认值后）。日志用 `RUST_LOG=obscura=debug`。

## 高频坑（agent 必读）

1. **访问 localhost/内网默认被 SSRF 防护拦截**。调试本地服务必须加
   `--allow-private-network`（或 `OBSCURA_ALLOW_PRIVATE_NETWORK=1`），对 DNS 解析结果也校验。
2. **不读 `HTTP_PROXY`/`HTTPS_PROXY`/`NO_PROXY`**，只能用 `--proxy` 或 `OBSCURA_PROXY`。
3. `fetch --file` 批量是**原始抓取**（不过渲染层）；要渲染的批量用 `scrape`。
4. 多页共享一个 V8 isolate，某页 CPU 密集 JS 会串行阻塞其他页。
5. PDF 是 raster 输出：文字不可选中/搜索，无 tagged PDF/页眉页脚。
6. service worker、原生媒体播放、部分 Web API、长尾 CSS 尚不完整（相对 Chromium）。
7. 截图/PDF/MCP 视觉工具需要 render 构建（官方预编译二进制已含；源码构建需 `--features render`）。
8. 常见组合：`--quiet` 压日志；`-e "..."` 先执行 JS（滚动/展开）再 `--dump`/`-s`。

## 典型组合示例

```bash
# 抓 JS 渲染的 SPA 内容为 markdown
obscura fetch https://spa.example.com --dump markdown --wait-until networkidle0

# 反爬站点：stealth + 代理 + 持久会话
obscura fetch https://target.example.com --stealth \
  --proxy http://user:pass@proxy:8080 --storage-dir ./sess --dump markdown

# 本地开发调试
obscura fetch http://localhost:3000 --allow-private-network --dump html

# 生产部署（Docker）
docker run -d --name obscura -p 127.0.0.1:9222:9222 h4ckf0r0day/obscura
```

## 安装（仅在用户明确同意后才执行）

仅当 `which obscura` 找不到命令时才需要安装。**先向用户说明要做什么并征得同意，再动手**——
不要未经确认就下载文件或写入用户的 bin 目录。

预编译二进制（推荐，无需 Chrome/Node.js 依赖）：

```bash
curl -LO https://github.com/h4ckf0r0day/obscura/releases/latest/download/obscura-$(uname -m)-linux.tar.gz
tar xzf obscura-$(uname -m)-linux.tar.gz
# 将解压出的 obscura 和 obscura-worker 放入 PATH（如 ~/.local/bin），两者必须在同一目录
```

- 归档名按平台：x86_64-linux / aarch64-linux / aarch64-macos / x86_64-macos（Windows 走 Releases 页 .zip）
- 也可用 Docker：`docker run -d --name obscura -p 127.0.0.1:9222:9222 h4ckf0r0day/obscura`
- 源码构建（Rust 1.75+，首次约 5 分钟）：`cargo build --release -p obscura-cli --bins --features render`
