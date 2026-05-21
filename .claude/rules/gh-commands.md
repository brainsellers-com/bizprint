---
version: "1.1.0"
---

# gh コマンド運用ルール

Claude Code が gh コマンドを**一発で成功**させるためのルール集。
`gh` を使う前に必ず本ルールを参照する。

## GitHub 固有の制限

| 制限 | 回避策 |
|---|---|
| PR 作成者は自分の PR を approve できない | `gh pr merge --admin` でブランチ保護をバイパス |
| ネイティブな親子イシュー関係がない | 親イシューの本文にタスクリスト（Markdown チェックボックス）で子イシューをリンク、または GitHub Projects の Sub-issues 機能を利用 |
| ラベルは事前作成が必要 | `gh label create` で先に作成するか、既存ラベルを `gh label list` で確認 |

## Issue 操作

### 現在のユーザー名取得
```powershell
gh api user --jq ".login"
```

### イシュー作成
```powershell
gh issue create --title "<タイトル>" --body "<説明>" --assignee "@me"
```
- `--body` に複数行を渡す場合は PowerShell here-string `@'...'@` を使う（詳細は後述）。
- `--assignee "@me"` で現在のユーザーを自動割り当て。他ユーザーを指定する場合は `--assignee <username>`。

### イシュー一覧
```powershell
# オープン中のイシュー（既定）
gh issue list

# クローズ済みのみ
gh issue list --state closed

# 全件
gh issue list --state all

# ラベルで絞り込み
gh issue list --label "bug"

# JSON 出力
gh issue list --json number,title,state

# REST API 直接呼び出し（大量取得）
gh api "repos/{owner}/{repo}/issues?state=open&per_page=100"
```

### イシューのクローズ
```powershell
gh issue close <イシュー番号>

# Won't do 相当（理由付きクローズ）
gh issue close <イシュー番号> --reason "not planned"
```

### イシューにコメント投稿
```powershell
gh issue comment <イシュー番号> --body "<コメント>"
```

## PR 作成・参照

### PR 作成
```powershell
$desc = @'
## 概要
...
'@
gh pr create --title "<title>" --body $desc --base develop
```
- `--base` でマージ先ブランチを指定。
- ソースブランチの削除はマージ時に `--delete-branch` で指定する（作成時は不要）。

### 既存 PR の確認
```powershell
# 現在のブランチに紐づく open PR を探す
gh pr list --head <branch>

# 全状態（open + closed + merged）
gh pr list --head <branch> --state all

# PR 詳細
gh pr view <PR番号>

# diff（infra/ 変更検知などに）
gh pr diff <PR番号>

# JSON で情報取得
gh pr view <PR番号> --json state,author,baseRefName,headRefName

# コメント一覧
gh pr view <PR番号> --comments
```

### PR の状態確認
```powershell
# state を取得（OPEN / MERGED / CLOSED — 大文字）
gh pr view <PR番号> --json state --jq ".state"
```

### PR 作成者の取得
```powershell
gh pr view <PR番号> --json author --jq ".author.login"
```

## PR 承認・マージ

### 承認（他者が作成した PR）

LGTM + CI pass を確認後に実行する。

```powershell
gh pr review <PR番号> --approve
```

### 通常マージ（承認後）
```powershell
gh pr merge <PR番号> --merge --delete-branch
```

### バイパスマージ（自分が作成した PR）

GitHub では自分が作成した PR を自分で承認できない。
PR 作成者＝責任者の場合、`--admin` でブランチ保護ルールをバイパスして直接マージする。

```powershell
gh pr merge <PR番号> --merge --delete-branch --admin
```

- `--admin` は Repository Admin 以上の権限が必要。
- 承認ステップをスキップして直接マージする。

### マージ完了の確認（**必須**）

`gh pr merge` の出力だけでは信用しない。必ず state を確認する:

```powershell
gh pr view <PR番号> --json state --jq ".state"
# "MERGED" を確認
```

### マージコミット SHA の取得
```powershell
gh api "repos/{owner}/{repo}/pulls/<PR番号>" --jq ".merge_commit_sha"
```

## PR コメント・レビュー

```powershell
# コメント投稿（複数行は here-string）
$msg = @'
LGTM です。
マージお願いします。
'@
gh pr comment <PR番号> --body $msg

# コメント一覧取得
gh pr view <PR番号> --comments

# コメント一覧（JSON）
gh api "repos/{owner}/{repo}/issues/<PR番号>/comments" --jq ".[].body"
```

## CI（GitHub Actions）

```powershell
# 直近のワークフロー実行一覧
gh run list --limit 5

# 特定ブランチの run 一覧
gh run list --branch <branch> --limit 5

# ワークフロー指定
gh run list --workflow "CI" --limit 5

# run 詳細（ジョブ一覧含む）
gh run view <run_id>

# run のログ（失敗ジョブのみ — 大量ログを避けたい場合に有用）
gh run view <run_id> --log-failed

# run の全ログ
gh run view <run_id> --log

# run のジョブ一覧（JSON）
gh run view <run_id> --json jobs

# SHA から run を特定（CI 監視）
gh api "repos/{owner}/{repo}/actions/runs?head_sha=<SHA>&per_page=1"

# PR に紐づく check runs
gh api "repos/{owner}/{repo}/commits/<SHA>/check-runs" --jq ".check_runs[] | {name, status, conclusion}"

# run の再実行
gh run rerun <run_id>

# 失敗ジョブのみ再実行
gh run rerun <run_id> --failed
```

### PR の CI チェック状況

```powershell
# PR に紐づくチェック一覧（テキスト表示）
gh pr checks <PR番号>

# JSON で取得（プログラム的に判定する場合）
gh pr view <PR番号> --json statusCheckRollup
```

### GitHub Actions のステータス値

| フィールド | 値 | 意味 |
|---|---|---|
| `status` | `completed` | 実行完了（結論は `conclusion` で判定） |
| `status` | `in_progress` | 実行中 |
| `status` | `queued` | キュー待ち |
| `conclusion` | `success` | CI 通過 |
| `conclusion` | `failure` | CI 失敗 |
| `conclusion` | `cancelled` | キャンセル |
| `conclusion` | `skipped` | スキップ |

- `status` が `completed` になるまで待ち、`conclusion` で結果を判定する。

## API 直接呼び出し

```powershell
# REST（{owner}/{repo} は gh が自動解決）
gh api "repos/{owner}/{repo}/pulls/<PR番号>"
gh api "repos/{owner}/{repo}/issues?state=open&per_page=100"

# GraphQL
gh api graphql -f query='{ viewer { login } }'

# JSON フィルタ（--jq）
gh api "repos/{owner}/{repo}/pulls/<PR番号>" --jq ".merge_commit_sha"
```

- `{owner}/{repo}` は `gh api` が現在のリポジトリから自動解決する。

## PowerShell 固有の落とし穴

### 1. `gh` の出力を `python3 -c` にパイプしない
PowerShell のパイプは文字列解析が独自で、Python の f-string 内の `[`・`\"`・`#` 等が壊れる。また `.exe` 出力完了前にパイプが閉じられ「パイプを閉じています」エラーになる。

```powershell
# NG
gh api "repos/{owner}/{repo}/issues" | python3 -c "import sys,json; ..."

# OK: --jq を使う（gh 組み込み）
gh api "repos/{owner}/{repo}/issues" --jq ".[].title"

# OK: 一旦変数に受けて ConvertFrom-Json
$json = gh api "repos/{owner}/{repo}/issues"
$json | ConvertFrom-Json | ForEach-Object { ... }
```

### 2. 複数行文字列は here-string で渡す
コミットメッセージ・PR 説明文等は single-quoted here-string を使う。閉じる `'@` は列 0 に置く必要がある（インデント不可）。

```powershell
$desc = @'
## 概要
本文に $literal や `backtick` を含めても展開されない
'@
gh pr create --body $desc ...
```

- `@'...'@`（リテラル）が基本。変数展開が必要なときだけ `@"..."@`。

### 3. GraphQL クエリのクォート

```powershell
# シングルクォートで囲み、内部のダブルクォートは素のまま
gh api graphql -f query='{ repository(owner: "org", name: "repo") { issues { totalCount } } }'
```

ダブルクォートで囲む場合は内部のダブルクォートをバッククォートでエスケープする（`` `" ``）:

```powershell
gh api graphql -f query="{ repository(owner: `"org`", name: `"repo`") { id } }"
```

## `--json` + `--jq` で構造化出力を取得

gh は `--json` フラグで JSON 出力を指定し、`--jq` で jq フィルタを適用できる。`ConvertFrom-Json` を使わずに済む場面が多い:

```powershell
# PR の state を文字列で取得
gh pr view <PR番号> --json state --jq ".state"

# PR 一覧から番号とタイトルを取得
gh pr list --json number,title --jq '.[] | "\(.number): \(.title)"'

# イシュー一覧の特定フィールド
gh issue list --json number,title,state --jq '.[] | "\(.number) [\(.state)]: \(.title)"'
```

## 出力を信用しないケース（MUST）

- **マージ完了**: `gh pr merge` の出力だけでは成功と断定しない。必ず `gh pr view --json state --jq ".state"` で `MERGED` を確認する。
- **CI 状態**: `gh run list` のテキスト出力ではなく、`gh pr view <PR番号> --json statusCheckRollup` または SHA 指定の API で正確な状態を取得する。

## 関連

- `powershell-environment.md`: PowerShell 環境全般の落とし穴（UTF-8 表示、`/dev/null` 代替 等）
- `pr-approval.md`: PR 承認権限ルール（責任者のみ）
