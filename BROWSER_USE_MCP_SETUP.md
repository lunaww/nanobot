# Browser-Use MCP Server 部署指南

本指南将帮助你在 nanobot 中配置和使用 browser-use MCP server，让 Agent 能够控制浏览器。

---

## 目录
1. [什么是 browser-use MCP server？](#1-什么是-browser-use-mcp-server)
2. [前置要求](#2-前置要求)
3. [安装步骤](#3-安装步骤)
4. [配置 nanobot](#4-配置-nanobot)
5. [测试使用](#5-测试使用)

---

## 1. 什么是 browser-use MCP server？

**browser-use** 是一个让 AI Agent 能够控制真实浏览器的工具。通过 MCP (Model Context Protocol)，nanobot 可以使用 browser-use 来：

- 打开网页
- 点击元素
- 填写表单
- 截取屏幕
- 提取网页内容
- ... 等等浏览器操作

---

## 2. 前置要求

| 软件 | 要求 |
|------|------|
| **Python** | ≥ 3.11 |
| **Node.js** | ≥ 18 (用于 npx) |
| **uv** | 已安装 |
| **Playwright 浏览器** | 需要安装 |

---

## 3. 安装步骤

### 3.1 安装 Playwright（如果还没有）

browser-use 需要 Playwright 来控制浏览器。

```bash
# 安装 Playwright
pip install playwright

# 安装浏览器（Chromium, Firefox, WebKit）
playwright install
```

### 3.2 安装 browser-use MCP server

有两种方式安装：

#### 方式一：通过 npx（推荐，最简单）

不需要额外安装，直接在配置里使用 `npx` 命令即可。

#### 方式二：全局安装

```bash
npm install -g @browser-use/mcp-server
```

---

## 4. 配置 nanobot

编辑你的 nanobot 配置文件：`~/.nanobot/config.yaml`（或 `.json`）

添加 `browser-use` MCP server 配置：

### 4.1 YAML 格式 (config.yaml)

```yaml
tools:
  mcp_servers:
    browser-use:
      command: "npx"
      args:
        - "-y"
        - "@browser-use/mcp-server"
      env:
        # 可选：设置浏览器类型（chromium, firefox, webkit）
        BROWSER_TYPE: "chromium"
        # 可选：是否显示浏览器窗口（true=显示，false=无头模式）
        HEADLESS: "false"
      tool_timeout: 60  # 浏览器操作可能需要更长时间
```

### 4.2 JSON 格式 (config.json)

```json
{
  "tools": {
    "mcpServers": {
      "browser-use": {
        "command": "npx",
        "args": [
          "-y",
          "@browser-use/mcp-server"
        ],
        "env": {
          "BROWSER_TYPE": "chromium",
          "HEADLESS": "false"
        },
        "toolTimeout": 60
      }
    }
  }
}
```

### 4.3 配置选项说明

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `command` | 运行命令 | `npx` |
| `args` | 命令参数 | `["-y", "@browser-use/mcp-server"]` |
| `env.BROWSER_TYPE` | 浏览器类型：`chromium`, `firefox`, `webkit` | `chromium` |
| `env.HEADLESS` | 是否无头模式：`"true"`=不显示窗口，`"false"`=显示窗口 | `"false"` |
| `tool_timeout` / `toolTimeout` | 工具超时时间（秒） | `30`，建议设为 `60` |

---

## 5. 测试使用

### 5.1 启动 nanobot

```bash
# 方式一：用 Gradio 界面
uv run python nanobot_gradio.py

# 方式二：用 CLI
nanobot agent

# 方式三：用 Gateway
nanobot gateway
```

### 5.2 测试 browser-use 功能

启动后，你可以问 Agent：

| 测试命令 | 预期效果 |
|---------|---------|
| "打开百度首页" | 浏览器打开 https://www.baidu.com |
| "搜索 '人工智能'" | 在百度搜索框输入并搜索 |
| "截取当前页面" | 截取浏览器屏幕 |
| "点击第一个搜索结果" | 点击搜索结果的第一个链接 |

### 5.3 可用的 browser-use 工具

配置成功后，nanobot 会自动注册以下工具（工具名格式：`mcp_browser_use_xxx`）：

- `mcp_browser_use_navigate` - 打开网页
- `mcp_browser_use_click` - 点击元素
- `mcp_browser_use_type` - 输入文本
- `mcp_browser_use_screenshot` - 截取屏幕
- `mcp_browser_use_get_page_content` - 获取页面内容
- `mcp_browser_use_scroll` - 滚动页面
- ... 等等

---

## 常见问题

### Q: 提示 "playwright: command not found"
A: 运行 `pip install playwright && playwright install`

### Q: 浏览器窗口不显示
A: 检查配置中 `HEADLESS` 是否设为 `"false"`（注意是字符串，不是布尔值）

### Q: 操作超时
A: 增加 `tool_timeout` 的值，比如从 30 改为 60 或 120

### Q: 如何确认 MCP server 加载成功？
A: 启动 nanobot 时，日志中会显示类似这样的信息：
```
Registered MCP tools: mcp_browser_use_navigate, mcp_browser_use_click, ...
```

---

## 快速参考（一键配置）

将以下内容复制到你的 `~/.nanobot/config.yaml` 中：

```yaml
tools:
  mcp_servers:
    browser-use:
      command: "npx"
      args: ["-y", "@browser-use/mcp-server"]
      env:
        BROWSER_TYPE: "chromium"
        HEADLESS: "false"
      tool_timeout: 60
```

然后启动 nanobot 即可！
