---
name: gemini-researcher
description: 用 Gemini CLI 进行研究任务，支持深度研究、快速总结、对比分析等模式
---

# gemini-researcher

用 Gemini CLI 进行研究任务的小助手。

## 安装

```bash
npm install -g @google/gemini-cli
```

首次运行需要登录：

```bash
gemini
```

## 快速开始

所有 prompt 默认都会要求 Gemini 先做 Google 搜索，确保信息是最新的。

```bash
# 直接搜索（默认模式，会联网搜索）
gemini-researcher "今天发生了什么AI新闻"

# 使用预设命令
gemini-researcher deep "研究主题"
gemini-researcher summary "URL 或内容"
```

## 功能特点

- **自动联网搜索**：每个 prompt 都会要求 Gemini 先 Google 搜索，确保信息实时准确
- **核心检索模板**：deep, summary, compare, list, news, find 等
- **可自定义模型**：通过 `--model` 指定模型

## 命令列表

| 命令      | 说明                      | 示例                                             |
| --------- | ------------------------- | ------------------------------------------------ |
| `deep`    | 深度研究（详细 Markdown） | `gemini-researcher deep "AI Agent 2025 趋势"`    |
| `summary` | 快速总结                  | `gemini-researcher summary "https://..."`        |
| `compare` | 对比分析                  | `gemini-researcher compare "Notion vs Obsidian"` |
| `list`    | 列表整理                  | `gemini-researcher list "Python 技巧" 10`        |
| `news`    | 新闻汇总                  | `gemini-researcher news "AI 领域"`               |
| `find`    | 资源查找                  | `gemini-researcher find "免费 API"`              |
| `links`   | 纯链接检索                | `gemini-researcher links "最新开源大模型"`       |
| `extract` | 提取要点                  | `gemini-researcher extract "URL" key insights`   |

## 选项

| 选项             | 说明                                      |
| ---------------- | ----------------------------------------- |
| `--model <name>` | 指定模型（默认：gemini CLI 默认模型）     |
| `--search`       | 强制明确要求 Google 搜索                  |
| `--json`         | 强制输出严格的 JSON 格式，不包裹 Markdown |

## 示例

```bash
# 直接问（会自动搜索）
gemini-researcher "今天AI圈发生了什么"
gemini-researcher "比特币现在多少钱"

# 深度研究
gemini-researcher deep "WebAssembly 应用场景"

# 对比分析
gemini-researcher compare "PostgreSQL vs MySQL"

# 指定模型
gemini-researcher --model gemini-2.0-flash "简单问题"
```

## 内部命令

| 命令              | 说明                       |
| ----------------- | -------------------------- |
| `help`            | 显示帮助                   |
| `template <type>` | 显示指定类型的 prompt 模板 |
| `types`           | 列出所有可用的命令类型     |

## 注意事项

- 每个问题都会自动要求 Gemini 先做 Google 搜索，确保信息是最新的
- 免费账号有速率限制，付费版更稳定
- 模型名称使用 gemini CLI 的格式
