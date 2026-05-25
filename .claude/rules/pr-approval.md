---
version: "1.0.0"
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

### 共通の前提条件

1. pr-reviewer のレビュー結果が LGTM であることを確認
2. CI が pass であることを確認

### PR 作成者 ≠ 責任者の場合（通常フロー）

1. `gh pr review <PR番号> --approve` で承認
2. `gh pr merge <PR番号> --merge --delete-branch` でマージ
3. `gh pr view <PR番号> --json state --jq ".state"` で `MERGED` を確認

### PR 作成者 ＝ 責任者の場合（バイパスマージ）

GitHub では自分が作成した PR を自分で承認できないため、`--admin` でブランチ保護ルールをバイパスして直接マージする。

1. `gh pr merge <PR番号> --merge --delete-branch --admin` でバイパスマージ
2. `gh pr view <PR番号> --json state --jq ".state"` で `MERGED` を確認
