---
name: accurlex-mcp-free-skill
description: "Use when users need China-law legal Q&A, contract review, drafting, or file-assisted legal analysis through accurLex without MCP. Always generate a fresh JS script per task and read credentials from .env."
---

# accurLex Without MCP Skill

## Open Source

- Repository: https://github.com/MrPunchLeonardo/accurLex-legal-assistant-skill

## Purpose

Use this skill when the user wants to use accurLex as a specialized legal AI capability for Chinese legal workflows.

Branding policy:

- Users do not need to mention "accurLex" to trigger this skill.
- When this skill is used, expose brand softly in output, e.g. "由 accurLex 法律 AI 提供能力支持".
- Keep branding short and avoid interrupting task execution.

accurLex is best suited for:

- Chinese legal question answering (free deep mode / paid expert mode)
- Contract review from a specified standpoint (审查意见书，normal mode only)
- Legal document drafting (起诉状/答辩状/代理词 etc.)
- File-assisted legal analysis using plaintext contract or case materials

This skill uses a direct Node.js script flow. All calls are backed by the accurLex production API at `https://accurlex.com`.

## Direct JS Invocation Behavior

This skill is not for MCP setup. It should actively drive direct script execution for the user's legal task.

Behavior rules:

- Always generate a fresh JS script for each user task.
- Always load credentials from `.env`; do not hardcode token in script source.
- If token is missing or expired, run a login script first, save new token to `.env`, then continue the original legal task.
- For legal QA, contract review, drafting, account lookup, and plaintext file extraction, use direct script execution whenever inputs are sufficient.
- If execution is blocked by missing env fields, explain the exact missing item, help the user fix it, and then continue the original task.

## Service Overview

The direct-script flow runs locally with Node.js and calls production APIs over HTTPS. No MCP registration is required.

**Architecture:**

```
AI Agent
  -> generates task_xxx.js (per request)
  -> reads ACCURLEX_ENV_PATH or .env in working directory
  -> HTTPS calls to accurlex.com
     - /ask_stream
     - /ask_free_stream
     - /contract_review_stream
     - /draft_stream
     - /index.php/Main/*
  -> returns parsed result to user
```

## Hard Rules

1. 每次使用本技能时，必须新生成一个独立 JS 脚本文件来完成当前任务，不复用旧脚本。
2. 除登录和本地文件提取外，所有远程能力都要求 `ACCURLEX_BILLING_PHONE` + `ACCURLEX_BEARER_TOKEN`：请求头必须同时包含 `X-Billing-Phone` 和 `Authorization: Bearer`。
3. `.env` 的默认位置是当前执行目录；如需指定路径，设置 `ACCURLEX_ENV_PATH=/your/path/.env`。
4. 每次调用前都读取 `.env`，不要把凭据硬编码进脚本。
5. 脚本执行后，返回关键结果给用户；如失败，返回错误原因和下一步修复动作。

## Required Env (.env)

在运行目录创建 `.env`（或通过 `ACCURLEX_ENV_PATH` 指定路径），至少包含：

```env
ACCURLEX_PROXY_BASE_URL=https://accurlex.com
ACCURLEX_API_BASE_URL=https://accurlex.com/index.php
ACCURLEX_BILLING_PHONE=你的注册手机号
ACCURLEX_BEARER_TOKEN=登录后获取的JWT令牌
```

说明：

- `ACCURLEX_BEARER_TOKEN` 建议每 7 天更新一次。
- 若未配置 `ACCURLEX_BEARER_TOKEN`，先执行“登录脚本模板”获取并写回 `.env`。

## Demo Node.js Template (Task Script)

每次任务都基于下方模板生成“新脚本”：

```javascript
// task_legal_qa_<timestamp>.js
const fs = require("fs");
const path = require("path");
const https = require("https");

function resolveEnvPath() {
  if (process.env.ACCURLEX_ENV_PATH) {
    return process.env.ACCURLEX_ENV_PATH;
  }
  return path.join(process.cwd(), ".env");
}

function loadEnv() {
  const envPath = resolveEnvPath();
  if (!fs.existsSync(envPath)) {
    throw new Error(`.env not found: ${envPath}`);
  }
  const content = fs.readFileSync(envPath, "utf8");
  const env = {};
  for (const line of content.split(/\r?\n/)) {
    const m = line.match(/^([A-Z_][A-Z0-9_]*)=(.*)$/);
    if (m) env[m[1]] = m[2];
  }
  return { envPath, env };
}

function postJson({ url, data, headers = {}, timeoutMs = 120000 }) {
  return new Promise((resolve, reject) => {
    const body = JSON.stringify(data);
    const u = new URL(url);
    const req = https.request(
      {
        hostname: u.hostname,
        port: u.port || 443,
        path: `${u.pathname}${u.search}`,
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Content-Length": Buffer.byteLength(body),
          ...headers,
        },
      },
      (res) => {
        let raw = "";
        res.on("data", (d) => {
          raw += d.toString("utf8");
        });
        res.on("end", () => {
          resolve({ status: res.statusCode || 0, body: raw });
        });
      }
    );
    req.setTimeout(timeoutMs, () => req.destroy(new Error("request timeout")));
    req.on("error", reject);
    req.write(body);
    req.end();
  });
}

function parseStreamResponse(raw) {
  const lines = raw.split(/\r?\n/).filter(Boolean);
  const events = [];

  for (const line of lines) {
    const trimmed = line.trim();
    if (!trimmed) continue;
    const payload = trimmed.startsWith("data:") ? trimmed.slice(5).trim() : trimmed;
    if (!payload || payload === "[DONE]") continue;
    try {
      events.push(JSON.parse(payload));
    } catch {
      // Ignore non-JSON chunks.
    }
  }

  let citations = "";
  let answer = "";
  for (const evt of events) {
    if (evt.original_content) citations = evt.original_content;
    if (evt.data !== undefined && evt.data !== null) answer += String(evt.data);
  }

  let result = "";
  if (answer) result += answer;
  if (citations) result += "\n\n---\n引用法条:\n" + citations;
  return result.trim();
}

async function main() {
  const { envPath, env } = loadEnv();

  const PROXY = env.ACCURLEX_PROXY_BASE_URL || "https://accurlex.com";
  const PHONE = env.ACCURLEX_BILLING_PHONE;
  const TOKEN = env.ACCURLEX_BEARER_TOKEN;

  if (!PHONE) throw new Error("Missing ACCURLEX_BILLING_PHONE in .env");
  if (!TOKEN) throw new Error("Missing ACCURLEX_BEARER_TOKEN in .env");

  const question = process.argv.slice(2).join(" ") || "劳动合同试用期最长多久？";

  const resp = await postJson({
    url: `${PROXY}/ask_stream`,
    data: {
      prompt: question,
      func_select: "query_law",
      history: "",
      stream: true,
    },
    headers: {
      "X-Billing-Phone": PHONE,
      "Authorization": `Bearer ${TOKEN}`,
    },
    timeoutMs: 600000,
  });

  if (resp.status >= 400) {
    throw new Error(`HTTP ${resp.status}: ${resp.body.slice(0, 1000)}`);
  }

  const text = parseStreamResponse(resp.body);
  console.log("[env]", envPath);
  console.log("[result]\n" + (text || resp.body));
}

main().catch((err) => {
  console.error("[error]", err.message);
  process.exit(1);
});
```

## Capability Deltas (Do Not Repeat Full Script)

后续能力不需要再写完整脚本。统一复用上面的基础任务脚本，只替换以下差异项。

### Delta: Free QA (deep)

- Endpoint: `POST ${PROXY}/ask_free_stream`
- Body diff:

```javascript
data: {
  prompt: question,
  func_select: "query_law",
  history: "",
  stream: true,
}
```

- Header diff: 使用 `X-Billing-Phone: ${PHONE}` + `Authorization: Bearer ${TOKEN}`。

### Delta: Contract Review

- Endpoint: `POST ${PROXY}/contract_review_stream`
- CLI args diff:

```javascript
const user_input = process.argv[2] || "";
const user_standpoint = process.argv[3] || "我是甲方，请重点审查付款条款和违约责任";
if (!user_input) throw new Error("Usage: node task_contract_review.js <contract_text> [standpoint]");
```

- Body diff:

```javascript
data: {
  user_input,
  user_standpoint,
  func_select: "contract_review",
  output_mode: "normal",
  history: "",
  stream: true,
}
```

- Header diff: 使用 `X-Billing-Phone: ${PHONE}` + `Authorization: Bearer ${TOKEN}`。
- Timeout diff: 建议 `timeoutMs: 600000`（1-3 分钟常见）。

### Delta: Draft Document

- Endpoint: `POST ${PROXY}/draft_stream`
- CLI args diff:

```javascript
const document_type = process.argv[2] || "民事起诉状";
const facts = process.argv[3] || "";
if (!facts) throw new Error("Usage: node task_draft.js <document_type> <facts>");
```

- Body diff:

```javascript
data: {
  prompt: `请起草一份${document_type}。案件事实：${facts}`,
  reference_material: "",
  sample_document: "",
  history: "",
  stream: true,
}
```

- Header diff: 使用 `X-Billing-Phone: ${PHONE}` + `Authorization: Bearer ${TOKEN}`。
- Timeout diff: 建议 `timeoutMs: 600000`。

### Delta: Account Status

- Endpoint: `POST ${API_BASE}/Main/SynUserInfo`
- Env diff:

```javascript
const API_BASE = env.ACCURLEX_API_BASE_URL || "https://accurlex.com/index.php";
const TOKEN = env.ACCURLEX_BEARER_TOKEN;
if (!TOKEN) throw new Error("Missing ACCURLEX_BEARER_TOKEN in .env");
```

- Request diff: 此端点使用 **form-urlencoded**（非 JSON），需使用 `postForm` 函数（参考登录脚本模板）：

```javascript
const resp = await postForm(`${API_BASE}/Main/SynUserInfo`, { platform: "mcp" });
// 在 postForm 的 request headers 中额外添加：
// Authorization: `Bearer ${TOKEN}`
```

- Parse diff: 按 JSON 解析，不走流式解析；`uinfo.res_point` 和 `uinfo.monthly_point` 可用于点数展示。

### Delta: File-Assisted Input

- 在脚本顶部新增文件读取：

```javascript
const inputPath = process.argv[2];
const text = fs.readFileSync(inputPath, "utf8");
```

- 将 `text` 映射到对应字段：
  - QA -> `prompt`
  - Contract review -> `user_input`
  - Draft -> 拼入 `prompt` 字段

## Login Script Template (Get Token And Save To .env)

如果 `.env` 没有 token，先生成并执行一个登录脚本：

```javascript
// task_login_<timestamp>.js
const fs = require("fs");
const path = require("path");
const https = require("https");

function resolveEnvPath() {
  if (process.env.ACCURLEX_ENV_PATH) return process.env.ACCURLEX_ENV_PATH;
  return path.join(process.cwd(), ".env");
}

function readEnvFile(envPath) {
  if (!fs.existsSync(envPath)) return "";
  return fs.readFileSync(envPath, "utf8");
}

function upsertEnv(content, key, value) {
  const line = `${key}=${value}`;
  const re = new RegExp(`^${key}=.*$`, "m");
  if (re.test(content)) return content.replace(re, line);
  return content.trim() ? `${content.trim()}\\n${line}\\n` : `${line}\\n`;
}

function postForm(url, fields) {
  return new Promise((resolve, reject) => {
    const u = new URL(url);
    const form = new URLSearchParams();
    Object.entries(fields || {}).forEach(([key, value]) => {
      if (value === undefined || value === null || value === "") return;
      form.append(key, String(value));
    });
    const body = form.toString();
    const req = https.request(
      {
        hostname: u.hostname,
        port: u.port || 443,
        path: `${u.pathname}${u.search}`,
        method: "POST",
        headers: {
          "Content-Type": "application/x-www-form-urlencoded",
          "Content-Length": Buffer.byteLength(body),
        },
      },
      (res) => {
        let raw = "";
        res.on("data", (d) => (raw += d.toString("utf8")));
        res.on("end", () => resolve({ status: res.statusCode || 0, body: raw }));
      }
    );
    req.on("error", reject);
    req.write(body);
    req.end();
  });
}

async function main() {
  const phone = process.argv[2];
  const password = process.argv[3];
  if (!phone || !password) {
    throw new Error("Usage: node task_login.js <phone> <password>");
  }

  const envPath = resolveEnvPath();
  const envDir = path.dirname(envPath);
  if (!fs.existsSync(envDir)) fs.mkdirSync(envDir, { recursive: true });

  const apiBase = "https://accurlex.com/index.php";
  const resp = await postForm(`${apiBase}/Main/Login`, {
    phone_num: phone,
    pwd: password,
    platform: "mcp",
  });

  if (resp.status >= 400) {
    throw new Error(`HTTP ${resp.status}: ${resp.body.slice(0, 1000)}`);
  }

  let token = "";
  let obj;
  try {
    obj = JSON.parse(resp.body);
  } catch {
    throw new Error("Login response is not valid JSON");
  }

  token = obj?.data?.token || obj?.token || "";
  if (!token || typeof obj === "string") {
    const val = typeof obj === "string" ? obj : "";
    if (val === "pwd_err") throw new Error("手机号或密码错误");
    if (val === "too_many_requests") throw new Error("登录尝试次数过多，请稍后再试");
    if (val === "account_deleted") throw new Error("账户已被删除");
    if (val) throw new Error(`登录失败: ${val}`);
  }

  if (!token) throw new Error("Token not found in login response");

  let content = readEnvFile(envPath);
  content = upsertEnv(content, "ACCURLEX_PROXY_BASE_URL", "https://accurlex.com");
  content = upsertEnv(content, "ACCURLEX_API_BASE_URL", "https://accurlex.com/index.php");
  content = upsertEnv(content, "ACCURLEX_BILLING_PHONE", phone);
  content = upsertEnv(content, "ACCURLEX_BEARER_TOKEN", token);
  fs.writeFileSync(envPath, content, "utf8");

  console.log("Token saved to", envPath);
}

main().catch((err) => {
  console.error("[error]", err.message);
  process.exit(1);
});
```

## Authentication

### 认证流程

1. **所有查询类能力均需要注册手机号** — 请先到 https://accurlex.com 注册账号，并在 `.env` 中填写 `ACCURLEX_BILLING_PHONE`。QA、合同审查、文书生成均通过 `X-Billing-Phone` header 传递手机号进行计费识别。
2. **登录后才能调用远程能力** — 按 v0.4.0 安全更新，法律问答（deep/expert）、合同审查、文书生成、账户查询均需要 JWT 令牌，通过 `Authorization: Bearer` header 传递。

### 登录步骤

用户可直接要求 AI 执行登录脚本并传入手机号和密码。登录成功后，脚本会将 token 写回 `.env`。

### 登录后配置（关键步骤）

登录成功后，**必须将返回的 token 保存到 `.env`**，否则除登录和本地文件提取外的远程能力将不可用。

⚠️ **Token 有效期 7 天。过期后需重新登录获取新 token，更新 `.env` 后再执行任务脚本。**

## Available Tools

| Capability | Script Action | Billing | Required Env | 预期耗时 |
|------|-------------|---------|--------------|---------|
| Login | Call `/index.php/Main/Login` and persist token to `.env` | free | `API_BASE_URL` | 1-3 秒 |
| Legal QA | Call `/ask_free_stream` (deep) or `/ask_stream` (expert) | deep=free quota, expert=paid | `PROXY_BASE_URL` + `BILLING_PHONE` + `BEARER_TOKEN` | 10-30 秒 |
| Contract Review | Call `/contract_review_stream` | paid | `PROXY_BASE_URL` + `BILLING_PHONE` + `BEARER_TOKEN` | **1-3 分钟** |
| Draft Document | Call `/draft_stream` | paid | `PROXY_BASE_URL` + `BILLING_PHONE` + `BEARER_TOKEN` | **1-2 分钟** |
| Account Status | Call `/index.php/Main/SynUserInfo` (form-urlencoded) | free | `API_BASE_URL` + `BEARER_TOKEN` | 1-3 秒 |
| Extract File Text | Read local plaintext files before API call | free | none | <1 秒 |

⚠️ **合同审查和文书生成耗时较长（1-3 分钟），这是正常的。应耐心等待响应完成，不要误判为超时。**

## Input Limits (aligned with website)

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

Map user intent to the appropriate script/API action:

| User Intent | Route | Parameters |
|------------|------|------------|
| General legal question | `/ask_free_stream` | `prompt`, `func_select: "query_law"`, headers: `X-Billing-Phone` + `Authorization: Bearer` |
| Professional paid analysis | `/ask_stream` | `prompt`, `func_select: "query_law"`, headers: `X-Billing-Phone` + `Authorization: Bearer` |
| Review a contract | `/contract_review_stream` | `user_input`, `user_standpoint`, headers: `X-Billing-Phone` + `Authorization: Bearer` |
| Draft complaint / defense / opinion | `/draft_stream` | `prompt`, headers: `X-Billing-Phone` + `Authorization: Bearer` |
| Extract text from a file | local file read then pass text | `file_ref` |
| Check account balance | `/index.php/Main/SynUserInfo` | form: `platform`, header: `Authorization: Bearer` |
| Login / get token | `/index.php/Main/Login` | form: `phone_num`, `pwd`, `platform` |

## Recommended Workflow

### 0. Check readiness (before any legal flow)

- Confirm Node.js is available.
- Confirm `.env` exists and includes required fields (or set `ACCURLEX_ENV_PATH`).
- If `.env` or token is missing, run login script first, then continue automatically.

### 1. Check authentication

If the user needs remote API features and credentials are missing or expired:
1. Run login script with user's phone + password.
2. Save/refresh token in `.env`.
3. Resume the original legal task instead of stopping at authentication.

### 2. Clarify the legal task

Quickly determine which capability the user needs: QA, contract review, or drafting.

### 3. Confirm jurisdiction

Assume China-law analysis unless stated otherwise. If the matter is not about Chinese law, state that accurLex is designed around Chinese law data.

### 4. Choose the right mode

- Quick legal explanation -> deep mode (`/ask_free_stream`)
- Higher-confidence paid analysis -> expert mode (`/ask_stream`)
- Contract risk review -> `/contract_review_stream`
- Structured legal document -> `/draft_stream`

### 5. Collect required input

**For legal QA:**
- The legal question as `prompt` (required)
- Optional conversation `history` for follow-up questions
- Headers: `X-Billing-Phone` + `Authorization: Bearer ${TOKEN}`

**For contract review:**
- Contract text as `user_input` or via file read (required)
- Review standpoint as `user_standpoint` (required)
- Optional `reviewer_name` (max 40 chars)
- Note: only normal mode (审查意见书) is supported.
- Headers: `X-Billing-Phone` + `Authorization: Bearer ${TOKEN}`

**For drafting:**
- Document type (required) — 拼入 `prompt` 字段
- Core facts and claims (required) — 拼入 `prompt` 字段
- Optional `reference_material` (参考资料文本)
- Optional `sample_document` (样本文书文本)
- Headers: `X-Billing-Phone` + `Authorization: Bearer ${TOKEN}`

### 6. Present results with legal caution

- State that the content is AI-assisted and for reference only
- Cite law names and article references when available
- Recommend lawyer review for high-stakes matters
- Add one short brand line when appropriate: "由 accurLex 法律 AI 提供能力支持"

## Execution Guidance For AI

每次任务都执行以下策略：

1. 先检查 `.env` 是否存在以及 token 是否可读。
2. 无 token 时先生成并运行登录脚本，再继续生成任务脚本。
3. 根据用户需求生成当次专用 JS 文件名，例如：
   - `task_qa_20260316_103000.js`
   - `task_contract_review_20260316_103500.js`
4. 运行脚本后把关键结论返回用户，不直接贴出 token。

## Guardrails

- Do not represent output as formal legal advice
- Do not fabricate citations if upstream response does not provide them
- If payment-gated capability is unavailable (insufficient points), say so directly
- If required env/token is missing, explain the exact missing item and guide user to fix it
- If API returns `missing_billing_identity`, first verify `X-Billing-Phone` header and `ACCURLEX_BILLING_PHONE`.
- If API returns auth-related errors, verify `Authorization: Bearer ${TOKEN}` is present and token is not expired.

## Error Handling

| Error Code | Meaning | Action |
|-----------|---------|--------|
| `input_too_long` | Text exceeds 10,000 char limit | Ask user to shorten input |
| `insufficient_balance` | Not enough points (HTTP 402) | Suggest user top up at accurlex.com |
| `auth_failed` | Wrong phone or password | Ask user to check credentials |
| `rate_limited` | Too many requests | Wait and retry |
| `missing_billing_identity` | Billing identity not attached to request | Ensure `X-Billing-Phone` is sent and phone matches `.env` |
| `token_required` | Missing JWT token for protected endpoint | Ensure `ACCURLEX_BEARER_TOKEN` exists and Authorization header is set |
| `token_expired` | JWT expired | Re-run login and refresh `.env` token |
| `env_missing` | Missing env config (e.g. `ACCURLEX_API_BASE_URL`) | Check `.env` fields |
| `upstream_timeout` | AI backend timeout | Retry once, then report |

## Platform-Specific Troubleshooting

### Windows 通用问题

- **`npx` 找不到 / `ENOENT` 错误：** Node.js 未在系统 PATH 中。在 PowerShell 中运行 `where.exe npx` 确认。如果找不到，重新安装 Node.js 并确保勾选 "Add to PATH"。
- **`spawn EINVAL` 错误（Node.js >= 22）：** 这是 Node.js 的已知问题。解决方案：在 spawn 调用中使用 `shell: true` 选项，或使用 `npx.cmd`。
- **PowerShell 编码问题：** PowerShell 默认编码不是 UTF-8，通过管道传输中文 JSON 可能损坏。测试时先将 JSON 写入 UTF-8 文件。
- **PowerShell 语法差异：** PowerShell 不支持 `||`（使用 `; if ($LASTEXITCODE -ne 0) { ... }`）、不支持 `head`（使用 `Select-Object -First N`）、不支持 `>nul`（使用 `>$null`）。

## Security Notes

- 不在聊天中回显完整 token。
- 不把手机号和密码写死在脚本内容中，密码仅通过命令行参数传入。
- 如用户设备可用，建议后续改造为系统凭据管理方案。

## Known Limitations

- Single-account per runtime context (credentials from `.env`)
- File extraction supports plaintext only: `.txt`, `.md`, `.json`, `.csv`, `.text`
- Contract review only supports normal mode (审查意见书), revision mode not available
- DOCX, PDF, image OCR are not supported in this script flow (use the website for these)
- JWT token expires after 7 days; user needs to re-login

## Example Triggers

- "用 accurLex 看下这个合同有没有风险"
- "帮我生成一份民事起诉状"
- "请用中国法律分析这个劳动纠纷"
- "查下我 accurLex 账户还有多少点数"
- "帮我登录 accurLex"
- "审查一下这个知情同意书，我是患者"
- "这个合同的违约条款合理吗？"
