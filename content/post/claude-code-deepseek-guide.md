---
title: "小白教程：安装 Claude Code 并接入 DeepSeek API"
description: "手把手教你安装 Claude Code 命令行工具，配置 DeepSeek 的 API，零基础也能看懂"
date: 2026-06-11T16:00:00+08:00
slug: "claude-code-deepseek-guide"
tags: ["Claude Code", "DeepSeek", "教程", "AI"]
categories: ["技术"]
---

## 前言

Claude Code 是 Anthropic 官方推出的命令行 AI 编程助手，可以直接在你的终端里帮你写代码、读代码、重构代码。简单说，它就是你的"结对编程搭挡"。

但它默认调用的是 Anthropic 的 API，需要付费且国内网络访问不太方便。好在 **DeepSeek 提供了完全兼容 Anthropic 接口的 API 端点**，我们可以把 Claude Code 无缝切换到 DeepSeek 的模型上——速度快、价格低、国内直连。

本教程面向零基础用户，跟着一步步操作即可。

---

## 准备工作

你需要两样东西：

| 需要准备 | 说明 |
|---------|------|
| **Node.js**（>= 18） | JavaScript 运行环境，用来安装 Claude Code |
| **DeepSeek API Key** | 调用 DeepSeek 模型的密钥 |

---

## 第一步：安装 Node.js

Claude Code 通过 npm（Node.js 自带的包管理器）安装，所以需要先装 Node.js。

### Windows 用户

1. 打开浏览器访问 [https://nodejs.org](https://nodejs.org)
2. 下载左侧的 **LTS（长期支持版）**
3. 双击安装包，一路点 **Next**，直到安装完成
4. 安装完成后，按 `Win + R`，输入 `powershell` 打开终端
5. 输入以下命令验证安装：

```powershell
node --version
npm --version
```

如果能看到版本号（如 `v18.20.0` 和 `10.x.x`），说明安装成功。

### macOS / Linux 用户

建议使用 nvm 安装（这样可以方便切换版本），终端执行：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
# 重启终端后
nvm install 22
node --version
npm --version
```

---

## 第二步：获取 DeepSeek API Key

1. 打开 [https://platform.deepseek.com](https://platform.deepseek.com) 并注册/登录
2. 点击左侧 **API Keys**，然后点击 **Create API key**
3. 输入一个名称（比如 `claude-code`），点击确认
4. **复制生成的密钥**（以 `sk-` 开头），保存好——关闭页面后就看不到了

> 注意：API Key 是敏感信息，不要分享给任何人，也不要提交到 Git 仓库。

---

## 第三步：安装 Claude Code

打开终端（Windows 用 PowerShell，macOS/Linux 用 Terminal），执行：

```bash
npm install -g @anthropic-ai/claude-code
```

等待安装完成，然后验证：

```bash
claude --version
```

如果显示版本号，说明安装成功。

> 如果遇到权限错误，macOS/Linux 可以尝试 `sudo npm install -g @anthropic-ai/claude-code`，Windows 请以管理员身份运行 PowerShell。

---

## 第四步：配置 DeepSeek API

安装完成后，需要让 Claude Code 把请求发到 DeepSeek 的 API 而不是 Anthropic 的。有两种方式：

### 方式一：使用配置文件（推荐，一劳永逸）

创建配置文件 `~/.claude/settings.json`（Windows 路径为 `C:\Users\你的用户名\.claude\settings.json`）：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "sk-你的DeepSeek API Key",
    "ANTHROPIC_MODEL": "deepseek-chat",
    "CLAUDE_CODE_EFFORT_LEVEL": "max"
  }
}
```

把 `sk-你的DeepSeek API Key` 替换成第二步复制的真实密钥。

### 方式二：使用环境变量（适合临时测试）

**Windows PowerShell：**

```powershell
$env:ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
$env:ANTHROPIC_AUTH_TOKEN="sk-你的DeepSeek API Key"
$env:ANTHROPIC_MODEL="deepseek-chat"
$env:CLAUDE_CODE_EFFORT_LEVEL="max"
```

**macOS / Linux：**

```bash
export ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
export ANTHROPIC_AUTH_TOKEN=sk-你的DeepSeek API Key
export ANTHROPIC_MODEL=deepseek-chat
export CLAUDE_CODE_EFFORT_LEVEL=max
```

> 环境变量的方式只在当前终端会话有效，关闭终端后需要重新设置。

---

## 第五步：启动 Claude Code

在项目目录中打开终端，执行：

```bash
claude
```

第一次启动会显示欢迎界面。看到命令行提示符后，你就可以开始使用了。

### 验证是否配置成功

在 Claude Code 中输入：

```
请告诉我你现在使用的是哪个 API
```

如果配置正确，它会回答使用的是 DeepSeek 的 API。

---

## 常用命令速览

进入 Claude Code 后，可以试试这些：

| 命令 | 说明 |
|------|------|
| `/help` | 查看帮助 |
| `直接提问` | 比如"这个项目的入口文件在哪？" |
| `帮我写一个...` | 描述你想要的功能，它会自动生成代码 |
| `/clear` | 清除对话历史 |

---

## 进阶配置

### 使用不同模型

DeepSeek 提供多个模型，你可以在配置中指定：

| 模型 | 配置值 | 特点 |
|------|--------|------|
| deepseek-chat | `deepseek-chat` | 最新版对话模型，推荐 |
| deepseek-reasoner | `deepseek-reasoner` | 带深度推理，适合复杂问题 |

### 配置子代理模型

Claude Code 在运行时会启动子代理处理特定任务，可以单独配置：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "sk-你的API Key",
    "ANTHROPIC_MODEL": "deepseek-chat",
    "CLAUDE_CODE_SUBAGENT_MODEL": "deepseek-chat"
  }
}
```

---

## 常见问题

### Q：安装时报权限错误怎么办？

Windows 请以管理员身份运行 PowerShell；macOS/Linux 在命令前加 `sudo`。

### Q：启动时提示 "No API key found"？

检查 `settings.json` 中的 `ANTHROPIC_AUTH_TOKEN` 是否填写正确，以及文件路径是否放对。

### Q：Claude Code 启动后很慢或报网络错误？

检查是否能在浏览器中访问 `https://api.deepseek.com`。如果不能，可能需要配置代理：

```json
{
  "env": {
    "HTTP_PROXY": "http://127.0.0.1:7890",
    "HTTPS_PROXY": "http://127.0.0.1:7890"
  }
}
```

### Q：怎么卸载 Claude Code？

```bash
npm uninstall -g @anthropic-ai/claude-code
```

---

## 总结

回顾一下整个流程：

1. **安装 Node.js**
2. **申请 DeepSeek API Key**
3. **安装 Claude Code**：`npm install -g @anthropic-ai/claude-code`
4. **配置 API**：设置 `settings.json` 或环境变量
5. **启动**：在项目目录执行 `claude`

配置完成后，你就拥有了一个免费、高速、国内直连的 AI 编程助手。无论是读代码、写代码还是重构代码，都可以直接在终端里和它对话完成。

Happy coding！
