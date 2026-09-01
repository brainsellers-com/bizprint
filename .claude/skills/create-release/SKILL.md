---
name: create-release
description: GitHub リリースを作成する（タグを作成して push し、リリースワークフローを起動）。
model: sonnet
effort: low
shell: powershell
---

# create-release スキル

## Purpose

タグを作成・push して GitHub Actions のリリースワークフローを起動する。

## Usage

```
/create-release
```

## 前提条件

- main ブランチ上で実行すること（リリース対象のコミットが main にマージ済みであること）。
- 未コミットの変更がないこと。

## 手順

### 1. 事前チェック

以下をすべて確認する。1つでも失敗したら中断してユーザーに報告する。

```powershell
# main ブランチにいるか
git branch --show-current  # => main

# ワーキングツリーがクリーンか
git status --porcelain  # => 空であること

# リモートと同期済みか
git fetch origin main
git diff HEAD origin/main --quiet  # => 差分なし
```

### 2. バージョン番号の確認

ユーザーにバージョン番号を確認する。以下のルールを検証する:

- `v` プレフィックス付きのセマンティックバージョニング形式であること
  - 正規表現: `^v[0-9]+\.[0-9]+\.[0-9]+(-[a-zA-Z0-9.]+)?$`
  - 正しい例: `v1.0.0`, `v1.0.0-rc1`, `v2.1.0-beta.1`
  - 不正な例: `1.0.0`（v なし）, `v1.0`（パッチなし）
- 同名のタグが既に存在しないこと

```powershell
# 既存タグとの重複チェック
git tag -l "<バージョン>"  # => 空であること
```

- `-alpha`, `-beta`, `-rc` を含む場合はプレリリースになることをユーザーに伝える。

### 3. 既存リリースの表示

直近のリリースを表示して、バージョン番号が適切か確認する参考情報とする。

```powershell
gh release list --limit 5
```

### 4. タグ作成と push

ユーザーの最終確認を得てから実行する。

```powershell
git tag <バージョン>
git push origin <バージョン>
```

### 5. リリースワークフローの確認

タグ push 後、リリースワークフローが起動したことを確認する。

```powershell
gh run list --workflow=release.yml --limit 1
```

ユーザーに以下を案内する:

> タグ `<バージョン>` を push しました。リリースワークフローが実行中です。
> 完了すると GitHub Releases にリリースが作成され、成果物がアップロードされます。
> 進捗は `gh run watch` または GitHub の Actions タブで確認できます。

## 完了条件

- タグが push され、リリースワークフローが起動していること。
- ワークフローの成否確認はユーザーに委ねる。

## MUST

- タグ push 前にユーザーの最終確認を得ること。
- main ブランチ上で実行すること。

## MUST NOT

- ユーザーの確認なしにタグを push しないこと。
