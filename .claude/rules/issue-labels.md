# イシューラベル付与ルール

イシュー作成時（手動・CLI 問わず）および既存イシューへの着手時に適用するラベリング規約。

## ラベル体系

| 軸 | ラベル | 付与タイミング |
|---|---|---|
| 種別 | `type: bug` / `type: task` / `type: enhancement` / `type: feature` | **必須・1 つ**。作成時に付与 |
| コンポーネント | `comp: *`（下表参照） | **必須・1 つ**。作成時に付与 |
| 優先度 | `priority: blocker` / `priority: critical` / `priority: major` / `priority: minor` / `priority: trivial` | **任意**。明確な場合のみ |
| 解決区分 | `wontfix` / `duplicate` / `worksforme` / `invalid` | クローズ時のみ |

### 種別（type）の判定基準

| type ラベル | 用途 | ブランチプレフィックス対応 |
|---|---|---|
| `type: bug` | バグ・不具合の修正 | `bugfix/` `hotfix/` |
| `type: task` | 設定変更・リファクタリング・ドキュメント・テスト・雑務 | `chore/` `docs/` `refactor/` `test/` |
| `type: enhancement` | **既存**機能の改善・拡張 | `feature/`（既存改善の場合） |
| `type: feature` | **新規**機能の追加 | `feature/`（新規の場合） |

`feature/` プレフィックスでは新規追加か既存改善かを判断して `type: feature` または `type: enhancement` を選択する。迷う場合は `type: feature` をデフォルトとする。

### コンポーネント（comp）一覧

| ラベル | 対象 |
|---|---|
| `comp: server-java` | bizprint-server-java（Java 版 spp 生成ライブラリ） |
| `comp: server-csharp` | bizprint-server-csharp（C# 版 spp 生成ライブラリ） |
| `comp: client` | bizprint-client（Windows 印刷クライアント） |
| `comp: ci` | GitHub Actions・CI・リリースワークフロー |
| `comp: docs` | ドキュメント（docs/ 配下） |
| `comp: claude` | Claude Code スキル・ルール・プラグイン運用 |
| `comp: sample` | サンプルコード（sample/ 配下） |
| `comp: general` | 横断・その他 |

横断的なタスク（リポジトリ設定・依存更新等）でコンポーネントが特定できない場合は `comp: general` を使用する。

### 優先度（priority）の運用方針

- **不明な場合は付けない**。確信がある場合のみ付与する
- 判断の目安: `blocker`（リリースブロッカー）、`critical`（クラッシュ・データ破損）、`major`（主要機能の不具合）、`minor`（軽微な問題）、`trivial`（表示上の些細な問題）

## 既存イシューのラベル補完

start-task で既存イシューに着手する際、`type: *` または `comp: *` が欠けている場合はイシューの内容から判定して補完する。

- `gh issue view <番号> --json labels,title,body` で現在のラベルを確認する
- `type: *` ラベルが 1 つもなければ、内容から判定して `gh issue edit <番号> --add-label "type: <種別>"` で追加する
- `comp: *` ラベルが 1 つもなければ、内容から判定して `gh issue edit <番号> --add-label "comp: <コンポーネント>"` で追加する
- **既に付いているラベルは変更・削除しない**（`--add-label` のみ使用）
- 判定結果はユーザーに提示してから適用する

想定ケース: QA チームが別リポジトリの CLI からイシューを起票した場合など、issue form を通らずラベルなしで作成されるイシュー。

## gh コマンド例

```powershell
# ラベル付きでイシュー作成
gh issue create --title "<タイトル>" --body $body --assignee "@me" --label "type: task" --label "comp: general"

# 既存イシューにラベルを追加
gh issue edit <番号> --add-label "type: bug" --add-label "comp: server-java"
```
