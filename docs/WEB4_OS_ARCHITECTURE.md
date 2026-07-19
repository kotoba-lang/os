# etzhayyim OS — Web4 Local Interface Architecture

## Overview

etzhayyim OS is the **local interface** to the Web4 agent-populated internet.
It runs on the user's desktop (Tauri + App) and serves as the human's
gateway to manage, observe, and govern AI agents operating across local
and cloud environments.

**Web4 thesis**: The majority of internet participants will be AI agents
acting on behalf of humans or autonomously. etzhayyim OS ensures the human
retains oversight, trust control, and economic governance over their agents.

## Web4 6-Layer → etzhayyim OS Mapping

| Web4 Layer | etzhayyim OS Implementation | WIT Interfaces |
|---|---|---|
| **Environmental** | Tauri desktop (OS access, filesystem, notifications, GPU) | `wasi:filesystem`, Tauri commands |
| **Infrastructure** | App (local WASM runtime) + NATS (local mesh) | `wasi:http`, `wasi:keyvalue`, `wasi:config` |
| **Data/Knowledge** | drive-sync + Redis KV + gitstate | `etzhayyim:os/sync`, `wasi:keyvalue` |
| **Agent** | Agent runtime + Automaton (survival, memory, soul) | `etzhayyim:os/agent-runtime`, `etzhayyim:agent`, `etzhayyim:automaton` |
| **Behavioral** | Consent UI + Directory + Natural language interface | `etzhayyim:os/consent`, `etzhayyim:os/directory` |
| **Governance** | Budget enforcement + Audit trail + Policy engine | `etzhayyim:os/budget`, `etzhayyim:os/audit`, `etzhayyim:automaton/policy` |

## Component Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        etzhayyim OS Desktop                          │
│                     (Tauri + App)                          │
│                                                                  │
│  ┌──────────────┐    ┌──────────────────────────────────────┐   │
│  │  Tauri Shell  │    │  OS Runtime Component (etzhayyim-os-runtime) │
│  │  (Svelte UI)  │───▶│                                      │   │
│  │              │ gRPC│  exports:                             │   │
│  │  - Agent Dash │    │    etzhayyim:os/agent-runtime              │   │
│  │  - Consent UI │    │    etzhayyim:os/consent                    │   │
│  │  - Directory  │    │    etzhayyim:os/directory                  │   │
│  │  - Budgets    │    │    etzhayyim:os/budget                     │   │
│  │  - Audit Log  │    │    etzhayyim:os/audit                      │   │
│  │  - Sync Status│    │    etzhayyim:os/sync                       │   │
│  └──────────────┘    │                                      │   │
│                       │  imports:                             │   │
│                       │    etzhayyim:agent, etzhayyim:automaton/*       │   │
│                       │    etzhayyim:web3/wallet                   │   │
│                       │    etzhayyim:mesh/*, etzhayyim:inference        │   │
│                       │    etzhayyim:clerk/authn/authz             │   │
│                       └──────────┬───────────────────────────┘   │
│                                  │ wRPC/NATS                     │
│                       ┌──────────▼───────────────────────────┐   │
│                       │     Local Agent Pool (WASM sandbox)   │   │
│                       │                                       │   │
│                       │  Agent A ──── Agent B ──── Agent C    │   │
│                       │  (automaton)  (agent)     (automaton) │   │
│                       │                                       │   │
│                       │  Each agent has:                      │   │
│                       │  - Isolated WASM execution            │   │
│                       │  - Resource limits (CPU/mem/GPU)      │   │
│                       │  - GCC wallet (for automatons)        │   │
│                       │  - Policy engine (constitutional)     │   │
│                       │  - Memory (5-tier hierarchical)       │   │
│                       └───────────────────────────────────────┘   │
│                                  │                                │
│                                  │ NATS JetStream                │
│                                  ▼                                │
│                       ┌───────────────────────┐                  │
│                       │   Inference Mesh Node  │                  │
│                       │   (WebGPU local GPU)   │                  │
│                       └───────────────────────┘                  │
└──────────────────────────────────┬──────────────────────────────┘
                                   │ NATS (internet)
                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                    etzhayyim.com Cloud (App K8s)                  │
│                                                                   │
│   ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │
│   │ Cloud Agent │  │ Calendar   │  │ Communicator│  │ Collector│  │
│   │ Pool       │  │ Provider   │  │ Provider    │  │ Provider │  │
│   └────────────┘  └────────────┘  └─────────────┘  └──────────┘  │
│                                                                   │
│   ┌────────────┐  ┌────────────┐  ┌─────────────────────────┐   │
│   │ Mesh Nodes │  │ Wallet     │  │ NATS JetStream Cluster  │   │
│   │ (GPU)      │  │ Provider   │  │ (state sync backbone)   │   │
│   └────────────┘  └────────────┘  └─────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

## Agent Lifecycle State Machine

```
              spawn()
                │
                ▼
           ┌─────────┐
           │ spawning │
           └────┬─────┘
                │ WASM loaded + sandbox ready
                ▼
           ┌─────────┐         pause()         ┌────────┐
           │ running  │────────────────────────▶│ paused │
           └────┬─────┘                         └───┬────┘
                │         resume()                  │
                │◀──────────────────────────────────┘
                │
      ┌─────────┼──────────┐
      │         │          │
  stop()    migrate()   crash/OOM
      │         │          │
      ▼         ▼          ▼
 ┌─────────┐ ┌──────────┐ ┌────────┐
 │ stopped │ │migrating │ │ failed │
 └─────────┘ └────┬─────┘ └────────┘
                   │
                   │ state transfer complete
                   ▼
             (new agent at target)
```

## Consent Flow

When an agent attempts a dangerous action, the consent system intervenes:

```
Agent attempts tool call
        │
        ▼
automaton:policy.evaluate(tool-name, args, authority)
        │
        ├── risk = safe → execute immediately
        ├── risk = caution → log + execute
        ├── risk = dangerous → check delegations
        │       │
        │       ├── delegation exists + valid → auto-approve + execute
        │       └── no delegation → queue for human
        │               │
        │               ▼
        │       os:consent.request-approval()
        │               │
        │               ▼
        │       Tauri UI displays approval request
        │               │
        │               ├── User approves → execute + log
        │               ├── User denies → agent receives error + log
        │               └── TTL expires → treated as denial + log
        │
        └── risk = forbidden → deny immediately + log
```

## Budget Enforcement Flow

Pre-action check prevents overspending:

```
Agent requests tool execution
        │
        ▼
os:budget.check-allowance(agent-id, gcc-cost, api-calls)
        │
        ├── budget.frozen = true → deny
        ├── gcc_spent + cost > limit → deny (+ auto-freeze if configured)
        ├── api_calls + 1 > limit → deny
        └── within budget → allow
                │
                ▼
        Execute tool call
                │
                ▼
        os:budget.record-spend(agent-id, category, amount, desc)
                │
                ▼
        Check alert thresholds → notify if exceeded
```

## Local ↔ Cloud Agent Migration

```
Local Machine                              Cloud (App K8s)
─────────────                              ─────────────────────

1. os:agent-runtime.migrate(agent-id, cloud)
        │
        ▼
2. Serialize agent state:
   - automaton:memory (all 5 tiers)
   - automaton:soul (current snapshot)
   - agent:get-history (conversation)
   - wasi:keyvalue (all KV pairs)
        │
        ▼
3. NATS JetStream publish
   Subject: etzhayyim.os.migrate.{agent-id}
        │                                  │
        │                                  ▼
        │                          4. Cloud receives state
        │                          5. kubectl apply -f <repo-deploy-config>
        │                          6. Restore state to KV
        │                          7. Start agent component
        │                                  │
        ▼                                  ▼
8. os:agent-runtime state → stopped   9. Agent alive on cloud
10. os:directory updates source=cloud
11. os:audit logs migration event
```

## Trust Graph Model

```
User (human) ── sets trust level ──▶ Agent

Trust Levels:
  unknown    → No interaction history. Cannot communicate.
  untrusted  → Explicitly blocked. All messages rejected.
  limited    → Can receive messages. Cannot invoke tools on user's agents.
  trusted    → Can invoke tools within policy constraints. Standard level.
  verified   → Cryptographically verified identity + trusted. Full access.

Trust propagation:
  - Trust is NOT transitive (Agent A trusts Agent B ≠ User trusts Agent B)
  - Each user manages their own trust graph
  - Trust level maps to automaton:policy authority-level:
      verified  → creator
      trusted   → peer
      limited   → external
      untrusted → (blocked, never evaluated)
```

## Protocol Adapter (Unified Access)

The OS runtime component exposes all 6 interfaces via 4 protocols:

```
OS Runtime Component (etzhayyim:os@0.1.0)
  ├── wRPC/NATS  → Internal component calls (zero-cost)
  ├── MCP        → External AI agents (Claude, etc.)
  ├── cross-actor        → Autonomous agent-to-agent
  └── XRPC → Tauri Svelte frontend
```

MCP tool mapping:
```
os.agent.spawn        → agent-runtime.spawn()
os.agent.stop         → agent-runtime.stop()
os.agent.list         → agent-runtime.list-local()
os.consent.pending    → consent.get-pending()
os.consent.approve    → consent.approve()
os.directory.discover → directory.discover()
os.budget.status      → budget.get-status()
os.audit.query        → audit.query()
os.sync.status        → sync.get-status()
```

## Native Bridge Pattern

Platform-locked capabilities that cannot run in WASM are exposed via native
bridge providers. The WASM component imports the bridge interface; a native
host provider implements it.

```
WASM Component (os-messaging-component)
  └── import etzhayyim:native-bridge/imessage
        │
        │ wRPC link (WADM)
        ▼
Native Provider (imessage-native-provider)
  ├── Reads ~/Library/Messages/chat.db (SQLite)
  └── Sends via AppleScript `tell application "Messages"`
```

Current native bridges:
| Interface | Platform | Why Native |
|---|---|---|
| `etzhayyim:native-bridge/imessage` | macOS | chat.db + AppleScript required |

NOT native (WASM-internal via WebLLM):
- Text generation: `etzhayyim:inference/engine` (WebGPU LLM)
- Image generation: `etzhayyim:inference/engine` (WebGPU Stable Diffusion)
- Cloud fallback: `wasi:http/outgoing-handler` (OpenRouter API)

## legacy runtime to App Migration

The previous legacy 7-service architecture has been replaced:

| legacy runtime Service | App Component | Notes |
|---|---|---|
| etzhayyim-agent (core) | os-runtime-component | 6 OS interface exports |
| etzhayyim-agent (chat/AI) | os-ai-component | WebLLM inference |
| etzhayyim-agent (wellness) | os-wellbeing-component | automaton memory/soul |
| etzhayyim-messaging | os-messaging-component | + iMessage native bridge |
| etzhayyim-scheduler | os-scheduler-component | AI-driven scheduling |
| etzhayyim-system (Go) | os-system-component | AV, privacy, system |
| etzhayyim-imsg-legacy-runtime | imessage-native-provider | macOS capability provider |
| etzhayyim-llm (Python MLX) | WebLLM (WebGPU) | WASM-internal, no native |
| etzhayyim-agent-ops (XRPC) | os-runtime-component | Consolidated |

Infrastructure changes:
- legacy runtime SQLite state store → NATS KV (kvnats provider)
- legacy runtime in-memory pubsub → NATS Messaging
- App actors → App component instances
- legacy runtime workflows → App WADM app deployment

## WADM Local Topology

```yaml
# os-local.wadm.yaml — all components on single local host
components:
  # 6 OS service components (1 replica each)
  - os-runtime-component     :8080  (6 OS interface exports)
  - os-ai-component          :8081  (WebLLM + OpenRouter)
  - os-messaging-component   :8082  (iMessage + cloud messaging)
  - os-scheduler-component   :8083  (thread inbox + AI schedule)
  - os-system-component      :8084  (AV, privacy, system)
  - os-wellbeing-component   :8085  (becoming journey)

providers:
  - httpserver (HTTP incoming handler for all components)
  - kvnats (NATS KV — replaces legacy runtime SQLite state)
  - nats-messaging (NATS messaging — replaces legacy runtime pubsub)
  - imessage-native-provider (macOS iMessage bridge)

wRPC links:
  - os-messaging → imessage-native-provider (native-bridge/imessage)
  - all components → kvnats (wasi:keyvalue/store)
  - all components → httpserver (wasi:http/incoming-handler)
```

Full WADM manifest: `wasm/os-runtime-component/wadm/os-local.wadm.yaml`
