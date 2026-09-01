---
version: "1.8.0"
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
model: sonnet
effort: medium
shell: powershell
---
```

- `model`, `effort`, `shell` は必須。
- `model` は `sonnet` / `opus` / `fable`、または `claude-opus-4-6[1m]` のような具体的なモデルID文字列を指定する（**必須**）。`haiku` は禁止（200k コンテキストのため、1M の親セッションでコンテキストオーバーの懸念がある）。モデル選択基準は下記「モデル選択基準」を参照。スキル・エージェント共通のルール。
- `effort` は `low` / `medium` / `high` から選択。`shell` は `powershell` を指定する。

### モデル選択基準

| 用途 | model |
|---|---|
| 定型・オーケストレーション系のスキル | `sonnet` |
| 複雑な設計・実装（変更ファイル10未満かつ変更行数500行未満の目安） | `sonnet` |
| 複雑な設計・実装（据え置き対象） | `opus` 系（個別に `claude-opus-4-6[1m]` 等を指定） |
| 基本的なレビュー・複雑な設計判断 | `claude-opus-4-6[1m]` をデフォルトとする |

### override 運用（fable への切り替え）

上記のモデル選択基準は基本方針であり、以下のいずれかに該当する場合のみ、Agent 呼び出し時に `model: "fable"` を指定して override する（コストが Opus の2倍のため用途を限定）:

- 差分規模が大きい（目安: 変更ファイル10以上 または 変更行数500行以上）
- アーキテクチャ変更・複数モジュール横断の設計判断を伴う
- セキュリティ / データ整合性等クリティカルな変更
- 直前の Opus レビューの指摘の質に疑問がある
- 責任者から明示指示があった

該当しない通常のレビュー・実装ではデフォルト（`claude-opus-4-6[1m]` 等、フロントマターで定義されたモデル）をそのまま使用し、`fable` への override は行わない。

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

## Plan モードの品質チェック

plan モードで ExitPlanMode を呼ぶ**前に**、plan-reviewer エージェントに plan のレビューを依頼すること。

### 手順

1. plan ファイルへの書き込みが完了したら、Agent ツールで plan-reviewer を起動する

   ```
   Agent ツール起動:
     subagent_type: "bs-cc-plugins:plan-reviewer:plan-reviewer"
     prompt: |
       Plan をレビューしてください。
       プロジェクトルート: <プロジェクトルートの絶対パス>
       Plan ファイル: <plan ファイルの絶対パス>
       イシュー: <対象イシューのタイトルと説明（分かる場合）>
   ```

   `subagent_type` は必ず `"bs-cc-plugins:plan-reviewer:plan-reviewer"` を使用する。無修飾の `"plan-reviewer"` や2セグメントの `"bs-cc-plugins:plan-reviewer"` ではプラグインエージェントが起動されないため使わないこと。

   `model` は通常指定しない（デフォルトの `claude-opus-4-6[1m]` を使用）。差分規模が大きい・アーキテクチャ変更を伴う・クリティカルな変更等、難しいと判断した場合のみ Agent 呼び出しで `model: "fable"` を指定する（判断基準は上記「override 運用」を参照）。

2. plan-reviewer が **REVISE** を返した場合:
   - 「修正必須事項」に基づいて plan ファイルを修正する
   - 修正後、再レビューは行わず ExitPlanMode に進む（デフォルトが Opus 系となり運用コストが下がったため再レビューは省略とする。難しい plan で override により fable を使った場合も同様に再レビューは省略する）
   - ユーザーへの plan 提示時に、修正した指摘事項と修正しなかった指摘事項（理由付き）を一覧で示す

3. plan-reviewer が **APPROVED** を返した場合:
   - ExitPlanMode に進む
   - ユーザーへの plan 提示時に「plan-reviewer によるレビュー済み」と一言添える

### 適用外

以下の場合は plan-reviewer を省略してよい:

- 単純なタイプミス修正・1行変更など、レビューのコストに見合わない軽微な変更
- ユーザーが明示的にレビュー省略を指示した場合

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
| スキル | `dependabot-triage` | Dependabot PR トリアージ・統合 PR 作成・元 PR クローズ |
| スキル | `sync-rules` | ルールテンプレート同期 |
| スキル | `init-project` | プロジェクト初期セットアップ |
| スキル | `tf-coverage-check` | Terraform カバレッジ確認 |
| スキル | `design-reviewer` | 設計書レビュー起動（design-reviewer エージェント呼び出し） |
| エージェント | `pr-reviewer` | PR コードレビュー（独立コンテキスト） |
| エージェント | `design-reviewer` | 設計書レビュー（独立コンテキスト） |
| エージェント | `code-reviewer` | コードレビュー（汎用） |
| エージェント | `plan-reviewer` | Plan 品質レビュー（独立コンテキスト） |

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
