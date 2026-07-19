# etzhayyim OS Drive Sync - Design Document

## Overview

etzhayyim OS への drive 統合機能。Google Drive/Dropbox 風のローカル-クラウド双方向同期を実現し、project-hub の git repository 単位でプロジェクト管理・共有を行う。

## DoDAF v2 Architecture

### Capability (CV-1)

| ID | Capability | Description |
|---|---|---|
| CAP-OS-002 | Local-Cloud File Synchronization | ローカルファイルシステムとクラウドストレージの双方向同期 |
| CAP-OS-003 | Multi-User Project Management | user/org_id ごとのプロジェクト管理 |
| CAP-OS-004 | Git Repository Integration | project-hub との git 連携 |
| CAP-OS-005 | Sync Conflict Resolution | 同期競合の検出・解決 |
| CAP-OS-006 | Unified MCP API | App 経由の統一 XRPC API |

### Activity (OV-5b)

1. **File Watch & Upload**: ローカルファイル変更監視 → クラウドアップロード
2. **Remote Change Pull**: クラウド変更検出 → ローカル適用
3. **Project Binding**: drive folder と git repository の紐付け
4. **User/Org Project Management**: user/org_id スコープでのプロジェクト CRUD
5. **Conflict Detection & Resolution**: timestamp/checksum 比較による競合解決

### Performer (OV-2)

| Performer | Type | Location | Description |
|---|---|---|---|
| `drive-sync-component` | App component | `kotodama-system` ns | MCP API 提供 (TinyGo WebAssembly) |
| `os-drive-sync` | Tauri backend | Desktop (Rust) | ローカルファイル監視・同期エンジン |
| `project-hub-integration` | MCP component | `kotodama-system` ns | git repository メタデータ管理 |

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ etzhayyim OS (Tauri Desktop App)                                │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ os-drive-sync (Rust)                                 │  │
│  │  - File watcher (notify-rs)                          │  │
│  │  - Local sync engine                                 │  │
│  │  - Conflict resolver                                 │  │
│  └─────────────┬────────────────────────────────────────┘  │
│                │ XRPC (MCP)                                 │
└────────────────┼────────────────────────────────────────────┘
                 │
                 │ https://ds7yn3kw.etzhayyim.com/api/mcp
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ App Runtime (Kubernetes)                             │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ drive-sync-component (TinyGo WebAssembly)            │  │
│  │  - MCP XRPC server                                   │  │
│  │  - Project manager (user/org_id scope)               │  │
│  │  - Git binding manager (project-hub integration)     │  │
│  └─────────────┬────────────────────────────────────────┘  │
│                │                                            │
│  ┌─────────────┴─────────────┬──────────────────────────┐  │
│  │ Redis (org-redis)         │ NATS JetStream           │  │
│  │ - Project metadata        │ - File change events     │  │
│  │ - Sync state              │ - Pub/Sub                │  │
│  └───────────────────────────┴──────────────────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Data Model

### Project

```json
{
  "id": "proj_abc123",
  "name": "My Project",
  "org_id": "org_xyz",
  "user_id": "user_123",
  "git_repo_url": "https://github.com/org/repo.git",
  "local_path": "/Users/user/etzhayyim/My Project",
  "remote_path": "/drive/org_xyz/user_123/proj_abc123",
  "created_at": 1676419200
}
```

### Sync Status

```json
{
  "local_path": "/Users/user/etzhayyim/My Project/file.txt",
  "remote_path": "/drive/org_xyz/user_123/proj_abc123/file.txt",
  "last_sync": 1676419200,
  "state": "synced" | "pending" | "conflict" | "error"
}
```

## API Surface

### MCP Endpoint

- **URL**: `https://ds7yn3kw.etzhayyim.com/api/mcp`
- **Protocol**: XRPC (Connect protocol via Envoy Gateway)
- **nanoid**: `ds7yn3kw` (drive-sync component identifier)

### MCP Tools

| Tool | Description |
|---|---|
| `watch_directory` | ローカルディレクトリの監視開始 |
| `unwatch_directory` | 監視停止 |
| `get_sync_status` | パスの同期状態取得 |
| `trigger_sync` | 手動同期実行 |
| `create_project` | プロジェクト作成 (user/org scope) |
| `list_projects` | プロジェクト一覧 |
| `bind_git_repo` | git repository 紐付け (project-hub) |
| `get_project` | プロジェクト詳細取得 |
| `delete_project` | プロジェクト削除 |

## Sync Algorithm

### Upload (Local → Cloud)

1. ファイル変更検出 (notify-rs)
2. Checksum 計算 (SHA-256)
3. Remote checksum と比較
4. 差分がある場合のみアップロード
5. Redis に sync state 保存

### Download (Cloud → Local)

1. NATS topic `drive.changes.{org_id}.{user_id}` を subscribe
2. Remote file change event 受信
3. Local checksum と比較
4. 差分がある場合のみダウンロード
5. Local filesystem に書き込み
6. Redis に sync state 保存

### Conflict Resolution

1. Local/Remote の両方が `last_sync` 以降に変更されている場合、conflict
2. Conflict state を Redis に保存
3. UI で conflict 通知 (両バージョンを保持、ユーザーに選択を促す)

## project-hub Integration

### Git Binding Flow

1. `bind_git_repo(project_id, git_repo_url)` 呼び出し
2. project-hub に repository metadata 登録
3. `.etzhayyim/project.json` を local/remote に作成
4. Git commit/push で project metadata を共有
5. 他のメンバーが pull → 自動的に同じ project に参加

### Shared Project Structure

```
/drive/org_xyz/shared/proj_abc123/
  .etzhayyim/
    project.json       # Project metadata (org_id, members, git_repo_url)
  src/
  docs/
  ...
```

## Security

- **E2EE**: ファイル内容は暗号化してクラウドに保存 (AES-256-GCM)
- **Access Control**: org_id/user_id による厳格なスコープ制御
- **Audit Log**: 全 sync 操作を NATS に publish (監査証跡)

## Deployment

### App Component

- **Path**: `60-apps/etzhayyim-project-os/wasm/drive-sync-component/`
- **WADM**: `wadm/drive-sync.wadm.yaml`
- **HTTPRoute**: `k8s/http-routes.yaml`
- **Build**: Tekton pipeline `etzhayyim-wasm-sync` → `ghcr.io/etzhayyim/drive-sync-component:latest`

### Tauri Backend

- **Path**: `60-apps/etzhayyim-project-os/legacy-runtime/sys-os-jt4qfzv1/src-tauri/`
- **Module**: `src-tauri/src/drive_sync.rs`
- **Dependencies**: `notify`, `tokio`, `sha2`, `tonic` (XRPC client)

## Implementation Phases

1. **Phase 1 (Design)**: WIT/WADM/capabilities.jsonld 設計 ✅
2. **Phase 2 (MCP Component)**: TinyGo で drive-sync-component 実装
3. **Phase 3 (Tauri Backend)**: Rust で file watcher/sync engine 実装
4. **Phase 4 (project-hub Integration)**: Git binding API 実装
5. **Phase 5 (Testing)**: E2E sync test, conflict resolution test
6. **Phase 6 (Production)**: Flux CD で deploy, monitoring 追加

## References

- DoDAF v2 DM2: https://dodcio.defense.gov/Library/DoD-Architecture-Framework/
- App component model documentation
- WebAssembly Interface Types (WIT): https://component-model.bytecodealliance.org/design/wit.html
