# 製品概要
bizprint は [biz-Stream](https://www.tecnos.co.jp/bs/biz-stream/) 製品のダイレクト印刷・バッチ印刷機能を
オープンソース化したプロジェクトである（Apache License v2.0）。

サーバーサイドで印刷データ（spp ファイル）を生成し、クライアント（Windows サービス）が受信して印刷を実行する。

## biz-Stream との関係

bizprint は biz-Stream の印刷系モジュールを独立させたもの。

- **印刷系機能を改修する際は biz-Stream プロジェクトの対応モジュールも同時に改修すること。**

## モジュール構成
| モジュール | 言語 | 責務 |
|---|---|---|
| `bizprint-server-java` | Java | spp ファイル生成ライブラリ（サーバー側） |
| `bizprint-server-csharp` | C# | spp ファイル生成ライブラリ（サーバー側） |
| `bizprint-client` | C# | ダイレクト印刷・バッチ印刷 Windows クライアント |
| `pom.xml`（ルート） | Maven | 一括ビルド用親プロジェクト |
| `sample/` | — | サンプルコード・動作確認用資材 |

## ビルド

### 必要環境
- Windows 10 / Windows Server 2019 以上（.NET Framework 3.5 有効化）
- JDK 8 以上
- Maven 3.0.3 以上
- Visual Studio 2019 以上
- Inno Setup 6 以上
- Adobe Acrobat Reader（クライアント実行時に必要。ActiveX コントロールを利用）

### Claude Code 利用時の追加要件
- PowerShell 7+（pwsh）
- GitHub CLI (`gh`)

### 注意
- 暗号化キーはビルドのたびにランダム生成される。サーバー側とクライアント側で同じキーが必要なため、**一部モジュールのみのビルドは不可。必ず一括ビルドすること。**

### よく使うコマンド

```powershell
# 一括ビルド（JAVA_HOME に JDK 8 を設定済みであること）
mvn clean install

# テスト実行
mvn test -pl bizprint-server-java

# CheckStyle チェック
mvn checkstyle:check -pl bizprint-server-java -Dcheckstyle.consoleOutput=true

# EditorConfig チェック
mvn editorconfig:check -pl bizprint-server-java
```

# 基本方針
- 既存の慣習（ライブラリ、フレームワーク、コーディングスタイル、命名規則）を尊重する。
- 指示が曖昧な場合は必ず質問して確認する。
- 機能追加やバグ修正の際はユニットテストを作成・更新する。省略する場合はユーザーに確認する。
- **AI 駆動開発前提**: ビルド・テスト・実行は `mvn` 等のコマンド一発で完結する設計にする。手動セットアップを前提とした設計は避ける。
- **シェル**: コマンド実行には PowerShell ツールを使用する。Bash ツールは使わない。
- **エージェント起動**: Agent ツールでエージェントを起動する際は `run_in_background` を使用しない（フォアグラウンド起動必須）。バックグラウンドではユーザー承認ダイアログを表示できず PowerShell 等のツールが拒否される。

# コーディング規約

- **Java**: JDK 8 互換。CheckStyle（`.claude/rules/checkstyle.md`）と EditorConfig（`.editorconfig`）に準拠。
- **C#**: .NET Framework 2.0 以上互換（動作確認は 4.6）。既存コードのスタイルに従う。

# 出力
- 丸数字（(1)(2)(3)等）は一切使用禁止。番号付きリスト（1. 2. 3.）を使うこと。これはソースコード、ドキュメント、Claude Code の回答すべてに適用する。

# ワークフロー（必須）

- **リポジトリへのあらゆる変更（コード・ドキュメント・設定ファイル）は、必ず `/bs-cc-plugins:start-task` でイシュー作成・ブランチ作成してから行う。**
- ユーザーが「着手して」「やって」等と指示した場合でも、イシューとブランチが未作成なら先に作成する。
- `main` での直接作業は禁止。
- ブランチ名: `<種別>/<イシュー番号>`（例: `feature/123`, `bugfix/456`, `chore/789`）
- コミット・プッシュ前に必ず `/bs-cc-plugins:maven-build` でローカルビルドを確認する（ドキュメントのみの変更を除く）。
- PR 作成は `/bs-cc-plugins:create-pr` スキルに従う。
- すべての変更は PR 経由で行う。
- PR 本文に `Closes #<イシュー番号>` を記述してイシューと連携する。
- CI が通っていない PR はマージしない。
- **PR 承認は責任者のみ**。詳細: `.claude/rules/pr-approval.md`

## イシューラベル付与ルール

詳細は `.claude/rules/issue-labels.md` を参照。

### 新規イシュー作成時

| ブランチプレフィックス | type ラベル |
|---|---|
| `feature/` | `type: feature`（新規）または `type: enhancement`（既存改善） |
| `bugfix/` `hotfix/` | `type: bug` |
| `chore/` `docs/` `refactor/` `test/` | `type: task` |

`comp: *` はイシュー内容から判定する。横断タスクは `comp: general`。

## コミットメッセージ規約
- フォーマット: `<プレフィックス> <変更内容の要約（日本語）> (#<イシュー番号>)`
- プレフィックス: `feat:` / `fix:` / `chore:` / `docs:` / `refactor:` / `test:`
- 例: `fix: PDF出力時のフォント埋め込みエラーを修正する (#45)`
- Claude Code がコミットする場合は末尾に `Co-Authored-By: <モデル名> <noreply@anthropic.com>` を付与する。

## CI
- GitHub Actions（self-hosted Windows ランナー）で `mvn clean install` を実行。
- PR 作成時と main への push 時に自動実行。手動実行（`workflow_dispatch`）も可能。
- `release.yml`: タグ push 時にリリースワークフローを実行。
