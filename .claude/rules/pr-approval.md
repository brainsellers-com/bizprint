---
version: "1.2.0"
has_placeholders: false
description: "PR 承認・マージ運用ルール（責任者のみ）"
---

# PR 承認・マージ運用ルール

## 承認権限

PR の承認・マージは **責任者のみ** が行う。変更対象による例外はない。

## Claude Code の振る舞い

### 責任者の判定

- auto memory（`memory/user_role.md` 等）に「責任者」と明記されているユーザーのみを責任者とみなす
- 記録がない・不明な場合は **責任者ではないとみなす**（安全側にフォールバック）

### 振る舞い

- pr-reviewer で LGTM が出ても、責任者以外のユーザー（および責任者かどうか不明なユーザー）には「承認＆マージしますか？」と聞かない
- 責任者以外のユーザーには「pr-reviewer の結果は LGTM です。責任者の承認をお待ちください。」と案内する
- 責任者から明示的に承認・マージの指示があった場合、または責任者から委任された場合のみ実行する

## 承認・マージ手順（`/approve-pr` スキル）

責任者が実行する手順。PR 作成者が責任者自身かどうかで分岐する。

> **実行上の注意**: 以下の手順は PowerShell を前提とする。`$prSource` 等の PowerShell 変数はコマンドブロック（ツール呼び出し）をまたいで保持されないため、`$prSource` を取得する手順から `git push origin --delete $prSource` までは**同一の PowerShell コマンドブロック内**で実行すること。

### 共通の前提条件

1. pr-reviewer のレビュー結果が LGTM であることを確認
2. CI が pass であることを確認

### PR 作成者 ≠ 責任者の場合（通常フロー）

1. `gh pr review <PR番号> --approve` で承認
2. ソースブランチ名を**マージ前に**取得しておく（`$prSource = gh pr view <PR番号> --json headRefName --jq ".headRefName"`。マージ後にブランチ削除済みだと取得できなくなるリスクを回避）
3. `gh pr merge <PR番号> --merge` でマージ
4. `gh pr view <PR番号> --json state --jq ".state"` で `MERGED` を確認する。**`MERGED` を確認できた場合のみ**手順5に進む。確認できなければ中止してユーザーに報告する（ブランチ削除はしない）
5. `$prSource` が保護ブランチまたは既定ブランチでない場合のみリモートブランチを削除する（詳細は下記「保護ブランチ・既定ブランチの削除防止」参照）
6. ローカルのメインブランチに戻って最新を取り込む（`git switch <メインブランチ>` → `git pull`。`<メインブランチ>` は `main` / `develop` / `v5.2_BS` 等、PJ のメインブランチに読み替える）

### PR 作成者 ＝ 責任者の場合（バイパスマージ）

GitHub では自分が作成した PR を自分で承認できないため、`--admin` でブランチ保護ルールをバイパスして直接マージする。

1. ソースブランチ名を**マージ前に**取得しておく（`$prSource = gh pr view <PR番号> --json headRefName --jq ".headRefName"`。マージ後にブランチ削除済みだと取得できなくなるリスクを回避）
2. `gh pr merge <PR番号> --merge --admin` でバイパスマージ
3. `gh pr view <PR番号> --json state --jq ".state"` で `MERGED` を確認する。**`MERGED` を確認できた場合のみ**手順4に進む。確認できなければ中止してユーザーに報告する（ブランチ削除はしない）
4. `$prSource` が保護ブランチまたは既定ブランチでない場合のみリモートブランチを削除する（詳細は下記「保護ブランチ・既定ブランチの削除防止」参照）
5. ローカルのメインブランチに戻って最新を取り込む（`git switch <メインブランチ>` → `git pull`。`<メインブランチ>` は `main` / `develop` / `v5.2_BS` 等、PJ のメインブランチに読み替える）

### 保護ブランチ・既定ブランチの削除防止

マージ先が既定ブランチ以外（例: `develop`）の PR では、head が保護ブランチや既定ブランチであることがある。誤削除を防ぐため、削除前に必ず判定する（後述の逆マージ PR の head は本判定を行わず常に削除しない）。

```powershell
$defaultBranch = gh repo view --json defaultBranchRef --jq ".defaultBranchRef.name"
$protected = gh api "repos/{owner}/{repo}/branches/$prSource" --jq ".protected" 2>$null
if ($LASTEXITCODE -ne 0 -or [string]::IsNullOrWhiteSpace($protected)) {
    if ($protected -match '"status":\s*"404"') {
        Write-Output "INFO: '$prSource' は既に削除済みのため削除不要です。"
    } else {
        Write-Output "WARN: '$prSource' の保護ブランチ判定に失敗したため、安全側に倒して削除をスキップします。"
    }
} elseif ($protected -eq "true" -or $prSource -eq $defaultBranch) {
    Write-Output "INFO: '$prSource' は保護ブランチまたは既定ブランチのため削除をスキップします。"
} else {
    git push origin --delete $prSource
}
```

`delete_branch_on_merge` が有効なリポジトリではマージ直後に GitHub 側で作業ブランチが既に削除されており、404（レスポンスに `"status":"404"` を含む。実機確認済み）が返ることがある。この場合は「既に削除済みのため削除不要」として INFO で扱う。それ以外の理由で判定に失敗した場合（API エラー・空文字）は安全側に倒して削除しない（WARN）。

### 既定ブランチ以外へのマージ後の逆マージ

マージ先（`baseRefName`）がリポジトリの既定ブランチでない場合、その変更を既定ブランチにも反映するため、マージ先 → 既定ブランチの逆マージを提案する（AskUserQuestion で確認）。

承認されたら、マージ先ブランチをそのまま head にして既定ブランチ向け PR を作成する（作業ブランチは作らない）。

```powershell
gh pr create --base <既定ブランチ> --head <マージ先ブランチ> --title "..." --body "..."
```

- 逆マージ PR も上記の承認・マージ手順でマージする。**マージ方式は必ず `--merge`**（squash / rebase は履歴が揃わず逆マージの意味がなくなるため禁止）。**ただし逆マージ PR は「共通の前提条件」の例外として pr-reviewer の LGTM は不要**とする（既にレビュー済みの差分をそのまま既定ブランチへ反映するだけで新規コード変更を含まないため）。CI の pass は必須。
- コンフリクトは PR の `mergeable` フィールド（`gh pr view <PR番号> --json mergeable --jq ".mergeable"`）が `CONFLICTING` かどうかで検知する。コンフリクト時はユーザーに報告し、手動対応を促す。
- 逆マージ PR 自体（base が既定ブランチ）をマージした後は、その base は既定ブランチ自身なので、再度の逆マージ提案は発生しない。
- **逆マージ PR の head（マージ先ブランチ）は削除しない**（元 PR のマージ先＝長期運用ブランチであり、削除対象ではない）。`delete_branch_on_merge` が有効かつ head が未保護のリポジトリでは GitHub 側でマージ直後に自動削除される可能性があるが、これはスキル側では制御できないため、長期運用ブランチには branch protection を設定しておくこと。

詳細な手順は `/bs-cc-plugins:approve-pr` スキルの SKILL.md（手順 4a・4b）を参照する。
