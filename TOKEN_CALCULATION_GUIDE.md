# Token 计算和调试指南

本指南将帮助你准确计算文本和图像的 token 数，在发送给 LLM 之前就能预估，避免报错。

---

## 目录
1. [为什么 token 计算会有偏差？](#1-为什么-token-计算会有偏差)
2. [文本 Token 准确计算方法](#2-文本-token-准确计算方法)
3. [图像 Token 准确计算方法](#3-图像-token-准确计算方法)
4. [完整的预算公式](#4-完整的预算公式)
5. [调试技巧](#5-调试技巧)
6. [配置建议](#6-配置建议)

---

## 1. 为什么 token 计算会有偏差？

### 常见原因

| 原因 | 说明 |
|------|------|
| **Tokenizer 不同** | 不同模型用不同的 tokenizer（GPT-4、Claude、DeepSeek 都不一样） |
| **估算 vs 实际** | nanobot 用的是估算，不是实际调用该模型的 tokenizer |
| **图像没算进去** | 图像占用大量 token，但估算可能没包含 |
| **工具定义没算全** | MCP 工具的定义也占用 token |
| **System Prompt 没算全** | AGENTS.md、SOUL.md 等 bootstrap 文件 |

---

## 2. 文本 Token 准确计算方法

### 2.1 使用对应模型的 Tokenizer

要准确计算，必须用该模型官方的 tokenizer：

| 模型 | Tokenizer 安装 |
|------|---------------|
| **OpenAI GPT 系列** | `pip install tiktoken` |
| **Anthropic Claude 系列** | `pip install anthropic` |
| **DeepSeek** | `pip install transformers` |

#### OpenAI GPT 系列计算示例

```python
import tiktoken

def count_openai_tokens(text: str, model: str = "gpt-4o") -> int:
    """计算 OpenAI 模型的 token 数"""
    enc = tiktoken.encoding_for_model(model)
    return len(enc.encode(text))

# 测试
text = "你好，这是一段测试文本。"
tokens = count_openai_tokens(text, model="gpt-4o")
print(f"Token 数: {tokens}")
```

#### Anthropic Claude 系列计算示例

```python
from anthropic import Anthropic

def count_anthropic_tokens(text: str) -> int:
    """计算 Claude 模型的 token 数"""
    client = Anthropic()
    # Claude 不直接暴露 tokenizer，但可以用这个估算
    # 注意：这只是近似值
    return len(text) // 4  # 粗略估算：1 token ≈ 4 个字符

# 或者用第三方库（更准确）
# pip install anthropic-tokenizer
try:
    from anthropic_tokenizer import get_tokenizer
    tokenizer = get_tokenizer()
    tokens = len(tokenizer.encode(text))
except ImportError:
    print("请安装: pip install anthropic-tokenizer")
```

### 2.2 计算整个对话历史

```python
def count_messages_tokens(messages: list[dict], model: str = "gpt-4o") -> int:
    """计算整个消息列表的 token 数"""
    enc = tiktoken.encoding_for_model(model)
    total = 0

    for msg in messages:
        # role
        total += len(enc.encode(msg["role"]))
        # content
        if isinstance(msg["content"], str):
            total += len(enc.encode(msg["content"]))
        elif isinstance(msg["content"], list):
            # 多模态内容
            for item in msg["content"]:
                if item["type"] == "text":
                    total += len(enc.encode(item["text"]))
                elif item["type"] == "image_url":
                    # 图像单独计算（见下一节）
                    total += estimate_image_tokens(item["image_url"]["url"])
    return total
```

---

## 3. 图像 Token 准确计算方法

### 3.1 OpenAI 图像 Token 计算

OpenAI 的计算方式：
- 低分辨率：512×512 = **85 tokens**
- 高分辨率：按瓦片计算，每个瓦片 170 tokens，再加 85 tokens 基础

```python
def estimate_openai_image_tokens(
    width: int,
    height: int,
    detail: str = "auto"  # "low", "high", "auto"
) -> int:
    """
    估算 OpenAI 图像的 token 数

    Args:
        width: 图像宽度
        height: 图像高度
        detail: 细节模式

    Returns:
        token 数量
    """
    if detail == "low":
        return 85

    # 高分辨率模式
    # 1. 先缩放到最大边 2048
    max_dim = max(width, height)
    if max_dim > 2048:
        scale = 2048 / max_dim
        width = int(width * scale)
        height = int(height * scale)

    # 2. 再缩放到最短边 768
    min_dim = min(width, height)
    if min_dim > 768:
        scale = 768 / min_dim
        width = int(width * scale)
        height = int(height * scale)

    # 3. 计算瓦片数（512×512 为一个瓦片）
    tiles_w = (width + 511) // 512  # 向上取整
    tiles_h = (height + 511) // 512
    total_tiles = tiles_w * tiles_h

    # 4. 计算 token
    return 170 * total_tiles + 85

# 测试
print(estimate_openai_image_tokens(1024, 1024, detail="high"))  # 765
print(estimate_openai_image_tokens(2048, 2048, detail="high"))  # 170*4 +85 = 765
print(estimate_openai_image_tokens(512, 512, detail="low"))   # 85
```

### 3.2 Anthropic 图像 Token 计算

Claude 的图像 token 计算：
- 每个图像固定 **1600 tokens**（大约）
- 或者按像素估算：`(width * height) / 750`

```python
def estimate_anthropic_image_tokens(width: int, height: int) -> int:
    """估算 Claude 图像的 token 数"""
    # 方法 1：固定 1600
    # return 1600

    # 方法 2：按像素估算
    return (width * height) // 750
```

---

## 4. 完整的预算公式

### 4.1 nanobot 实际预算

```python
# 来自 nanobot/memory.py
context_window_tokens = 65536  # 你的配置
max_completion_tokens = 8192    # 默认
SAFETY_BUFFER = 1024            # 固定

budget = (
    context_window_tokens
    - max_completion_tokens
    - SAFETY_BUFFER
)
# budget = 65536 - 8192 - 1024 = 56320
```

### 4.2 你的输入实际占用

```python
def calculate_total_input_tokens(
    system_prompt: str,
    messages: list[dict],
    tool_definitions: list[dict],
    model: str = "gpt-4o"
) -> int:
    """
    计算完整的输入 token 数

    Returns:
        总 token 数
    """
    total = 0

    # 1. System Prompt
    total += count_openai_tokens(system_prompt, model)

    # 2. 消息历史
    total += count_messages_tokens(messages, model)

    # 3. 工具定义
    for tool in tool_definitions:
        total += count_openai_tokens(str(tool), model)

    return total
```

---

## 5. 调试技巧

### 5.1 启用 LLM 请求打印

我们之前添加的功能！在启动前设置环境变量：

```bash
# macOS/Linux
export NANOBOT_PRINT_LLM_REQUESTS=1
export NANOBOT_PRINT_FULL_REQUEST=1
uv run python nanobot_gradio.py

# Windows CMD
set NANOBOT_PRINT_LLM_REQUESTS=1
set NANOBOT_PRINT_FULL_REQUEST=1
uv run python nanobot_gradio.py

# Windows PowerShell
$env:NANOBOT_PRINT_LLM_REQUESTS=1
$env:NANOBOT_PRINT_FULL_REQUEST=1
uv run python nanobot_gradio.py
```

这样会打印出：
- 请求的 token 估算
- 完整的请求内容（如果设置了 `NANOBOT_PRINT_FULL_REQUEST`）

### 5.2 用 `/status` 命令查看

在聊天中输入 `/status`，会显示：
- 当前 token 估算值
- context window 总量
- 使用百分比

### 5.3 手动检查 Session 文件

Session 文件位置：`~/.nanobot/workspace/.sessions/`

打开 JSON 文件，查看 `messages` 字段，自己估算一下。

---

## 6. 配置建议

### 6.1 保守配置（推荐）

```yaml
agents:
  defaults:
    # 设为模型限制的 70-80%
    context_window_tokens: 32768    # 如果模型是 65536
    # 或者 49152（如果模型是 65536）

    # 降低一些，给输入留更多空间
    max_tokens: 4096
```

### 6.2 不同模型的建议值

| 模型 | 官方限制 | 建议 `context_window_tokens` |
|------|---------|-----------------------------|
| GPT-4o | 128k | 98304 (75%) |
| GPT-4o mini | 128k | 98304 (75%) |
| Claude 3.5 Sonnet | 200k | 153600 (75%) |
| Claude 3 Opus | 200k | 153600 (75%) |
| DeepSeek V3 | 64k | 49152 (75%) |
| 其他 64k 模型 | 64k | 49152 (75%) |

### 6.3 如果还是报错

```yaml
agents:
  defaults:
    # 进一步降低
    context_window_tokens: 24576

    # 或者更激进
    context_window_tokens: 16384

tools:
  # 如果用了很多 MCP 工具，考虑只启用需要的
  mcp_servers:
    browser-use:
      enabled_tools:
        - "navigate"
        - "click"
        - "type"
```

---

## 快速参考：一键配置

```yaml
agents:
  defaults:
    # 保守配置，适用于大多数 64k 模型
    context_window_tokens: 32768
    max_tokens: 4096
    temperature: 0.1
```

然后启动时加上：
```bash
NANOBOT_PRINT_LLM_REQUESTS=1 uv run python nanobot_gradio.py
```
