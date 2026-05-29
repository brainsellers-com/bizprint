---
version: "1.3.1"
has_placeholders: true
description: "Claude Code スキル設定ルール（ディレクトリ構成・フロントマター・登録一覧・プラグイン優先順位）"
---

# Claude Code スキル設定ルール

このプロジェクトで Claude Code のカスタムスキルを追加・変更する際のルール。

## ディレクトリ構成（必須）

```
.claude/skills/<skill-name>/SKILL.md
```

- **ディレクトリ形式のみ有効**。フラット形式（`.claude/skills/<skill-name>.md`）では Claude Code に認識されない。
- `SKILL.md` はファイル名を大文字にすること。
- `SKILL.md` の先頭に YAML フロントマター（`name`, `description`）が必要。

```yaml
---
name: skill-name
description: スキルの説明
effort: medium
shell: powershell
---
```

- `effort` と `shell` は必須。`effort` は `low` / `medium` / `high` から選択、`shell` は `powershell` を指定する。
- `model` は **`opus` のみ指定可**。`sonnet` は指定禁止（親セッションが Opus 1M コンテキストで動作している場合、Sonnet 160k のコンテキスト制限でエラーになる）。Sonnet で動かしたい場合は `model` フィールド自体を省略する（省略時はデフォルトモデルが使われる）。※本制約はスキルのみ対象。エージェント（`.claude/agents/`）は独立コンテキストで動作するため `model` を自由に指定できる。

### スキルの必須セクション

すべてのスキルに以下のセクションを含めること:

- **Purpose** — スコープと使用タイミング
- **Usage** — 呼び出し方法
- **手順**（番号付き） — 段階的な実行手順
- **完了条件** — 何を満たせば完了か
- **MUST** / **MUST NOT** — 制約事項

## .gitignore との衝突に注意

`.gitignore` に登録されているパターンがスキルディレクトリ名にマッチする場合、
Git 管理から除外されてしまう。

**既知の例**: `build/` が `.gitignore` にマッチするため、`build` という名前のスキルは
リポジトリに追加できない。この場合は `maven-build` のように別名にすること。

スキルを追加したら `git status` で意図通り追跡されているか確認すること。

## 共通パーツ（.claude/skills/_shared/）

複数スキルで共通する手順を `_shared/` に切り出し、各スキルから `@_shared/<ファイル名>` で参照する。

<!-- {{SHARED_PARTS_TABLE}} — PJ固有の _shared ファイル一覧をテーブルで記載 -->
（なし — プラグイン移管により全ファイル削除済み）
<!-- END {{SHARED_PARTS_TABLE}} -->

## カスタムエージェント（.claude/agents/）

```
.claude/agents/<agent-name>/AGENT.md
```

- スキルと同様にディレクトリ形式で配置する。
- `AGENT.md` の先頭に YAML フロントマター（`name`, `description`, `tools`, `model` 等）が必要。
- エージェントは親セッションとは独立したコンテキストで動作する。

## プラグイン（settings.json の enabledPlugins）

プラグインのスキル・エージェントは `/<プラグイン名>:<スキル名>` で呼び出す。

| プラグイン名 | 用途 |
|---|---|
| `bs-cc-plugins` | BrainSellers 共通ワークフロー |
| `claude-md-management` | CLAUDE.md の監査・改善 |
| `skill-creator` | スキルのレビュー・評価 |

### bs-cc-plugins が提供するスキル・エージェント

| 種別 | 名前 | 用途 |
|---|---|---|
| スキル | `start-task` | イシュー・ブランチ作成（開発着手前に必須） |
| スキル | `create-pr` | GitHub PR 作成 |
| スキル | `approve-pr` | PR 承認・マージ・完了確認・ブランチ戻し |
| スキル | `check-review` | PR レビューコメント取得・指摘対応支援 |
| スキル | `pr-review` | PR レビュー実行（pr-reviewer エージェント起動） |
| スキル | `self-audit` | Claude Code 運用の自己改善監査 |
| スキル | `design-prep` | 設計前提メモ作成（要件分析） |
| スキル | `design-doc` | 設計書セット生成 |
| スキル | `maven-build` | Maven ビルド実行 |
| スキル | `sync-rules` | ルールテンプレート同期 |
| スキル | `init-project` | プロジェクト初期セットアップ |
| スキル | `tf-coverage-check` | Terraform カバレッジ確認 |
| スキル | `design-reviewer` | 設計書レビュー起動（design-reviewer エージェント呼び出し） |
| エージェント | `pr-reviewer` | PR コードレビュー（独立コンテキスト） |
| エージェント | `design-reviewer` | 設計書レビュー（独立コンテキスト） |
| エージェント | `code-reviewer` | コードレビュー（汎用） |

### プラグインとの同名スキルの優先順位

ローカルスキルとプラグインスキルに同名がある場合、**必ずローカル版を使用する**。ローカル版は PJ 固有の要件（JDK バージョン、出力構成、セルフチェック項目等）を反映しているため、プラグイン版（汎用版）では不足する。

<!-- {{OVERRIDE_SKILLS_TABLE}} — ローカル版でオーバーライドするスキルをテーブルで記載 -->
（なし）
<!-- END {{OVERRIDE_SKILLS_TABLE}} -->

## PJ 固有のスキル・エージェント一覧

<!-- {{PROJECT_SKILLS_INVENTORY}} — PJ固有のスキル・エージェント一覧をテーブル形式で記載 -->
### PJ 固有スキル

| スキル名 | 用途 |
|---|---|
| `create-release` | GitHub リリース作成（タグ push → リリースワークフロー起動） |

### PJ 固有エージェント

（なし）
<!-- END {{PROJECT_SKILLS_INVENTORY}} -->
