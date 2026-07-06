---
layout:     post
title:     通过 CC Switch 配置 Codex 接入第三方模型（以 GLM 为例）
subtitle:   语义化版本
date:       2026-07-06  
author:     zeroGoZhang
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - cc-switch
    - glm
    - codex
    - 语义化版本
    - 版本控制
---


# 通过 CC Switch 配置 Codex 接入第三方模型（以 GLM 为例）

## 背景

OpenAI Codex 默认只连接官方 OpenAI 模型。CC Switch 是一个本地代理工具，可以在 Codex（以及 Claude、Gemini 等）与第三方模型之间建立代理层，无需修改 Codex 自身代码即可接入 GLM、通义千问、DeepSeek 等模型。

> **版本要求**：CC Switch v3.16.0 及以上版本已将"智谱 GLM"作为 Codex 的官方预设加入。升级到最新版后，可直接在可视化界面中一键配置，无需手动填写 API Endpoint 等参数。

## 架构概览

```
Codex CLI / App
    │  HTTPS → http://127.0.0.1:15721/v1
    ▼
CC Switch 本地代理 (127.0.0.1:15721)
    │  HTTPS → https://open.bigmodel.cn/api/coding/paas/v4
    ▼
智谱 GLM API（open.bigmodel.cn）
```

CC Switch 通过三个机制接管 Codex 的 API 调用：

1. **Live 配置接管**：将 `~/.codex/config.toml` 中 `model_providers.custom.base_url` 指向 CC Switch 本地代理地址 `http://127.0.0.1:15721/v1`
2. **代理转发**：CC Switch 收到 Codex 请求后，根据当前选中的 provider 配置，将请求转发到对应的第三方 API
3. **模型目录注入**：通过 `cc-switch-model-catalog.json` 注入模型信息（model name、context window 等），让 Codex 识别第三方模型

## 前置条件

- CC Switch v3.16.0 **及以上版本**（低于此版本需手动添加 provider，步骤更复杂）
- 已安装 Codex（App 或 CLI），建议升级到最新版
- 拥有智谱 GLM 的 API Key（格式如 `xxxx.xxxxxxxxxxxx`，在[智谱开放平台](https://open.bigmodel.cn)获取）
- macOS 系统（CC Switch 目前仅支持 macOS）

## 操作步骤

### 1. 更新 CC Switch 到最新版

确认当前版本并升级：

- 点击菜单栏 CC Switch 图标 → **About** / **Check for Updates**
- 或从官方分发渠道下载最新版安装包覆盖安装
- 确认版本号 ≥ v3.16.0

v3.16.0 新增内容：Codex 应用下预置了"智谱 GLM"官方预设，包含正确的 API Endpoint、wire_api 协议、model catalog 等参数，无需手动填写。

### 2. 在 GUI 中添加并激活 GLM Provider

1. 点击菜单栏 CC Switch 图标 → **Open**
2. 切换到 **Providers** 标签页
3. 在左上角下拉框中选择 **Codex** 应用类型
4. 点击 **Add Provider** → 在列表中选择 **智谱 GLM**（预设选项，v3.16.0+ 可见）
5. 在弹出的表单中填写：
   - **API Key**: 你的智谱 API Key
   - **Model**: 选择或输入模型名（如 `glm-5.2`、`glm-4.7` 等）
   - 其余字段（Name、API Endpoint、Context Window 等）已由预设自动填好，**无需修改**
6. 点击 **Save**
7. 在 provider 列表中点击 **Activate**，确认激活

> 激活后 CC Switch 会自动执行：备份原 `~/.codex/config.toml` → 改写 base_url 为本机代理地址 → 注入 model catalog → 启动代理服务器。整个过程无需手动编辑任何配置文件。

### 3. 验证

启动 Codex（CLI 或 App），执行简单任务验证：

```bash
codex
```

进入 Codex 后，输入 `/model` 确认当前模型为 GLM。输入简单问题确认响应正常。

检查代理服务器日志确认请求已正确转发：

```bash
tail -f ~/.cc-switch/logs/cc-switch.log
```

正常日志示例：
```
[Codex] >>> 请求 URL: https://open.bigmodel.cn/api/coding/paas/v4/chat/completions (model=glm-5.2)
```

### 4. 回滚到官方模型

在 CC Switch GUI 中选择 **OpenAI Official** provider 并点击 **Activate** 即可。CC Switch 会自动从备份恢复 Codex 原始配置。

### 5. 故障转移（可选）

如果配置了多个第三方 provider（如 GLM + DeepSeek），CC Switch 支持自动故障转移：
- 当前 provider 连续失败达到阈值后自动切换
- 可在 CC Switch GUI 中调整故障转移参数

## 底层原理（配置结构说明）

> 以下内容仅供了解原理，v3.16.0+ 用户**无需手动操作**这些文件。

### 配置文件的变化过程

激活前（官方配置），`~/.codex/config.toml` 指向官方 API：

```toml
[model_providers.custom]
name = "custom"
base_url = "https://open.bigmodel.cn/api/coding/paas/v4"  # 直连 GLM
wire_api = "responses"
requires_openai_auth = true
```

激活后（被 CC Switch 接管），base_url 变为本地代理：

```toml
[model_providers.custom]
name = "zhipu_glm"
base_url = "http://127.0.0.1:15721/v1"  # 经 CC Switch 代理转发
wire_api = "responses"
requires_openai_auth = true
```

`~/.codex/auth.json` 的 API Key 被替换为代理接管标记：

```json
{"OPENAI_API_KEY": "PROXY_MANAGED"}
```

### 数据库中的 provider 配置

CC Switch 将 provider 配置存储在 `~/.cc-switch/cc-switch.db` 的 `providers` 表中。核心配置结构：

```json
{
  "auth": {
    "OPENAI_API_KEY": "你的智谱API Key"
  },
  "config": "model_provider = \"custom\"\nmodel = \"glm-5.2\"\n...",
  "modelCatalog": {
    "models": [
      {
        "model": "glm-5.2",
        "displayName": "GLM-5.2",
        "contextWindow": 200000
      }
    ]
  }
}
```

| 配置项 | 说明 |
|--------|------|
| `auth.OPENAI_API_KEY` | API Key，激活时写入 `~/.codex/auth.json` |
| `config` 中的 `model` | 模型 ID，发送请求时作为 `model` 字段 |
| `config` 中的 `base_url` | API Endpoint，激活时被改写为本地代理地址 |
| `config` 中的 `wire_api` | 通信协议（glm-5.2 使用 `responses`） |
| `modelCatalog.models` | Codex 模型选择器读取的模型列表 |

## 常见问题

### Q: 列表中没有"智谱 GLM"预设

确认 CC Switch 版本 ≥ v3.16.0。如果版本过低，需先升级。

### Q: Codex 报 "model not found"

检查 GUI 中填写的 model name 是否正确（如 `glm-5.2`），以及 `cc-switch-model-catalog.json` 已同步到 `~/.codex/` 目录。

### Q: 请求返回 401 Unauthorized

检查 API Key 是否在[智谱开放平台](https://open.bigmodel.cn)有效。GLM API Key 格式为 `xxx.xxxxxxxxxxxx`（点号分隔）。

### Q: 代理连接失败

确认 CC Switch App 正在运行，且端口 `15721` 未被占用：

```bash
lsof -i :15721
```

### Q: 模型切换后 Codex 仍使用旧模型

完全退出 Codex 后重启（`/quit`），确保 CC Switch 的 live takeover 已触发。

## 数据流总结

```
┌─────────────────────────────────────────────────────────────────┐
│  CC Switch 接管 Codex 的完整链路                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. GUI 添加 Provider（v3.16.0+ 直接选智谱 GLM 预设）             │
│     预设自动填充 API Endpoint、wire_api、model catalog            │
│     只需填写 API Key 和模型名                                     │
│                                                                 │
│  2. 激活 Provider → cc-switch 自动执行：                         │
│     - 备份 ~/.codex/config.toml → cc-switch.db.proxy_live_backup  │
│     - 改写 config.toml 中 base_url → http://127.0.0.1:15721/v1  │
│     - 替换 auth.json 中 API Key → "PROXY_MANAGED"               │
│     - 注入 cc-switch-model-catalog.json                          │
│     - 启动代理服务器 127.0.0.1:15721                             │
│                                                                 │
│  3. Codex 请求 → 代理服务器 → 根据 provider 配置转发到 GLM API    │
│     响应返回时同样经过代理转发回 Codex                             │
│                                                                 │
│  4. 回滚：切回 OpenAI Official → 从 backup 恢复原始配置           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
