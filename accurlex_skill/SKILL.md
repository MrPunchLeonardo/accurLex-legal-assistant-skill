---
name: accurlex-legal-assistant
description: "Use when users need China-law legal Q&A, contract review, legal document drafting, or file-based legal analysis through accurLex. Trigger for keywords like accurLex, 法律问答, 合同审查, 文书生成, 法律分析, 起诉状, 答辩状, 合同风险, 审查意见, 中国法律, Neo-RAG."
---

# accurLex Legal Assistant

## Purpose

Use this skill when the user wants to use accurLex as a specialized legal AI capability for Chinese legal workflows.

accurLex is best suited for:

- Chinese legal question answering (free deep mode / paid expert mode)
- Contract review from a specified standpoint (审查意见书，normal mode only)
- Legal document drafting (起诉状/答辩状/代理词 etc.)
- File-assisted legal analysis using plaintext contract or case materials

This skill routes user intent to the accurLex MCP server tools. All tools are backed by the accurLex production API at `https://accurlex.com`.

- **npm 包名：** `accurlex-mcp-server`
- **安装命令：** `npx -y accurlex-mcp-server`（无需 clone 代码）
- **npm 页面：** https://www.npmjs.com/package/accurlex-mcp-server

## MCP Server Overview

The accurLex MCP server is published on npm as `accurlex-mcp-server`. It runs locally on the user's machine (Node.js ≥ 18) and calls the production API over HTTPS. No server-side modification required.

**Architecture:**

```
AI Client (OpenClaw / VS Code Copilot / Claude Desktop)
    │  stdio
    ▼
accurlex-mcp-server (local Node.js process, via npx)
    │  HTTPS
    ├── https://accurlex.com/ask_stream          (expert QA)
    ├── https://accurlex.com/ask_free_stream     (free QA)
    ├── https://accurlex.com/contract_review_stream  (contract review)
    ├── https://accurlex.com/draft_stream        (drafting)
    ├── https://accurlex.com/index.php/Main/*    (login, account)
    ▼
accurLex Production Server (unchanged, zero server-side modification)
```

## Installation & Configuration

### Prerequisites

- Node.js ≥ 18（已安装即可，无需 clone 代码）

### OpenClaw / 龙虾

```json
{
  "mcpServers": {
    "accurlex": {
      "command": "npx",
      "args": ["-y", "accurlex-mcp-server"],
      "env": {
        "ACCURLEX_PROXY_BASE_URL": "https://accurlex.com",
        "ACCURLEX_API_BASE_URL": "https://accurlex.com/index.php"
      }
    }
  }
}
```

### VS Code GitHub Copilot

在工作区 `.vscode/settings.json` 中添加：

```json
{
  "mcp": {
    "servers": {
      "accurlex": {
        "type": "stdio",
        "command": "npx",
        "args": ["-y", "accurlex-mcp-server"],
        "env": {
          "ACCURLEX_PROXY_BASE_URL": "https://accurlex.com",
          "ACCURLEX_API_BASE_URL": "https://accurlex.com/index.php"
        }
      }
    }
  }
}
```

### Claude Desktop

编辑 `%APPDATA%\Claude\claude_desktop_config.json`（Windows）或 `~/Library/Application Support/Claude/claude_desktop_config.json`（macOS）：

```json
{
  "mcpServers": {
    "accurlex": {
      "command": "npx",
      "args": ["-y", "accurlex-mcp-server"],
      "env": {
        "ACCURLEX_PROXY_BASE_URL": "https://accurlex.com",
        "ACCURLEX_API_BASE_URL": "https://accurlex.com/index.php"
      }
    }
  }
}
```

### Cursor / Windsurf

在 Settings → MCP 中添加，配置格式同 OpenClaw。

### Authentication

连接 MCP 服务后，用户只需对 AI 说：

> "帮我登录 accurLex，手机号 138xxxx1234，密码 xxx"

`accurlex_login` 工具会返回 JWT token（7天有效期），按提示保存到 `.env` 后重启客户端即可使用付费功能。

免费功能（法律问答 deep 模式）无需登录。

## Available Tools

| Tool | Description | Billing | Required Config |
|------|-------------|---------|-----------------|
| `accurlex_login` | Login with phone + password, returns JWT token | free | `ACCURLEX_API_BASE_URL` |
| `accurlex_legal_qa` | Legal Q&A, deep (free) or expert (paid) mode | deep=free, expert=paid | `ACCURLEX_PROXY_BASE_URL` |
| `accurlex_contract_review` | Review contract → 审查意见书 (normal mode only) | paid | `ACCURLEX_PROXY_BASE_URL` + `BILLING_PHONE` |
| `accurlex_draft_document` | Draft legal documents | paid | `ACCURLEX_PROXY_BASE_URL` + `BILLING_PHONE` |
| `accurlex_get_account_status` | Check account point balances | free | `ACCURLEX_API_BASE_URL` + `BEARER_TOKEN` |
| `accurlex_extract_text_from_file` | Extract text from local plaintext file | free | none |

## Input Limits (aligned with website)

All input limits match the accurLex website frontend:

| Field | Max Length |
|-------|-----------|
| Legal QA question | 10,000 chars |
| Contract text + standpoint (combined) | 10,000 chars |
| Draft document facts | 10,000 chars |
| Reviewer name | 40 chars |
| Draft followup prompt | 2,000 chars |

Input is sanitized: null bytes and control characters are stripped (matching the website's `normalizeTextInput`).

## When To Use

Use this skill when the user asks for any of the following:

- Legal Q&A based on Chinese law
- Contract risk review or clause analysis
- Drafting legal documents such as complaints, replies, opinions, or agency statements
- Uploading files for legal interpretation
- Using accurLex specifically

Do not use this skill for:

- Non-legal casual conversation
- Legal systems outside China unless the user explicitly accepts Chinese-law-only analysis
- Final professional legal advice that requires a licensed lawyer opinion

## Capability Routing

Map user intent to the appropriate accurLex tool:

| User Intent | Tool | Parameters |
|------------|------|------------|
| General legal question | `accurlex_legal_qa` | `question`, `mode: "deep"` |
| Professional paid analysis | `accurlex_legal_qa` | `question`, `mode: "expert"` |
| Review a contract | `accurlex_contract_review` | `contract_text`, `standpoint` |
| Draft complaint / defense / opinion | `accurlex_draft_document` | `document_type`, `facts` |
| Extract text from a file | `accurlex_extract_text_from_file` | `file_ref` |
| Check account balance | `accurlex_get_account_status` | (none) |
| Login / get token | `accurlex_login` | `phone_num`, `password` |

## Recommended Workflow

### 1. Check authentication

If the user hasn't configured credentials yet, guide them to use `accurlex_login` first. The login tool returns a JWT token (valid for 7 days) and instructions to save it in `.env`.

### 2. Clarify the legal task

Quickly determine which capability the user needs: QA, contract review, or drafting.

### 3. Confirm jurisdiction

Assume China-law analysis unless stated otherwise. If the matter is not about Chinese law, state that accurLex is designed around Chinese law data.

### 4. Choose the right mode

- Quick legal explanation → `accurlex_legal_qa` with `mode: "deep"` (free)
- Higher-confidence paid analysis → `accurlex_legal_qa` with `mode: "expert"` (costs points)
- Contract risk review → `accurlex_contract_review` (costs points, normal/opinion mode only)
- Structured legal document → `accurlex_draft_document` (costs points)

### 5. Collect required input

**For legal QA:**
- The legal question (required)
- Optional background facts via `context_text`
- Optional conversation `history` for follow-up questions

**For contract review:**
- Contract text via `contract_text` or `file_ref` (required)
- Review standpoint — free text, e.g. "我是甲方，请重点审查付款条款和违约责任" (required)
- Optional `reviewer_name` for the opinion header (max 40 chars)
- Note: only normal mode (审查意见书) is supported. Revision mode (修订版) is not available via MCP.

**For drafting:**
- Document type, e.g. "民事起诉状", "答辩状", "代理词" (required)
- Core facts and claims (required)
- Optional `case_file_refs` (array of plaintext file paths)
- Optional `style_file_ref` (style sample file)
- Optional `followup_prompt` for additional instructions

### 6. Present results with legal caution

- State that the content is AI-assisted and for reference only
- Cite law names and article references when available
- Recommend lawyer review for high-stakes matters

## Guardrails

- Do not represent output as formal legal advice
- Do not fabricate citations if the tool does not provide them
- If payment-gated capability is unavailable (insufficient points), say so directly
- If no MCP server is connected, tell the user and offer to help configure it

## Error Handling

| Error Code | Meaning | Action |
|-----------|---------|--------|
| `input_too_long` | Text exceeds 10,000 char limit | Ask user to shorten input |
| `insufficient_balance` | Not enough points | Suggest user top up at accurlex.com |
| `auth_failed` | Wrong phone or password | Ask user to check credentials |
| `rate_limited` | Too many requests | Wait and retry |
| `tool_unavailable` | Missing env config | Guide user through `.env` setup |
| `upstream_timeout` | AI backend timeout | Retry once, then report |

## Known Limitations

- Single-account per server instance (credentials from `.env`)
- File extraction supports plaintext only: `.txt`, `.md`, `.json`, `.csv`, `.text`
- Contract review only supports normal mode (审查意见书)
- DOCX, PDF, image OCR are not supported in MCP (use the website for these)
- JWT token expires after 7 days; user needs to re-login

## Example Triggers

- "用 accurLex 看下这个合同有没有风险"
- "帮我生成一份民事起诉状"
- "请用中国法律分析这个劳动纠纷"
- "查下我 accurLex 账户还有多少点数"
- "帮我登录 accurLex"
