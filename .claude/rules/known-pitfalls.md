---
version: "2.2.1"
has_placeholders: true
description: "既知の落とし穴（Claude Code 汎用 + PJ固有追記用）"
---

# 既知の落とし穴

開発中に発見された、繰り返しハマりやすい問題のまとめ。

## エージェント調査結果の裏取り

リソースの有無・ファイルの存在といったエージェント調査結果は、重要な判断の前に grep 等で自分でも確認する。エージェントの「存在しない」報告を鵜呑みにしない。

## Monitor ツールのコマンドは bash で実行される

Monitor ツールの `command` は `/usr/bin/bash` で実行される。PowerShell 構文をそのまま書くと `syntax error` で exit 2 になる。

### スクリプトファイル化（必須）

Claude Code の worktree isolation check（Command shape チェック）は、`while` + `case` + ネストした `$(...)` 等の複雑な bash コマンドを「検証不能」として拒否する（無効化不可）。ワークツリー外のセッションでも同一手順で統一するため、**Monitor スクリプトは常にファイルに書き出して `bash <path>` で起動する**。

手順:

1. Write ツールで `$env:TEMP` 配下にスクリプトファイル（`.sh`）を作成する
   - **改行は LF**（CRLF で書き出すと Git Bash が `$'\r'` エラーで失敗する）
   - スクリプト内容は bash 構文で記述する
2. Monitor ツールの `command` に `bash "<フォワードスラッシュ絶対パス>"` を指定する
   - パスのバックスラッシュは bash に解釈されるため、**必ずフォワードスラッシュ**に変換する
   - 例: `bash "C:/Users/username/AppData/Local/Temp/ci-monitor-pr-123.sh"`

スクリプト内容は bash で記述すること（PowerShell ではない）。監視ロジック（ステータス取得・判定・exit）を省略しないこと。
実装例は `bs-cc-plugins` プラグインの `_shared/ci-monitor.md` テンプレートを使用すること（定義済み）。

<!-- {{PROJECT_PITFALLS}} — PJ固有の落とし穴があれば以下に追記 -->
⚠️ sync-rules placeholder — PJ固有の落とし穴があれば追記すること。マーカー行は削除禁止（sync-rules 更新保護に必要）。
<!-- END {{PROJECT_PITFALLS}} -->
