# kimi-code-mcp

MCP server that connects [Kimi Code](https://www.kimi.com/code) (K2.5, 256K context) with [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — letting Claude orchestrate while Kimi handles the heavy reading.

**[中文說明](#中文說明)**

---

## Why This Exists

Claude Code is powerful but expensive. Every file it reads, every codebase scan it performs, costs tokens. Meanwhile, many tasks — pre-reviewing large codebases, scanning for patterns across hundreds of files, generating audit reports — are **high-certainty work** that doesn't need Claude's full reasoning power.

**The idea is simple: Claude Code as the conductor, Kimi Code as the specialist reader.**

```
                          ┌─────────────────────────────┐
                          │   You (the developer)       │
                          └──────────┬──────────────────┘
                                     │ prompt
                                     ▼
                          ┌─────────────────────────────┐
                          │   Claude Code (conductor)   │
                          │   - orchestrates workflow    │
                          │   - makes decisions          │
                          │   - writes & edits code      │
                          └──────┬──────────────┬───────┘
                      precise    │              │  delegate
                      edits      │              │  bulk analysis
                                 ▼              ▼
                          ┌──────────┐   ┌──────────────┐
                          │ your     │   │  Kimi Code   │
                          │ codebase │   │  (K2.5)      │
                          └──────────┘   │  - 256K ctx  │
                                         │  - reads all │
                                         │  - reports   │
                                         └──────────────┘
```

### Save Claude Code Tokens

Instead of Claude reading 50+ files to understand your architecture, delegate that to Kimi:

1. **Claude** receives your task → decides it needs codebase understanding
2. **Claude** calls `kimi_analyze` via MCP → Kimi reads the entire codebase (256K context, near-zero cost)
3. **Kimi** returns a structured analysis
4. **Claude** acts on the analysis with precise, targeted edits

Result: Claude only spends tokens on **decision-making and code writing**, not on reading files.

### Mutual Code Review with K2.5

Kimi Code is powered by K2.5 — a model designed for deep code comprehension. This enables a powerful pattern:

1. **Kimi pre-reviews** — scan the full codebase for security issues, anti-patterns, dead code, or architectural problems (256K context means it sees everything)
2. **Claude cross-examines** — reviews Kimi's findings, challenges questionable items, adds its own insights
3. **Two perspectives** — different models catch different things. What one misses, the other finds.

This isn't just delegation — it's **AI pair review**. Two models with different strengths auditing the same code from different angles.

## Features

| Tool | Description | Timeout |
|------|-------------|---------|
| `kimi_analyze` | Deep codebase analysis (architecture, audit, refactoring) | 10 min |
| `kimi_query` | Quick programming questions, no codebase context | 2 min |
| `kimi_list_sessions` | List existing Kimi sessions with metadata | instant |
| `kimi_resume` | Resume a previous session (up to 256K token context) | 10 min |

## Prerequisites

1. **Kimi CLI** — install via [uv](https://docs.astral.sh/uv/):
   ```bash
   uv tool install kimi-cli
   ```
2. **Authenticate Kimi**:
   ```bash
   kimi login
   ```
3. **Node.js** >= 18

## Installation

```bash
git clone https://github.com/howardpen9/kimi-code-mcp.git
cd kimi-code-mcp
npm install
npm run build
```

## Usage with Claude Code

Add to your project's `.mcp.json` (or `~/.claude/mcp.json` for global):

```json
{
  "mcpServers": {
    "kimi-code": {
      "command": "node",
      "args": ["/absolute/path/to/kimi-code-mcp/dist/index.js"]
    }
  }
}
```

For development (auto-recompile):

```json
{
  "mcpServers": {
    "kimi-code": {
      "command": "npx",
      "args": ["tsx", "/absolute/path/to/kimi-code-mcp/src/index.ts"]
    }
  }
}
```

### Verify

In Claude Code, run `/mcp` to check the server is connected. You should see `kimi-code` with 4 tools.

## Example Workflows

**Delegate bulk analysis (save tokens):**
> "Use kimi_analyze to review this codebase's architecture, then tell me what needs refactoring"

Claude sends the task to Kimi, gets back a full report, then summarizes and acts on it — without reading hundreds of files itself.

**Mutual security audit:**
> "Have Kimi scan the codebase for security vulnerabilities, then review its findings and add anything it missed"

Two models, two perspectives. Kimi does the exhaustive scan (256K context), Claude cross-examines the results.

**Pre-review before refactoring:**
> "Ask Kimi to map all dependencies of the auth module, then plan the refactoring based on its analysis"

Kimi reads every file to build the dependency graph. Claude uses that map to plan precise changes.

**Resume context-heavy sessions:**
> "List Kimi sessions for this project, then resume the last one to ask about the auth flow"

Kimi retains up to 256K tokens of context from previous sessions — no need to re-read the codebase.

## How It Works

```
┌──────────────┐  stdio/MCP   ┌──────────────┐  subprocess   ┌──────────────┐
│  Claude Code │ ◄──────────► │ kimi-code-mcp│ ────────────► │ Kimi CLI     │
│  (conductor) │              │ (MCP server) │               │ (K2.5, 256K) │
└──────────────┘              └──────────────┘               └──────────────┘
```

1. Claude Code calls an MCP tool (e.g., `kimi_analyze`)
2. This server spawns the `kimi` CLI with the prompt and codebase path
3. Kimi autonomously reads files, analyzes the code (up to 256K tokens)
4. The result is parsed from Kimi's JSON output and returned to Claude Code
5. Claude acts on the structured results — edits, plans, or further analysis

## Project Structure

```
src/
├── index.ts           # MCP server setup, tool definitions
├── kimi-runner.ts     # Spawns kimi CLI, parses output, handles timeouts
└── session-reader.ts  # Reads Kimi session metadata from ~/.kimi/
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

MIT

---

# 中文說明

## 簡介

`kimi-code-mcp` 是一個 [MCP](https://modelcontextprotocol.io/) 伺服器，將 [Kimi Code](https://www.kimi.com/code)（K2.5，256K 上下文）與 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 串接。

## 為什麼需要這個？

Claude Code 很強大，但每次讀檔、掃描 codebase 都會消耗 token。很多工作——預審大型程式碼庫、跨檔案掃描模式、生成審計報告——屬於**確定性高的尾部任務**，不需要 Claude 的完整推理能力。

### 核心概念：Claude Code 當指揮家，Kimi Code 當專業讀者

**節省 Claude Code Token 成本：**
1. **Claude** 收到你的任務 → 判斷需要理解 codebase
2. **Claude** 透過 MCP 呼叫 `kimi_analyze` → Kimi 讀取整個程式碼庫（256K 上下文，近零成本）
3. **Kimi** 回傳結構化分析
4. **Claude** 根據分析做精準的程式碼修改

結果：Claude 只花 token 在**決策和寫碼**，不浪費在讀檔上。

### 基於 K2.5 的雙向程式碼審計

Kimi Code 搭載 K2.5 模型，專為深度程式碼理解而設計。這讓一個強大的工作流成為可能：

1. **Kimi 預審** — 用 256K 上下文掃描整個 codebase，找出安全問題、反模式、死代碼、架構問題
2. **Claude 交叉審查** — 審閱 Kimi 的發現，質疑可疑項目，補充自己的洞察
3. **雙重視角** — 不同模型捕捉不同問題。一個遺漏的，另一個能發現

這不只是委派工作——而是 **AI 結對審查**。兩個不同優勢的模型，從不同角度審計同一份程式碼。

## 功能

| 工具 | 說明 | 超時 |
|------|------|------|
| `kimi_analyze` | 深度程式碼分析（架構、審計、重構建議） | 10 分鐘 |
| `kimi_query` | 快速問答，不需要 codebase 上下文 | 2 分鐘 |
| `kimi_list_sessions` | 列出現有的 Kimi 分析 session | 即時 |
| `kimi_resume` | 恢復之前的 session（保留最多 256K token 上下文） | 10 分鐘 |

## 前置需求

1. **Kimi CLI**：`uv tool install kimi-cli`
2. **登入 Kimi**：`kimi login`
3. **Node.js** >= 18

## 安裝

```bash
git clone https://github.com/howardpen9/kimi-code-mcp.git
cd kimi-code-mcp
npm install
npm run build
```

## 配置 Claude Code

在專案的 `.mcp.json`（或 `~/.claude/mcp.json` 全域設定）加入：

```json
{
  "mcpServers": {
    "kimi-code": {
      "command": "node",
      "args": ["/你的路徑/kimi-code-mcp/dist/index.js"]
    }
  }
}
```

## 使用範例

**委派分析（省 token）：**
> 「用 kimi_analyze 分析這個 codebase 的架構，告訴我需要重構什麼」

**雙向安全審計：**
> 「讓 Kimi 掃描 codebase 的安全漏洞，然後審閱它的發現，補充它遺漏的」

**重構前預審：**
> 「請 Kimi 映射 auth 模組的所有依賴，然後根據分析規劃重構」

**恢復上次分析：**
> 「列出這個專案的 Kimi sessions，然後恢復上一次的分析繼續問」

## 貢獻

請參閱 [CONTRIBUTING.md](CONTRIBUTING.md)。
