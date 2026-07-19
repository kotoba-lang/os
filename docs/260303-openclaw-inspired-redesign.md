# etzhayyim OS — OpenClaw-Inspired Redesign on kotodama runtime

**Date**: 2026-03-03
**Status**: Draft Proposal
**Reference**: https://openclaws.io/

## 1. Executive Summary

OpenClaw の「AI がチャットで返事するだけでなく、実際にタスクを実行する」体験を
etzhayyim OS で再現する。ただしローカル Node.js ではなく **K8s + kotodama runtime** 上に構築し、
WASM サンドボックスの安全性とクラウドスケールを両立する。

### OpenClaw → etzhayyim OS 対応表

| OpenClaw 機能 | etzhayyim OS 実装 | kotodama runtime component |
|---|---|---|
| ローカル AI アシスタント | クラウド常駐 + Tauri ローカル UI | `os-agent-{nanoid}` |
| チャット統合 (8 platform) | Messaging component 拡張 | `os-messaging-component` |
| Skills (100+ plugins) | Performer Skills (WASM sandbox) | `os-skills-{nanoid}` |
| 3 層メモリ | KV-backed Memory Engine | `os-memory-{nanoid}` |
| ファイルシステム操作 | Drive Sync + KV FS | `drive-sync-component` (既存) |
| シェル実行 | Sandboxed Runner (etzhayyim-browserless 連携) | `os-runner-{nanoid}` |
| ブラウザ自動操作 | Browserless Playwright 連携 | `etzhayyim-browserless` (既存) |
| タスクスケジューリング | Scheduler + Reminder (performer) | `os-scheduler-component` (既存) |
| Human-in-the-loop 承認 | Consent UI (既存 WIT) | `os-runtime-component` (既存) |
| モデル切替 (Claude/GPT/Ollama) | Multi-provider LLM Router | `os-llm-{nanoid}` |
| プライバシー (ローカル実行) | Per-user KV isolation + WASM sandbox | performer `ScopeUser` |

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        User Touchpoints                              │
│                                                                      │
│   WhatsApp  Telegram  Discord  Slack  iMessage  Signal  LINE  Web   │
│      │         │         │       │       │        │      │     │    │
│      └─────────┴─────────┴───────┴───────┴────────┴──────┴─────┘    │
│                              │                                       │
│                              ▼                                       │
│              ┌──────────────────────────────┐                        │
│              │   os-messaging (App)      │                        │
│              │   Messaging Platform Adapter  │                        │
│              │   Webhook receiver + sender   │                        │
│              └──────────────┬───────────────┘                        │
│                              │ XRPC                          │
│                              ▼                                       │
│              ┌──────────────────────────────┐                        │
│              │   os-agent (App)          │    ◀── Core Brain      │
│              │   Message → Intent → Action  │                        │
│              │   Conversation management    │                        │
│              │   Tool dispatch              │                        │
│              └──┬───────┬──────┬──────┬────┘                        │
│                 │       │      │      │                               │
│      ┌──────────┘  ┌────┘  ┌───┘  ┌───┘                             │
│      ▼             ▼       ▼      ▼                                  │
│  ┌────────┐ ┌────────┐ ┌───────┐ ┌──────────┐                      │
│  │os-llm  │ │os-memory│ │os-    │ │os-skills │                      │
│  │Router  │ │Engine   │ │runner │ │Registry  │                      │
│  │        │ │         │ │       │ │          │                      │
│  │Claude  │ │short    │ │shell  │ │100+ WASM │                      │
│  │GPT     │ │long     │ │browse │ │sandboxed │                      │
│  │Ollama  │ │semantic │ │file   │ │skills    │                      │
│  └────────┘ └────────┘ └───────┘ └──────────┘                      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                   Shared Infrastructure                       │    │
│  │  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌────────────┐  │    │
│  │  │NATS KV   │  │Consent   │  │Budget     │  │Audit       │  │    │
│  │  │(state)   │  │(approval)│  │(GCC token)│  │(log trail) │  │    │
│  │  └──────────┘  └──────────┘  └───────────┘  └────────────┘  │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  All Apps in `kotodama-runtime` namespace                                │
│  All routing via `{nanoid}.etzhayyim.com` per-subdomain direct routing     │
└─────────────────────────────────────────────────────────────────────┘
```

## 3. Component Design (kotodama runtime Apps)

### 3.1 os-messaging-component 拡張 — メッセージングゲートウェイ統合

**役割**: 8 つのチャットプラットフォームからの Webhook を受け取り、統一メッセージ形式に変換して os-agent に XRPC で転送する。返答も同じ経路で各プラットフォームに返す。新規ゲートウェイ App は作らず、既存の `os-messaging-component` に集約する。

```
60-apps/etzhayyim-project-os/wasm/os-messaging-component/
├── main.go                # Connect adapter entry
├── platform_discord.go    # Discord Interactions API
├── platform_telegram.go   # Telegram Bot API
├── platform_whatsapp.go   # WhatsApp Business Cloud API
├── platform_slack.go      # Slack Events API
├── platform_signal.go     # Signal Bot (via signal-cli REST)
├── platform_line.go       # LINE Messaging API
├── platform_web.go        # WebSocket / os-ui direct chat
├── platform_imessage.go   # iMessage (macOS native bridge proxy)
├── message.go             # 統一メッセージ型 (UnifiedMessage)
├── kotodama.toml
├── deploy config
└── go.mod
```

**統一メッセージ型**:

```go
type UnifiedMessage struct {
    ID          string   // platform-specific message ID
    Platform    string   // "discord" | "telegram" | "whatsapp" | ...
    ChannelID   string   // chat/group/channel ID
    UserID      string   // platform user ID
    UserName    string   // display name
    Text        string   // message body
    MentionedMe bool     // @mention or DM
    ReplyToID   string   // reply context
    Attachments []Attach // files, images
    Timestamp   int64
}
```

**Webhook → Agent flow**:

```
Platform webhook POST → os-messaging-component (App)
  → adapter_*.go: parse platform-specific payload
  → UnifiedMessage に変換
  → XRPC → os-agent/agent.chat
  → os-agent returns reply text + actions
  → adapter_*.go: platform API で返信送信
```

**KV 構成** (`wasi:keyvalue/store@0.2.0-draft2`):

| Key pattern | Purpose |
|---|---|
| `msg.{platform}.{channel_id}.config` | Channel config (enabled, mention-only, etc.) |
| `msg.{platform}.{user_id}.mapping` | Platform user → etzhayyim user mapping |
| `msg.webhook.secret.{platform}` | Webhook verification secrets |

### 3.2 os-agent — AI エージェントコア

**役割**: OpenClaw の心臓部に相当。メッセージを受け取り、LLM で意図を解析し、適切な Skill を呼び出し、結果を返す。会話管理・コンテキスト構築も担当。

```
60-apps/etzhayyim-project-os/wasm/etzhayyim-wasm-os-agent-<nanoid>/
├── main.go              # performer framework entry
├── agent.go             # Agent loop (message → think → act → respond)
├── conversation.go      # Conversation state machine
├── tool_dispatch.go     # Skill/Tool routing
├── context_builder.go   # Memory + conversation → LLM prompt 構築
├── kotodama.toml
├── deploy config
└── go.mod
```

**Agent Loop** (OpenClaw の ReAct パターン再現):

```
agent.chat(msg UnifiedMessage) → Reply
  │
  ├─ 1. Context Build
  │     ├─ os-memory/GetSemanticProfile(user_id)   → 常時プロフィール
  │     ├─ os-memory/SearchLongTerm(query, limit)   → 関連過去記憶
  │     └─ conversation history (KV, last N turns)  → 直近会話
  │
  ├─ 2. LLM Call (via os-llm)
  │     ├─ system prompt + semantic profile
  │     ├─ relevant memories
  │     ├─ conversation history
  │     ├─ user message
  │     └─ available tools (from os-skills registry)
  │     → LLM returns: text response + tool_calls[]
  │
  ├─ 3. Tool Execution Loop (max 10 iterations)
  │     ├─ for each tool_call:
  │     │     ├─ os/consent.evaluate(tool, args, risk_level)
  │     │     │     ├─ safe → execute
  │     │     │     ├─ dangerous → queue approval, wait/timeout
  │     │     │     └─ forbidden → deny
  │     │     ├─ os/budget.check(user_id, estimated_cost)
  │     │     ├─ os-skills/Execute(skill_id, method, args)
  │     │     └─ os/audit.log(tool_call, result)
  │     ├─ Collect results → feed back to LLM
  │     └─ LLM returns next response or final answer
  │
  ├─ 4. Memory Update
  │     ├─ os-memory/AppendShortTerm(conversation_turn)
  │     ├─ os-memory/ExtractFacts(conversation)  → semantic memory update
  │     └─ os-memory/StoreLongTerm(conversation_summary)
  │
  └─ 5. Return Reply to os-messaging-component
```

**Performer Config**:

```go
rt.Register(&performer.PerformerConfig{
    ID:           "<nanoid>",
    Name:         "os-agent",
    DefaultScope: performer.ScopeUser,
    Methods: map[string]performer.PerformerMethod{
        "agent.chat":          handleChat,         // main entry
        "get_conversation":    getConversation,     // 会話履歴取得
        "list_conversations":  listConversations,   // 会話一覧
        "clear_conversation":  clearConversation,   // 会話リセット
        "set_system_prompt":   setSystemPrompt,     // カスタム指示
    },
})
```

### 3.3 os-llm — マルチプロバイダ LLM ルーター

**役割**: OpenClaw の model-agnostic 設計を再現。Claude, GPT, Gemini, Ollama (ローカル) を統一インターフェースで切り替え可能にする。

```
60-apps/etzhayyim-project-os/wasm/etzhayyim-wasm-os-llm-<nanoid>/
├── main.go
├── router.go            # Provider routing + fallback chain
├── provider_anthropic.go # Claude API
├── provider_openai.go    # OpenAI API
├── provider_google.go    # Gemini API
├── provider_ollama.go    # Ollama (self-hosted, K8s internal)
├── provider_openrouter.go # OpenRouter (300+ models)
├── tools_schema.go       # Tool definition → provider-specific format 変換
├── kotodama.toml
├── deploy config
└── go.mod
```

**統一インターフェース**:

```go
type LLMRequest struct {
    Model       string          // "claude-sonnet-4-6", "gpt-4o", "ollama/llama3"
    Messages    []ChatMessage   // conversation
    Tools       []ToolDef       // available tools
    MaxTokens   int
    Temperature float64
    Stream      bool            // SSE streaming
}

type LLMResponse struct {
    Text      string       // generated text
    ToolCalls []ToolCall   // requested tool invocations
    Usage     TokenUsage   // input/output tokens (for budget tracking)
    Model     string       // actual model used
}
```

**Provider 選択ロジック**:

```
User preference (KV: pf.<id>.u.<uid>.s.llm_provider)
  → preferred provider available? → use it
  → fallback chain: Claude → OpenRouter → Ollama
  → budget check: remaining GCC tokens → auto-downgrade if low
```

**HTTPS 制約対応**: current Go/WASI toolchain では HTTP transport 実装を明示する。
→ `ydnar/wasi-http-go` (`wasihttp.Transport`) を使用 (crawler-mcp パターン踏襲)。

### 3.4 os-memory — 3 層メモリエンジン

**役割**: OpenClaw の 3-tier memory (short-term / long-term / semantic) を NATS KV 上に実装。

```
60-apps/etzhayyim-project-os/wasm/etzhayyim-wasm-os-memory-<nanoid>/
├── main.go
├── short_term.go    # 直近会話 (ring buffer in KV)
├── long_term.go     # 過去会話の要約・検索
├── semantic.go      # ユーザープロフィール・事実抽出
├── search.go        # KV-based keyword search (TF-IDF lite)
├── kotodama.toml
├── deploy config
└── go.mod
```

**3 層メモリ設計**:

| 層 | OpenClaw 相当 | KV Key Pattern | 保持期間 | 用途 |
|---|---|---|---|---|
| **Short-term** | Current conversation | `mem.{uid}.st.{conv_id}.{seq}` | Session | LLM に直接送信する直近 N ターン |
| **Long-term** | Past conversations (vector search) | `mem.{uid}.lt.{date}.{hash}` | 永続 | 過去会話の要約。キーワード検索で取得 |
| **Semantic** | Distilled knowledge | `mem.{uid}.sem.{category}` | 永続 (上書き更新) | 名前, 役割, 好み, 定型タスク, 連絡先 |

**Short-term memory** (Ring Buffer):

```
mem.{uid}.st.{conv_id}.meta   → { turn_count, created_at, platform }
mem.{uid}.st.{conv_id}.0      → { role: "user", text: "...", ts: ... }
mem.{uid}.st.{conv_id}.1      → { role: "assistant", text: "...", ts: ... }
...
最大 50 ターン保持。超過分は long-term に圧縮移行。
```

**Long-term memory** (要約 + キーワード検索):

```
会話終了時 or 50 ターン超過時:
  1. LLM で会話を 3-5 行に要約
  2. キーワード抽出 (名詞, 固有名詞, 日付)
  3. KV に保存:
     mem.{uid}.lt.{YYMMDD}.{hash}.summary  → 要約テキスト
     mem.{uid}.lt.{YYMMDD}.{hash}.keywords → ["keyword1", "keyword2", ...]
     mem.{uid}.lt.idx.{keyword}            → ["{YYMMDD}.{hash}", ...] (逆引き)

検索:
  SearchLongTerm(query) → query をキーワード分割 → idx 逆引き → summary 取得
  → 関連度順にソート → top-K を返却
```

**Semantic memory** (プロフィール):

```
mem.{uid}.sem.name        → "Jun Kawasaki"
mem.{uid}.sem.role        → "AI architect, founder of etzhayyim.com"
mem.{uid}.sem.preferences → { "language": "ja", "model": "claude-sonnet-4-6", ... }
mem.{uid}.sem.contacts    → [{ name: "...", relation: "...", platform: "..." }, ...]
mem.{uid}.sem.routines    → [{ time: "09:00", task: "morning briefing", ... }, ...]
mem.{uid}.sem.facts       → [{ fact: "...", source_conv: "...", date: "..." }, ...]

毎会話後に LLM で新事実を抽出 → 既存 semantic memory にマージ/更新
```

### 3.5 os-skills — スキルレジストリ & エグゼキュータ

**役割**: OpenClaw の AgentSkills / ClawHub に相当。
各 Skill は **独立した WASM module** として実行され、完全にサンドボックス化される。

```
60-apps/etzhayyim-project-os/wasm/etzhayyim-wasm-os-skills-<nanoid>/
├── main.go              # Skill registry + executor
├── registry.go          # Skill 登録・検索・有効化管理
├── executor.go          # Skill 呼び出し (XRPC to skill Apps)
├── builtin_fs.go        # Built-in: File operations (KV-backed virtual FS)
├── builtin_web.go       # Built-in: Web fetch, scraping
├── builtin_calc.go      # Built-in: Calculator, date/time
├── builtin_notify.go    # Built-in: Notification dispatch
├── kotodama.toml
├── App manifest
└── go.mod
```

**Skill 定義 (OpenClaw skill.json 相当)**:

```go
type SkillDefinition struct {
    ID          string            // "github-pr", "notion-page", "smart-home"
    Name        string            // "GitHub PR Manager"
    Description string            // LLM が選択判断に使う説明
    Version     string
    Author      string
    Tools       []ToolDefinition  // LLM に公開する tool 定義
    Permissions []string          // 必要な権限 ("http:github.com", "kv:write")
    AppID   string            // 実装 App の nanoid (外部 skill の場合)
}

type ToolDefinition struct {
    Name        string
    Description string
    Parameters  map[string]ParamDef // JSON Schema 形式
    ReturnType  string
    RiskLevel   string // "safe" | "caution" | "dangerous" | "forbidden"
}
```

**Skill カテゴリ (初期実装)**:

| カテゴリ | Skills | 実装方式 |
|---|---|---|
| **Built-in** | ファイル操作, Web検索, 計算, 通知 | os-skills 内蔵 |
| **Productivity** | GitHub, Linear, Notion, Google Calendar | 既存 performer App 連携 |
| **Communication** | Email送信, Slack DM, SMS | os-messaging-component 経由 |
| **Smart Home** | SwitchBot, Home Assistant | 外部 API コール |
| **Media** | 画像生成, 音声文字起こし | 外部 API or etzhayyim-browserless |
| **Developer** | コード実行, Git操作, Deploy | os-runner 連携 |
| **etzhayyim Platform** | News投稿, 記事検索, Analytics | 既存 etzhayyim App 連携 |

**既存 etzhayyim App との統合**: 既存の performer App (news-mcp, threads-mcp, www-crawler 等) は XRPC で呼び出すことで、そのまま Skill として利用可能。新しい App を作る必要はない。

```
os-agent → tool_call: "search_news"
  → os-skills/Execute("etzhayyim-news", "search", {query: "..."})
    → XRPC → news-mcp (r5wvpkot.etzhayyim.com)
    → result 返却
```

### 3.6 os-runner — サンドボックス実行環境

**役割**: OpenClaw の「シェル実行」「ブラウザ操作」に相当。
WASM 内で直接実行不可な操作を安全に委任する。

```
60-apps/etzhayyim-project-os/wasm/etzhayyim-wasm-os-runner-<nanoid>/
├── main.go
├── shell.go        # etzhayyim-browserless 経由のシェルコマンド実行
├── browser.go      # Playwright ブラウザ自動操作
├── scraper.go      # Web scraping (www-crawler 連携)
├── kotodama.toml
├── App manifest
└── go.mod
```

**実行フロー**:

```
os-agent → tool_call: "run_shell" { command: "ls -la /project" }
  → os/consent.evaluate("run_shell", {command}, risk=dangerous)
    → User approval required (Consent UI に表示)
    → Approved
  → os-runner → XRPC → etzhayyim-browserless (kotodama-runtime:8080)
    → Playwright: page.evaluate() で sandboxed 実行
    → stdout/stderr 返却
  → os/audit.log(tool="run_shell", command="ls -la", result=...)
```

**ブラウザ操作**:

```
os-agent → tool_call: "browse" { url: "https://...", action: "screenshot" }
  → os-runner → etzhayyim-browserless
    → Playwright: goto(url) → screenshot() → base64 返却
  → LLM が画像を解析して次のアクションを決定
```

### 3.7 os-ui — Web フロントエンド (既存拡張)

既存の `os-ui-6s80i2ya` を拡張。OpenClaw の Web Chat UI 相当。

**追加ページ**:

| ページ | Path | 機能 |
|---|---|---|
| Chat | `/` | メイン会話 UI (OpenClaw のチャット画面) |
| Skills | `/skills` | Skill 一覧・有効化・設定 |
| Memory | `/memory` | 3 層メモリの閲覧・編集 |
| Consent | `/consent` | 承認待ちキュー (既存) |
| Settings | `/settings` | LLM プロバイダ, Platform 接続, プロフィール |
| Audit | `/audit` | 行動ログ閲覧 (既存) |

## 4. プラットフォーム接続設定 (OpenClaw onboarding 相当)

OpenClaw の「Setup Wizard」に相当する設定フローを os-ui `/settings` で提供。

```
Settings → Platforms
  ├── Discord:    Bot Token + Guild ID → Webhook URL 自動生成
  ├── Telegram:   Bot Token (BotFather) → Webhook 自動登録
  ├── WhatsApp:   Meta Business API credentials
  ├── Slack:      OAuth App → Events API subscription
  ├── LINE:       Channel Access Token + Channel Secret
  ├── Signal:     signal-cli REST API endpoint
  ├── iMessage:   macOS native bridge (Tauri desktop only)
  └── Web:        Always enabled (os-ui built-in)

Settings → AI Model
  ├── Provider:   Anthropic / OpenAI / Google / OpenRouter / Ollama
  ├── API Key:    Encrypted in KV (performer ScopeUser)
  ├── Model:      Provider-specific model selection
  └── Fallback:   Secondary provider on failure

Settings → Profile (Semantic Memory bootstrap)
  ├── Name, Role, Language
  ├── Timezone, Work hours
  └── Custom instructions (system prompt override)
```

## 5. Proactive Scheduling (OpenClaw のタスクスケジューリング)

既存 `os-scheduler-component` を拡張。performer の `reminder` 機能を活用。

```
┌─────────────────────────────────────────────┐
│  os-scheduler (App)                      │
│                                              │
│  Reminder Engine (performer.Reminder)        │
│  ├── Cron-like: "0 9 * * *" → morning brief │
│  ├── Interval: every 30min → inbox check    │
│  ├── One-shot: "2026-03-04 14:00" → meeting │
│  └── Reactive: on-event → notification      │
│                                              │
│  Proactive Actions:                          │
│  ├── Morning briefing (calendar + news)     │
│  ├── Inbox summary (unread across platforms)│
│  ├── Task follow-up ("Did you finish X?")   │
│  ├── Smart reminders (from conversation)    │
│  └── Routine automation (user-defined)      │
└──────────────┬──────────────────────────────┘
               │ XRPC
               ▼
         os-agent → os-messaging-component → Platform に proactive 送信
```

## 6. App 構成一覧

| App | nanoid | Type | KV | 依存先 |
|---|---|---|---|---|
| `os-agent` | 新規割当 | API-only | Yes | os-llm, os-memory, os-skills, os-runner, consent, budget, audit |
| `os-llm` | 新規割当 | API-only | Yes (API key 暗号化) | External LLM APIs |
| `os-memory` | 新規割当 | API-only | Yes (3 層メモリ全量) | os-llm (要約・事実抽出) |
| `os-skills` | 新規割当 | API-only | Yes (registry) | 各 performer App |
| `os-runner` | 新規割当 | API-only | No | etzhayyim-browserless |
| `os-scheduler` | 既存拡張 | API-only | Yes (reminders) | os-agent |
| `os-messaging` | 既存拡張 | API-only | Yes | os-agent, native bridge, cloud messaging APIs |
| `os-runtime` | 既存拡張 | API-only | Yes | consent, budget, audit, sync |
| `os-ui` | `6s80i2ya` | SSG+API | No | os-agent, os-skills, os-memory |

**合計**: 新規 App 4 個 + 既存拡張 4 個 + 既存 UI 1 個 = **9 Apps**

## 7. 通信フロー全体図

```
[WhatsApp] ─┐
[Telegram] ─┤
[Discord]  ─┤  Webhook
[Slack]    ─┼──────────▶ os-messaging ──gRPC──▶ os-agent
[Signal]   ─┤            {nanoid}.etzhayyim.com     {nanoid}.etzhayyim.com
[LINE]     ─┤                                      │
[iMessage] ─┤                              ┌───────┼───────┐
[os-ui]    ─┘                              │       │       │
                                        XRPC    XRPC    XRPC
                                           │       │       │
                                           ▼       ▼       ▼
                                       os-llm  os-memory  os-skills
                                                             │
                                                      XRPC to existing
                                                      performer Apps
                                                             │
                                              ┌──────────────┼──────────┐
                                              ▼              ▼          ▼
                                          news-mcp     threads-mcp   www-crawler
                                          r5wvpkot     br8bojxp      o0dqx491
```

## 8. KV Store 設計

全 App は NATS JetStream KV (`wasi:keyvalue/store`) を使用。

| Prefix | Owner | Purpose |
|---|---|---|
| `msg.*` | os-messaging | Platform config, user mapping |
| `ag.*` | os-agent | Conversation state, settings |
| `llm.*` | os-llm | Provider config, API keys (encrypted) |
| `mem.*` | os-memory | 3-tier memory (short/long/semantic) |
| `sk.*` | os-skills | Skill registry, user enable/disable |
| `sch.*` | os-scheduler | Reminders, cron definitions |
| `pf.*` | performer (共通) | Performer framework standard prefix |

## 9. 実装フェーズ

### Phase 1: Core Loop (MVP)
**目標**: Web UI からチャットして AI が応答 + 基本 tool 実行

1. `os-llm` — Claude API 単一プロバイダ
2. `os-memory` — Short-term のみ (会話履歴 KV)
3. `os-agent` — Message → LLM → Response ループ
4. `os-ui` 拡張 — Chat ページ追加
5. `os-skills` — Built-in skills のみ (Web検索, 計算)

### Phase 2: Memory & Multi-Platform
**目標**: 過去を覚える AI + Discord/Telegram 連携

6. `os-memory` 拡張 — Long-term + Semantic memory
7. `os-messaging-component` 拡張 — Discord + Telegram adapter
8. `os-llm` 拡張 — OpenAI + OpenRouter 追加
9. `os-skills` 拡張 — etzhayyim platform skills (news, threads, crawler)

### Phase 3: Full Experience
**目標**: OpenClaw 同等の体験

10. `os-messaging-component` 拡張 — 全 8 platform
11. `os-runner` — Shell + Browser automation
12. `os-scheduler` 拡張 — Proactive automation
13. `os-skills` 拡張 — Smart Home, Developer tools
14. `os-ui` — Skills marketplace, Memory viewer, Settings wizard
15. Consent + Budget + Audit 統合

## 10. OpenClaw との差別化ポイント

| 観点 | OpenClaw | etzhayyim OS |
|---|---|---|
| **実行環境** | ローカル Node.js | K8s kotodama runtime (WASM sandbox) |
| **安全性** | OS レベルのファイルアクセス | WASM サンドボックス + Consent UI |
| **スケール** | 単一マシン | K8s クラスタ (水平スケール) |
| **Skill 隔離** | Node.js process (同一信頼境界) | 各 Skill が独立 WASM module |
| **経済ガバナンス** | API コスト自己管理 | GCC トークン + Budget enforcement |
| **可用性** | マシン起動時のみ | 24/7 クラウド常駐 |
| **プライバシー** | 完全ローカル | Per-user KV isolation (ScopeUser) |
| **デスクトップ連携** | 直接 OS アクセス | Tauri + native bridge (optional) |

## 11. Proto 定義 (新規)

```
proto/etzhayyim/os/v1/
├── agent.proto      # agent.chat, Conversation management
├── messaging.proto  # UnifiedMessage, inbound/outbound routing RPC
├── llm.proto        # ChatCompletion, ToolCall definitions
├── memory.proto     # GetProfile, SearchLongTerm, StoreFact
├── skills.proto     # ListSkills, ExecuteSkill, RegisterSkill
└── runner.proto     # RunShell, BrowseURL, Scrape
```

## 12. 技術的制約と対策

| 制約 | 対策 |
|---|---|
| TinyGo 0.35.0 必須 | Build env 固定 (`~/sdk/tinygo-0.35.0`) |
| HTTP transport 要件 | `ydnar/wasi-http-go` 使用 (LLM API コール) |
| WASI IO 4096 byte limit | chunkedResponseWriter / chunkedBodyReader |
| KV list pagination | offset + limit 必須 (全 API) |
| TinyGo regexp 禁止 | strings パッケージで代替 |
| TinyGo json.NewDecoder 禁止 | io.ReadAll + json.Unmarshal |
| Method-prefixed mux 未対応 | `mux.HandleFunc("/path", ...)` のみ |
| Go func() stall | 同期処理のみ、goroutine 禁止 |
