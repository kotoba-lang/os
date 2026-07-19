# Agent Ops Runner on Kubernetes (Performers Control Plane) Design

## Goal
`/jobs/scheduler-tick` (etzhayyim backend) が起点となって、`jt4qfzv1` (sys-os) の Agent Ops による自動実行で、次を **高確度** に成立させる。

- 対象リポジトリの checkout（必要なら shallow / mirror）
- git 認証（push できる）
- PR 作成（gh / GitHub API）
- 実行環境の起動/終了を最適化（Bazel + image cache で高速化）

ここでは「K8s Job (Pod) を Performers control plane が生成し、1 automation run = 1 Job」として設計する。

## Non-Goals
- LLM の品質最適化（プロンプト設計やモデル選定は別）
- monorepo の全ビルドを毎回走らせる（必要な場合のみ）

## High-Level Architecture

```mermaid
flowchart LR
  A["App scheduler binding\ncron-etzhayyim-scheduler\n(/jobs/scheduler-tick)"] --> B["etzhayyim backend\n(automation store + tick)"]
  B --> C["Performers Control Plane\n(Runner Manager)"]
  C --> D["Kubernetes Job\nagent-ops-runner"]
  D --> E["GitHub\nclone/push/PR"]
  D --> F["Cache PV / Remote Cache\n(Bazel, git mirror, npm, go)"]
  D --> G["Logs/Status sink\n(App state + scheduler thread)"]
```

## Control Plane Responsibilities (Runner Manager)
Performers control plane 側に Runner Manager を置く（既存の deploy/agent 管理と同じ平面）。

- `CreateRun`:
  - automation の `project`, `work_dir`, `prompt` を受け取る
  - 実行対象 repo（URL/branch）を決定（project.jsonld / scheduler.jsonld 参照）
  - K8s Job を作成
- `ObserveRun`:
  - Pod 状態 / exit code / log を収集
  - PR URL / エラーを etzhayyim backend に返す（Scheduler thread を更新）
- `Cleanup`:
  - TTLAfterFinished により Job/Pod を自動削除
  - 必要なら workspace volume の掃除

## Runner Pod Template
### コンテナ構成
- `initContainer: repo-sync`
  - repo を workspace に展開（clone/pull）
  - mirror / reference clone を使って高速化
- `mainContainer: agent-ops-runner`
  - Agent Ops ループを開始
  - ツール呼び出し（`git`, `gh`, `bazel` 等）を許可
- optional: `sidecar: bazel-remote-proxy`（クラスタ内 remote cache を使うなら不要）

### Workspace と Cache
ワークスペースは run ごとに isolated、cache は共有できるように分離する。

- `/workspace`:
  - run ごとの作業ディレクトリ
  - `emptyDir` でも良いが、clone が重い場合は RWX を検討
- `/cache`:
  - 共有キャッシュ用（PVC）
  - サブパスで分割して衝突回避

推奨マウント（例）:
- `/cache/bazel-disk` (PVC)
- `/cache/bazel-repo` (PVC)
- `/cache/git-mirror` (PVC)
- `/cache/npm` (PVC)
- `/cache/go` (PVC)

### Bazel 実行を速くする設定
runner 内の `bazel` 実行は以下の二段構え:

1. Remote cache（最優先）
   - `--remote_cache=http://bazel-remote.legacy-runtime-shared.svc.cluster.local:8080`
   - 許せるなら `--remote_upload_local_results=true`
2. Local disk cache（PVC）
   - `--disk_cache=/cache/bazel-disk`
   - `--repository_cache=/cache/bazel-repo`
   - `--output_user_root=/cache/bazel-ou`（ノード依存を避けるなら run 毎に分ける）

### Container Image / Image Cache（起動を速くする）
runner image は「必要ツール全部入り」で固定化し、pod 起動が apt/yum に依存しないようにする。

runner image に入れるもの:
- `git`, `openssh-client`
- `gh` CLI
- `bazelisk`（もしくは bazel）
- `node`, `npm`
- `go`（必要なら）
- `jq`, `ripgrep`, `python3`（補助）

image cache 戦略:
- `imagePullPolicy: IfNotPresent`
- 主要ノードに pre-pull する DaemonSet（`agent-ops-runner-prepull`）を用意
- ベースイメージは更新頻度を落とし、タグは digest pin を基本

## GitHub 認証設計
### 推奨: GitHub App (Installation Token)
長期トークンを配らず、Runner Manager が短命 token を発行して pod に渡す。

- Secret 取り回し:
  - GitHub App private key は control plane のみが保持（KMS/Secret Manager 推奨）
  - Runner pod には短命 `GITHUB_TOKEN` だけを env で注入（寿命 1h 程度）
- メリット:
  - token 流出時の被害範囲が最小
  - repo への権限を細かく制御可能

### 代替: Fine-grained PAT
最短で動かす場合。K8s Secret に `GH_TOKEN` を保存し、Runner pod に注入。

必要 scope:
- push する repo への `contents:write`
- PR 作成の `pull_requests:write`

### gh / git の使い分け
- `gh`:
  - `GH_TOKEN` / `GITHUB_TOKEN` が env にあると non-interactive で動作
  - `gh pr create` / `gh pr view` など
- `git`:
  - HTTPS 推奨（SSH 鍵配布より管理が簡単）
  - `git remote set-url origin https://x-access-token:${GITHUB_TOKEN}@github.com/ORG/REPO.git`

## PR 作成フロー（runner 内）
最小の成功パス:
1. `git checkout -b codex/auto/<run-id>`
2. 変更を作成
3. `git status --porcelain` が空なら終了（No-op）
4. `git commit -am ...`（必要な add を含める）
5. `git push -u origin HEAD`
6. `gh pr create --title ... --body ... --base main --head <branch>`

冪等性:
- 既存 PR がある場合は `gh pr view` で拾う
- push 失敗時は backoff + 再試行（ただし二重実行防止の lock は control plane 側で）

## Lock / Concurrency
プロジェクト単位で「同時に複数の runner が同じ repo を書き換えない」保証が必要。

推奨:
- App state に `lock:repo:<org>/<repo>` を置き、TTL 付き CAS を control plane で取得
- lock を取れなければ run を delay（次 tick に回す）

## Observability
Runner Manager が以下を収集し、etzhayyim backend の scheduler thread に反映する。

- Pod phase / exit code
- 重要ログ（clone/push/pr URL）
- 生成された PR URL
- エラー（最終 4KB など）

ログ収集:
- `kubectl logs` 相当を control plane が取得して state store へ
- 長期保存するなら object storage に jsonl を置いて URL を返す

## Security
- Runner pod の ServiceAccount は最小権限（Job 作成などは不要、作成は control plane が行う）
- Pod は `runAsNonRoot`, `readOnlyRootFilesystem` を基本
- NetworkPolicy で egress を限定（GitHub, registry, remote cache, App runtime sidecar 程度）
- secrets は env 注入 + short-lived token を優先

## Rollout Plan
1. Runner image を作成（ツール同梱、digest pin）
2. control plane に Runner Manager を追加（Job 作成 + log 回収）
3. etzhayyim backend の automation 実行を「legacy runtime invoke 直叩き」から「Runner Manager 呼び出し」に切替
4. cache PV / remote cache を追加し、ウォームアップ DaemonSet を投入
5. 本番 automation を段階的に enable
