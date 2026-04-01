# Windows 从零部署指南（带 Gradio UI）

本指南将帮助你在全新的 Windows 环境上部署带 Gradio 界面的 nanobot。

---

## 目录
1. [安装 Python](#1-安装-python)
2. [安装 uv 包管理器](#2-安装-uv-包管理器)
3. [克隆项目代码](#3-克隆项目代码)
4. [安装项目依赖](#4-安装项目依赖)
5. [初始化配置](#5-初始化配置)
6. [启动 Gradio 界面](#6-启动-gradio-界面)
7. [（可选）启用 LLM 请求追踪](#7-可选启用-llm-请求追踪)

---

## 1. 安装 Python

### 1.1 下载 Python
访问 Python 官网下载页面：https://www.python.org/downloads/

点击 "Download Python 3.11.x"（推荐 3.11 或更高版本）

### 1.2 安装 Python
运行下载的安装程序，**重要**：
- ✅ 勾选 **"Add Python 3.x to PATH"**
- 点击 "Install Now"

### 1.3 验证安装
打开 **命令提示符 (CMD)** 或 **PowerShell**，运行：

```cmd
python --version
```

应该显示类似：`Python 3.11.x`

---

## 2. 安装 uv 包管理器

### 2.1 用 PowerShell 安装（推荐）
打开 **PowerShell**（不是 CMD），运行：

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

### 2.2 或者用 pip 安装
如果你已经有 pip，也可以运行：

```cmd
pip install uv
```

### 2.3 验证安装
重新打开命令行，运行：

```cmd
uv --version
```

应该显示 uv 的版本号。

---

## 3. 克隆项目代码

### 3.1 克隆仓库
```cmd
git clone https://github.com/lunaww/nanobot.git
cd nanobot
```

### 3.2 切换到功能分支
```cmd
git checkout feat/gradio-ui-and-llm-tracing
```

---

## 4. 安装项目依赖

### 4.1 安装核心依赖
```cmd
uv sync
```

### 4.2 安装 Gradio 依赖
```cmd
uv pip install gradio
```

---

## 5. 初始化配置

### 5.1 首次运行生成配置
```cmd
uv run nanobot
```

这会在 `C:\Users\你的用户名\.nanobot\` 目录下生成默认配置文件。

看到输出后，按 **`Ctrl + C`** 退出。

### 5.2 配置 LLM（可选）
编辑配置文件：`C:\Users\你的用户名\.nanobot\config.yaml`

设置你的模型和 API Key：
```yaml
agents:
  defaults:
    model: gpt-4o-mini  # 或其他模型

providers:
  - name: openai
    api_key: sk-xxxxxxxxxxxxxxxxx  # 你的 API Key
```

---

## 6. 启动 Gradio 界面

### 6.1 运行 Gradio
```cmd
uv run python nanobot_gradio.py
```

### 6.2 打开浏览器
访问：**http://localhost:7860**

---

## 7. （可选）启用 LLM 请求追踪

如果你想在控制台看到 LLM 请求参数：

### 使用 CMD：
```cmd
set NANOBOT_PRINT_LLM_REQUESTS=1
uv run python nanobot_gradio.py
```

### 使用 PowerShell：
```powershell
$env:NANOBOT_PRINT_LLM_REQUESTS=1
uv run python nanobot_gradio.py
```

### 打印完整请求参数：
```cmd
# CMD
set NANOBOT_PRINT_LLM_REQUESTS=1
set NANOBOT_PRINT_FULL_REQUEST=1
uv run python nanobot_gradio.py
```

---

## 常见问题

### Q: 提示 "git 不是内部或外部命令"
A: 安装 Git：https://git-scm.com/download/win

### Q: 提示 "uv 不是内部或外部命令"
A: 关闭并重新打开命令行窗口，或者重启电脑。

### Q: Gradio 页面打不开
A: 检查防火墙设置，确保 7860 端口没有被阻止。

---

## 快速参考（一键复制）

```cmd
# ===== 从零开始的完整命令 =====

# 1. 克隆项目
git clone https://github.com/lunaww/nanobot.git
cd nanobot
git checkout feat/gradio-ui-and-llm-tracing

# 2. 安装依赖
uv sync
uv pip install gradio

# 3. 初始化配置（按 Ctrl+C 退出）
uv run nanobot

# 4. 启动 Gradio
uv run python nanobot_gradio.py
```

然后打开浏览器访问 http://localhost:7860
